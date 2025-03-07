<template>
  <div class="dashboard">
    <div>
      <b-row style="margin:auto">
        <b-col cols="auto" class="p-1">
          <b-pagination
          v-model="currentPage"
          :total-rows="num_of_rows"
          aria-controls="dashboard-table"
          align="right"
          :per-page="rowsPerPage"
          ></b-pagination>
        </b-col>
        <b-col cols="auto" class="p-1" style="margin-left:15pt">
          Rows per page <b-form-select v-model="rowsPerPage" :options="pageLengthOptions" style="width:50pt" align="right"></b-form-select>
        </b-col>
      </b-row>
    </div>

    <b-table
      id="dashboard-table"
      striped
      hover
      bordered
      class="tbl_dashboard"
      :fields="fields"
      :items="filtered"
      :sort-by.sync="sortBy"
      :sort-desc.sync="sortDesc"
      :per-page="rowsPerPage"
      :current-page="currentPage"
    >
      <template slot="top-row" slot-scope="{ fields }">
        <td v-for="field in fields" :key="field.key">
          <input v-if="field.filter" v-model="filters[field.key]" placeholder="filter">
        </td>
      </template>
      <template #thead-top>
        <b-tr>
          <b-th colspan="1"><span class="sr-only">ID</span></b-th>
          <b-th variant="light" colspan="4">Metadata</b-th>
          <b-th variant="info" colspan="6">Builds</b-th>
          <b-th variant="primary" colspan="6">Workers</b-th>
        </b-tr>
      </template>
      <template #cell()="data">
        <b-icon v-if="data.field.type === 'pass_fail'" v-on:click="show_html(data.value.msg)" :icon="`${data.value.icon}`" scale="2" :variant="`${data.value.variant}`" style="cursor:pointer"></b-icon>
        <a v-else-if="data.field.type === 'link'" :href="`${data.field.link_prefix}${data.value}`" target="_blank">{{ data.value }}</a>
        <div v-else> {{ data.value }} </div>
      </template>

    </b-table>
  </div>
</template>

<script>
import DashboardDataService from "../services/DashboardDataService";
import { onValue } from "firebase/database";

export default {
  name: "dashboard",
  data() {
    return {
      db_data: {},
      sortBy: 'm_time',
      sortDesc: true,
      items: [],
      filters: {
        branch: '',
        actor: '',
      },
      fields: [
          {
            key: 'build_id',
            label: 'Build ID',
            type: 'link',
            link_prefix: 'https://github.com/magma/magma/commit/',
            sortable: false,
            formatter: value => {
              return value.substring(0,8)
            }
          },
          {
            key: 'm_time',
            label: 'Time',
            sortable: false,
            formatter: value => {
              var date = new Date(value*1000);
              return date.toLocaleString("en-US", { year: '2-digit', month: 'numeric', day: 'numeric' , hour: 'numeric', minute: 'numeric', hour12: false})
            },
          },
          {
            key: 'm_run_id',
            label: 'Run',
            type: 'link',
            link_prefix: 'https://github.com/magma/magma/actions/runs/',
          },
          {
            key: 'm_branch',
            label: 'Branch',
            filter: true,
          },
          {
            key: 'm_actor',
            label: 'Actor',
            filter: true
          },
          {
            key: 'b_agw',
            label: 'AGW',
            type: 'pass_fail',
            formatter: 'build_formatter',
          },
          {
            key: 'b_arm_agw',
            label: 'ARM AGW',
            type: 'pass_fail',
            formatter: 'build_formatter',
          },
          {
            key: 'b_feg',
            label: 'FEG',
            type: 'pass_fail',
            formatter: 'build_formatter',
          },
          {
            key: 'b_orc8r',
            label: 'ORC8R',
            type: 'pass_fail',
            formatter: 'build_formatter',
          },
          {
            key: 'b_nms',
            label: 'NMS',
            type: 'pass_fail',
            formatter: 'build_formatter',
          },
          {
            key: 'b_cwag',
            label: 'CWAG',
            type: 'pass_fail',
            formatter: 'build_formatter',
          },
          {
            key: 'w_wl5g',
            label: 'WL 5G',
            type: 'pass_fail',
            formatter: 'test_worker_formatter',
          },
          {
            key: 'w_sudo_python',
            label: 'SUDO PYTHON',
            type: 'pass_fail',
            formatter: 'test_worker_formatter',
          },
          {
            key: 'w_lte_integ_containerized',
            label: 'LTE INTEG CONTAINERIZED',
            type: 'pass_fail',
            formatter: 'test_worker_formatter',
          },
          {
            key: 'w_lte_integ_debian',
            label: 'LTE INTEG DEBIAN',
            type: 'pass_fail',
            formatter: 'test_worker_formatter',
          },
          {
            key: 'w_feg_integ',
            label: 'FEG INTEG',
            type: 'pass_fail',
            formatter: 'test_worker_formatter',
          },
          {
            key: 'w_cwf_integ',
            label: 'CWF INTEG',
            type: 'pass_fail',
            formatter: 'test_worker_formatter',
          },
        ],
      currentPage: 1,
      rowsPerPage: 20,
      pageLengthOptions: [
        { value: 20, text: '20' },
        { value: 50, text: '50' },
        { value: 100, text: '100' },
      ]
    };
  },
  computed: {
    num_of_rows() {
      return this.items.length
    },
    filtered () {
      const filtered = this.items.filter(item => {
        return Object.keys(this.filters).every(key =>
            String(item[key]).includes(this.filters[key]))
      })
      return filtered.length > 0 ? filtered : []
    }
  },
  methods: {
    onDataChange(dbObject) {
      this.db_data = dbObject;
      this.items = []
      for (const [key, build] of Object.entries(dbObject.builds)) {
        var branch_path = build.metadata["github:ref"].split('/')
        var branch_name = branch_path[branch_path.length - 1]

        // get reports
        var sudo_python_report = {};
        if (dbObject.workers.sudo_python_tests.reports) {
          sudo_python_report = (key in dbObject.workers.sudo_python_tests.reports) ? dbObject.workers.sudo_python_tests.reports[key] : {"verdict": "_not_present"};
        }
        var wl5g_report = {};
        if (dbObject.workers.wl_lab_5g.reports) {
          wl5g_report = (key in dbObject.workers.wl_lab_5g.reports) ? dbObject.workers.wl_lab_5g.reports[key] : {"verdict": "_not_present"};
        }
        var lte_integ_containerized_report = {};
        if (dbObject.workers.containerized_lte_integ_test.reports) {
          lte_integ_containerized_report = (key in dbObject.workers.containerized_lte_integ_test.reports) ? dbObject.workers.containerized_lte_integ_test.reports[key] : {"verdict": "_not_present"};
        }
        var lte_integ_debian_report = {};
        if (dbObject.workers.debian_lte_integ_test.reports) {
          lte_integ_debian_report = (key in dbObject.workers.debian_lte_integ_test.reports) ? dbObject.workers.debian_lte_integ_test.reports[key] : {"verdict": "_not_present"};
        }
        var feg_integ_report = {};
        if (dbObject.workers.feg_integ_test.reports) {
          feg_integ_report = (key in dbObject.workers.feg_integ_test.reports) ? dbObject.workers.feg_integ_test.reports[key] : {"verdict": "_not_present"};
        }
        var cwf_integ_report = {};
        if (dbObject.workers.cwf_integ_test && dbObject.workers.cwf_integ_test.reports) {
          cwf_integ_report = (key in dbObject.workers.cwf_integ_test.reports) ? dbObject.workers.cwf_integ_test.reports[key] : {"verdict": "_not_present"};
        }

        this.items.push(
          {
            build_id: key,
            m_time: build.metadata.timestamp,
            m_run_id: build.metadata["github:run_id"],
            m_actor: build.metadata["github:actor"],
            m_branch: branch_name,
            b_agw: build.agw,
            b_feg: build.feg,
            b_orc8r: build.orc8r,
            b_nms: build.nms,
            b_cwag: build.cwag,
            w_sudo_python: sudo_python_report,
            w_wl5g: wl5g_report,
            w_lte_integ_containerized: lte_integ_containerized_report,
            w_lte_integ_debian: lte_integ_debian_report,
            w_feg_integ: feg_integ_report,
            w_cwf_integ: cwf_integ_report,
          })
      }
      //console.log(this.items)
    },
    test_worker_formatter(value) {
      let newValue = {
        "icon": "circle",
        "variant": "secondary",
        "msg": value.report
      }
      if (value.verdict == "pass") {
        newValue.icon = "check-circle-fill"
        newValue.variant = "success"
      } else if(value.verdict == "fail") {
        newValue.icon = "exclamation-circle-fill"
        newValue.variant = "danger"
      } else if(value.verdict != "_not_present") {
        newValue.icon = "exclamation-circle-fill"
        newValue.variant = "warning"
      }
      return newValue
    },
    build_formatter(value) {
      var htmlStringify = require('html-stringify');
      var msg = htmlStringify(value);
      let newValue = {
        "icon": "check-circle-fill",
        "variant": "success",
        "msg": msg
      }
      if (value == null) {
        newValue.icon = "circle"
        newValue.variant = "secondary"
      } else if (value.valid != true) {
        newValue.icon = "exclamation-circle-fill"
        newValue.variant = "danger"
      }
      return newValue
    },
    show_html: function (message) {
      // Workaround if window already open
      var wnd = window.open("", "Test Report", "_blank");
      wnd.close()
      wnd = window.open("", "Test Report", "_blank");
      wnd.document.write('<title>Report</title>');
      wnd.document.write(message);
    }
  },
  mounted() {
    let dbRef = DashboardDataService.getAll()
    onValue(dbRef, (snapshot) => {
      this.onDataChange(snapshot.val())
    });
  }
};
</script>

<style>
.dashboard {
  width: 90%;
  margin-top: 20pt;
  margin-left: auto;
  margin-right: auto;
}
</style>
