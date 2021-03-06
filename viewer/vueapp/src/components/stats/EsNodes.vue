<template>

  <div class="container-fluid mt-2">

    <moloch-loading v-if="initialLoading && !error">
    </moloch-loading>

    <moloch-error v-if="error"
      :message="error">
    </moloch-error>

    <div v-show="!error">

      <moloch-paging v-if="stats"
        class="mt-1 ml-2"
        :info-only="true"
        :records-total="recordsTotal"
        :records-filtered="recordsFiltered">
      </moloch-paging>

      <moloch-table
        id="esNodesTable"
        :data="stats"
        :loadData="loadData"
        :columns="columns"
        :no-results="true"
        :show-avg-tot="true"
        :action-column="true"
        :desc="query.desc"
        :sortField="query.sortField"
        table-animation="list"
        table-classes="table-sm text-right small mt-2"
        table-state-name="esNodesCols"
        table-widths-state-name="esNodesColWidths">
        <template slot="actions"
          slot-scope="{ item }">
          <b-dropdown size="sm"
            class="row-actions-btn"
            v-has-permission="'createEnabled'">
            <b-dropdown-item v-if="!item.nodeExcluded"
              @click="exclude('name', item)">
              Exclude node {{ item.name }}
            </b-dropdown-item>
            <b-dropdown-item v-else
              @click="include('name', item)">
              Include node {{ item.name }}
            </b-dropdown-item>
            <b-dropdown-item v-if="!item.ipExcluded"
              @click="exclude('ip', item)">
              Exclude IP {{ item.ip }}
            </b-dropdown-item>
            <b-dropdown-item v-else
              @click="include('ip', item)">
              Include IP {{ item.ip }}
            </b-dropdown-item>
          </b-dropdown>
        </template>
      </moloch-table>

    </div>

  </div>

</template>

<script>
import MolochTable from '../utils/Table';
import MolochError from '../utils/Error';
import MolochLoading from '../utils/Loading';
import MolochPaging from '../utils/Pagination';

let reqPromise; // promise returned from setInterval for recurring requests
let respondedAt; // the time that the last data load succesfully responded

export default {
  name: 'EsStats',
  props: [ 'dataInterval', 'refreshData', 'searchTerm' ],
  components: {
    MolochTable,
    MolochError,
    MolochPaging,
    MolochLoading
  },
  data: function () {
    return {
      error: '',
      initialLoading: true,
      stats: null,
      recordsTotal: null,
      recordsFiltered: null,
      query: {
        filter: this.searchTerm || undefined,
        sortField: 'nodeName',
        desc: false
      },
      columns: [ // es stats table columns
        // default columns
        { id: 'name', name: 'Name', sort: 'nodeName', doStats: false, default: true, width: 120 },
        { id: 'docs', name: 'Documents', sort: 'docs', doStats: true, default: true, width: 120, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.docs); } },
        { id: 'storeSize', name: 'Disk Used', sort: 'storeSize', doStats: true, default: true, width: 100, dataFunction: (item) => { return this.$options.filters.humanReadableBytes(item.storeSize); } },
        { id: 'freeSize', name: 'Disk Free', sort: 'freeSize', doStats: true, default: true, width: 100, dataFunction: (item) => { return this.$options.filters.humanReadableBytes(item.freeSize); } },
        { id: 'heapSize', name: 'Heap Size', sort: 'heapSize', doStats: true, default: true, width: 100, dataFunction: (item) => { return this.$options.filters.humanReadableBytes(item.heapSize); } },
        { id: 'load', name: 'OS Load', sort: 'load', doStats: true, default: true, width: 100, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.load, 2); } },
        { id: 'cpu', name: 'CPU', sort: 'cpu', doStats: true, default: true, width: 80, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.cpu, 1) + '%'; } },
        { id: 'read', name: 'Read/s', sort: 'read', doStats: true, default: true, width: 90, dataFunction: (item) => { return this.$options.filters.humanReadableBytes(item.read); } },
        { id: 'write', name: 'Write/s', sort: 'write', doStats: true, default: true, width: 90, dataFunction: (item) => { return this.$options.filters.humanReadableBytes(item.write); } },
        { id: 'searches', name: 'Searches/s', sort: 'searches', doStats: true, width: 100, default: true, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.searches); } },
        // all the rest of the available stats
        { id: 'ip', name: 'IP', sort: 'ip', doStats: false, width: 100 },
        { id: 'ipExcluded', name: 'IP Excluded', sort: 'ipExcluded', doStats: false, width: 100 },
        { id: 'nodeExcluded', name: 'Node Excluded', sort: 'nodeExcluded', doStats: false, width: 125 },
        { id: 'nonHeapSize', name: 'Non Heap Size', sort: 'nonHeapSize', doStats: false, width: 100, dataFunction: (item) => { return this.$options.filters.humanReadableBytes(item.nonHeapSize); } },
        { id: 'searchesTime', name: 'Search Time', sort: 'searchesTime', doStats: true, width: 100, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.searchesTime); } },
        { id: 'writesRejected', name: 'Write Tasks Rejected', sort: 'writesRejected', doStats: true, width: 100, canClear: true, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.writesRejected); } },
        { id: 'writesRejectedDelta', name: 'Write Tasks Rejected/s', sort: 'writesRejectedDelta', doStats: true, width: 100, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.writesRejectedDelta); } },
        { id: 'writesCompleted', name: 'Write Tasks Completed', sort: 'writesCompleted', doStats: true, width: 100, canClear: true, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.writesCompleted); } },
        { id: 'writesCompletedDelta', name: 'Write Tasks Completed/s', sort: 'writesCompletedDelta', doStats: true, width: 100, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.writesCompletedDelta); } },
        { id: 'writesQueueSize', name: 'Write Tasks Q Limit', sort: 'writesQueueSize', doStats: true, width: 100, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.writesQueueSize); } }
      ]
    };
  },
  computed: {
    loading: {
      get: function () {
        return this.$store.state.loadingData;
      },
      set: function (newValue) {
        this.$store.commit('setLoadingData', newValue);
      }
    }
  },
  watch: {
    dataInterval: function () {
      if (reqPromise) { // cancel the interval and reset it if necessary
        clearInterval(reqPromise);
        reqPromise = null;

        if (this.dataInterval === '0') { return; }

        this.setRequestInterval();
      } else if (this.dataInterval !== '0') {
        this.loadData();
        this.setRequestInterval();
      }
    },
    refreshData: function () {
      if (this.refreshData) {
        this.loadData();
      }
    }
  },
  created: function () {
    // set a recurring server req if necessary
    if (this.dataInterval !== '0') {
      this.setRequestInterval();
    }
  },
  methods: {
    /* exposed page functions ------------------------------------ */
    exclude: function (type, column) {
      this.$http.post(`esshard/exclude/${type}/${column[type]}`)
        .then((response) => {
          if (type === 'name') {
            column.nodeExcluded = true;
          } else {
            column.ipExcluded = true;
          }
        }, (error) => {
          this.error = error.text || error;
        });
    },
    include: function (type, column) {
      this.$http.post(`esshard/include/${type}/${column[type]}`)
        .then((response) => {
          if (type === 'name') {
            column.nodeExcluded = false;
          } else {
            column.ipExcluded = false;
          }
        }, (error) => {
          this.error = error.text || error;
        });
    },
    /* helper functions ------------------------------------------ */
    setRequestInterval: function () {
      reqPromise = setInterval(() => {
        if (respondedAt && Date.now() - respondedAt >= parseInt(this.dataInterval)) {
          this.loadData();
        }
      }, 500);
    },
    loadData: function (sortField, desc) {
      this.loading = true;
      respondedAt = undefined;

      this.query.filter = this.searchTerm;

      if (desc !== undefined) { this.query.desc = desc; }
      if (sortField) { this.query.sortField = sortField; }

      this.$http.get('esstats.json', { params: this.query })
        .then((response) => {
          respondedAt = Date.now();
          this.error = '';
          this.loading = false;
          this.initialLoading = false;
          this.stats = response.data.data;
          this.recordsTotal = response.data.recordsTotal;
          this.recordsFiltered = response.data.recordsFiltered;
        }, (error) => {
          respondedAt = undefined;
          this.loading = false;
          this.initialLoading = false;
          this.error = error.text || error;
        });
    }
  },
  beforeDestroy: function () {
    if (reqPromise) {
      clearInterval(reqPromise);
      reqPromise = null;
    }
  }
};
</script>

<style scoped>
td {
  white-space: nowrap;
}
tr.bold {
  font-weight: bold;
}
table.table tr.border-bottom-bold > td {
  border-bottom: 2px solid #dee2e6;
}
table.table tr.border-top-bold > td {
  border-top: 2px solid #dee2e6;
}

.table .hover-menu:hover .btn-group {
  visibility: visible;
}
.table .hover-menu .btn-group {
  visibility: hidden;
}
</style>
