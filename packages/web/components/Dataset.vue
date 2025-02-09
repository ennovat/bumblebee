<template>
	<div class="table-container">
    <div v-if="(!(currentDataset && currentDataset.summary) && !loadPreviewActive) || $store.state.kernel=='loading'" class="no-data">
      <div v-if="appError" class="title grey--text text-center text-with-icons">
        There's a problem <br/><br/>
        <v-btn color="primary" depressed @click="reloadInit">Reload</v-btn>
      </div>

      <div
        v-else-if="(commandsDisabled || $store.state.kernel=='loading')"
        class="progress-middle title grey--text text-center"
        :class="{'has-text': $store.state.kernel=='loading'}"
      >
        <v-progress-circular
          indeterminate
          color="#888"
          size="64"
        />
        <template v-if="$store.state.kernel=='loading'">
          Initializing
        </template>
      </div>
      <div v-else class="title grey--text text-center text-with-icons">
        <v-progress-circular
          v-if="loadingDf"
          indeterminate
          color="#888"
          class="mb-10"
          size="64"
        />
        <template v-else>
          <div class="available-dfs mb-4" v-if="availableDatasets && availableDatasets.length">
            Load from existing data sources:
            <template v-for="(dfName, index) in availableDatasets">
              <span :key="'av'+dfName">
                <template v-if="index>0">, </template>
                <span class="primary--text hoverable" @click="openDf(dfName)">{{dfName}}</span>
              </span>
            </template>
          </div>
          <v-btn
            @click="commandHandle({command: 'loadFile'})"
            color="primary"
            class="mr-3"
            depressed
          >Load from file</v-btn>
          <span> or </span>
          <v-btn
            @click="commandHandle({command: 'loadDatabaseTable'})"
            color="primary"
            class="ml-3"
            depressed
          >Load from database</v-btn>
        </template>
      </div>
    </div>
		<div v-else-if="currentListView && !loadPreviewActive" class="table-view-container">
			<div class="table-controls d-flex">
				<v-btn
          color="#888" text icon small @click="toggleColumnsSelection">
					<v-icon>
						<template v-if="selectionStatus==-1">indeterminate_check_box</template>
						<template v-else-if="selectionStatus==1">check_box</template>
						<template v-else>check_box_outline_blank</template>
					</v-icon>
				</v-btn>
				<v-tooltip transition="tooltip-fade-transition" bottom>
					<template v-slot:activator="{ on }">
						<v-btn
              color="#888"
							:class="{'active': selectionStatus!=0}"
							text
							icon
							small
							class="hideable tr-o"
							v-on="on"
							@click="toggleColumnsVisibility(!visibilityStatus)">
							<v-icon class="small-icon">
								<template v-if="!visibilityStatus">visibility</template>
								<template v-else>visibility_off</template>
							</v-icon>
						</v-btn>
					</template>
					<span v-if="!visibilityStatus">Show selected columns on table</span>
					<span v-else>Hide selected columns on table</span>
				</v-tooltip>
				<v-tooltip transition="tooltip-fade-transition" bottom>
					<template v-slot:activator="{ on }">
						<v-btn
              color="#888"
							:class="{'active': currentListView && thereAreHidden}"
							text
							icon
							small
							class="hideable tr-o"
							v-on="on"
							@click="restoreVisibility">
							<v-icon style="position: absolute; left: 3px;"
                class="smaller-icon">
								visibility
							</v-icon>
							<v-icon>
								mdi-restore
							</v-icon>
						</v-btn>
					</template>
					<span>
            Restore visibility
          </span>
				</v-tooltip>
				<v-tooltip transition="tooltip-fade-transition" bottom>
					<template v-slot:activator="{ on }">
						<v-btn
              color="#888"
							:class="{'active': currentListView && selectionStatus==-1}"
							text
							icon
							small
							class="hideable tr-o"
							v-on="on"
							@click="toggleUnselectedColumnsVisibility">
							<v-icon style="position: absolute"
                class="smaller-icon">
								visibility
							</v-icon>
							<v-icon>
								mdi-select
							</v-icon>
						</v-btn>
					</template>
					<span>
            <template v-if="everyUnselectedIsHidden">
              Show unselected columns
            </template>
            <template v-else>
              Show only selected columns
            </template>
          </span>
				</v-tooltip>
        <v-tooltip transition="tooltip-fade-transition" bottom>
					<template v-slot:activator="{ on }">
						<v-btn
              color="#888"
							class="hideable tr-o pl-1"
							:class="{'active': currentListView && selectionStatus==-1}"
							text
							icon
							small
							v-on="on"
							@click="invertSelection">
							<v-icon>
								$vuetify.icons.check-box-invert
							</v-icon>
						</v-btn>
					</template>
					<span>Invert selection</span>
				</v-tooltip>
			</div>
			<v-data-table
				v-show="currentListView"
				:headers="columnsTableHeaders"
				:items="filteredColumns"
				:sort-by.sync="_sortBy"
				:sort-desc.sync="_sortDesc"
				:mobile-breakpoint="0"
				:height="600"
				class="columns-table"
				disable-pagination
				hide-default-footer
				fixed-header
				@click:row="rowClicked"
			>
				<template v-slot:[`item.controls`]="{ item }">
					<div class="cell-controls">
						<v-icon class="control-button control-check" @click.stop="toggleColumnSelection(item.name)">
							<template v-if="selectedColumns[item.name]">check_box</template>
							<template v-else>check_box_outline_blank</template>
						</v-icon>
						<v-tooltip transition="tooltip-fade-transition" bottom>
							<template v-slot:activator="{ on }">
								<v-icon
									:class="{'control-hide-hidden': hiddenColumns[item.name]}"
									class="control-button control-hide"
									@click.stop="toggleColumnVisibility(item.name)"
									v-on="on"
								>
									<template v-if="!!hiddenColumns[item.name]">
										visibility_off
									</template>
									<template v-else>
										visibility
									</template>
								</v-icon>
							</template>
							<span v-if="!hiddenColumns[item.name]">Hide column on table</span>
							<span v-else>Show column on table</span>
						</v-tooltip>
					</div>
				</template>
				<template v-slot:[`item.profilerDtype`]="{ item }">
					<v-tooltip transition="tooltip-fade-transition" bottom>
						<template v-slot:activator="{ on }">
							<div class="data-type pr-4" v-on="on">
								{{ dataTypeHint(item.profilerDtype) }}
							</div>
						</template>
						<span :key="'column-type-hint'" class="capitalize column-type">
							{{ item.profilerDtype }}
							<template v-if="item.profilerDtype==='string*'">
								<br>
								<span v-for="(subtype) in getSubTypes(item)" :key="subtype" class="subtype">
									{{ subtype }}
								</span>
							</template>
						</span>
					</v-tooltip>
				</template>
				<template
					v-slot:[`item.type`]="{item}"
				>
					<div class="data-type-name">
						{{ item.type }}
					</div>
				</template>
				<template v-slot:[`item.name`]="{ item }">
					<div class="pr-4">
						{{ item.name }}
					</div>
				</template>
				<template v-slot:[`item.missing`]="{ item }">
					<div class="pr-4">
						{{ item.missing | formatNumberInt }}
					</div>
				</template>
				<template v-slot:[`item.null`]="{ item }">
					<div class="pr-4" v-if="item.null!==undefined">
						{{ item.null | formatNumberInt }}
					</div>
				</template>
        <template v-slot:[`item.zeros`]="{ item }">
					<div class="pr-4" v-if="item.zeros!==undefined">
						{{ item.zeros | formatNumberInt }}
					</div>
				</template>
			</v-data-table>
		</div>
		<client-only>
			<div
				v-show="!currentListView && (currentDataset && currentDataset.summary || loadPreviewActive) && $store.state.kernel!='loading'"
				class="bumblebee-table-container"
			>
        <div v-if="noMatch" class="no-data">
          <div class="title grey--text text-center text-with-icons">
            No match
          </div>
        </div>
        <BumblebeeTable
					v-if="$store.state.optimusPromise && $store.state.optimusPromise.fulfilled && !currentListView && (currentDataset && currentDataset.summary || loadPreviewActive)"
          v-show="!noMatch"
          :bbColumns="bbColumns"
          @sort="updateSortedColumns"
          @updatedSelection="selectionEvent"
          ref="bumblebeeTable"
        />
			</div>
		</client-only>
	</div>
</template>

<script>

import { printError, parseResponse, debounce, throttle, getPropertyAsync, deepCopy } from 'bumblebee-utils'

import { mapGetters } from 'vuex'
import dataTypesMixin from '@/plugins/mixins/data-types'
import clientMixin from '@/plugins/mixins/client'
import BumblebeeTable from '@/components/BumblebeeTable'

export default {

  mixins: [
    dataTypesMixin,
    clientMixin
  ],

  components: {
    BumblebeeTable
  },

  props: {
    operationsActive: {
      type: Boolean,
      default: false
    },
    sortBy: {
      type: Array,
      default: ()=>{return []}
    },
    sortDesc: {
      type: Array,
      default: ()=>{return [false]}
    },
    columnsTableHeaders: {
      type: Array,
      default: ()=>{return []}
    },
		typesSelected: {
			default: () => ([]),
			type: Array
    },
    searchText: {
      default: '',
      type: String
    }
  },

  data () {
    return {

      loadingDf: false,

      resultsColumnsData: false, // search
      selectedColumns: {},

      sortedColumns: [], // table manual sorting

      isMounted: false,

      loadedPreviewCode: ''
    }
  },

  computed: {

    ...mapGetters([
      'currentSelection',
      'currentDataset',
      'currentListView',
      'secondaryDatasets',
      'currentHiddenColumns',
      'previewCode',
      'loadPreview',
      'appError'
    ]),

    hiddenColumns: {
      get () {
        return this.currentHiddenColumns || {};
      },
      set (value) {
        this.$store.commit('setHiddenColumns', value);
      }
    },

    noMatch () {
      return !this.loadPreviewActive && this.customSortedColumns && this.customSortedColumns.length && ( (this.resultsColumnsData && !this.resultsColumnsData.length) || !this.bbColumns.length ) ;
    },

    showingColumnsLength () {
      return this.bbColumns.length;
    },

    commandsDisabled: {
      get () {
        return this.$store.state.commandsDisabled
      },
      set (value) {
        this.$store.commit('mutation', {mutate: 'commandsDisabled', payload: value})
      }
    },

    everyUnselectedIsHidden () {
      for (let i = 0; i < this.filteredColumns.length; i++) {
				const column = this.filteredColumns[i]
				if (!this.selectedColumns[column.name] && !this.hiddenColumns[column.name]) {
          return false;
        }
      }
      return true;
    },

    thereAreHidden () {
      return Object.values(this.hiddenColumns).some(e=>e);
    },

    availableDatasets () {
      var sds = Array.from(this.secondaryDatasets)
        .filter(e=>e.startsWith('df'))

      this.$store.state.datasets.forEach(dataset => {
        if (!dataset.dfName) {
          return
        }
        var foundIndex = sds.findIndex(sd=>sd===dataset.dfName)
        if (foundIndex>=0) {
          sds.splice(foundIndex,1)
        }
      })

      return sds
    },

    loadPreviewActive () {
      try {
        return (this.previewCode.loadPreview && this.loadPreview)
      } catch (error) {
        return false
      }
    },

    customSortedColumns () {
      if (this.sortedColumns.length && this.currentDataset && this.columns && this.columns.length) {
        return this.sortedColumns.map(i=>this.columns[i])
      }
      else if (this.currentDataset && this.columns) {
        return this.columns
      }
      else {
        return []
      }
    },

    selectionStatus () {

      let status

      if (this.filteredColumns && this.filteredColumns.length) {
        for (let i = 0; i < this.filteredColumns.length; i++) {
          const column = this.filteredColumns[i]
          if (status !== undefined && status === +!this.selectedColumns[column.name]) { // different from previous value
            return -1 // indeterminate
          }
          status = +!!this.selectedColumns[column.name] // all or none selected
        }
      }
			return +!!status
    },

    visibilityStatus () {

      if (this.filteredColumns && this.filteredColumns.length) {
        for (let i = 0; i < this.filteredColumns.length; i++) {
          const column = this.filteredColumns[i]
          if (this.selectedColumns[column.name] && this.hiddenColumns[column.name]) {
            return 0 // at least one is hidden
          }
        }
      }
			return 1
    },

    columns () {
      let columns = [];
      if (this.currentDataset) {
        if (this.currentDataset.columns) {
          columns = this.currentDataset.columns;
        }
        if (this.$store.state.columns[this.currentDataset.dfName] && columns.length < this.$store.state.columns[this.currentDataset.dfName].length) {
          columns = this.$store.state.columns[this.currentDataset.dfName].map(column=>({ name: column }));
        }
      }
      return columns;
    },

    // affects table view only
    bbColumns () {
      var bbColumns = this.filteredColumns ? [...this.filteredColumns.filter((column) => {
				return !this.hiddenColumns[column.name]
      })] : []

      let selectColumns = Array.from(this.customSortedColumns)// [...(this.customSortedColumns || [])]

			if (this.sortBy[0] && bbColumns.length) {
				if (typeof bbColumns[0][this.sortBy[0]] === 'string') {
					bbColumns.sort((a, b) => {
						let _a = a[this.sortBy[0]].toLowerCase()
						let _b = b[this.sortBy[0]].toLowerCase()
						return (_a < _b) ? -1 : (_a > _b) ? 1 : 0
          })
          selectColumns.sort((a, b) => {
						let _a = a[this.sortBy[0]].toLowerCase()
						let _b = b[this.sortBy[0]].toLowerCase()
						return (_a < _b) ? -1 : (_a > _b) ? 1 : 0
          })
				} else {
					bbColumns.sort((a, b) => {
						let _a = a[this.sortBy[0]]
						let _b = b[this.sortBy[0]]
						return (_a < _b) ? -1 : (_a > _b) ? 1 : 0
          })
          selectColumns.sort((a, b) => {
						let _a = a[this.sortBy[0]]
						let _b = b[this.sortBy[0]]
						return (_a < _b) ? -1 : (_a > _b) ? 1 : 0
          })
				}
				if (this.sortDesc[0]) {
          bbColumns.reverse()
          selectColumns.reverse()
        }
      }

      this.$emit('sort',selectColumns.map(e=>e.name))

      if (this.columns && this.columns.length) {
        return bbColumns.map(e=>this.columns.findIndex(de => de.name === e.name))
      } else {
        return bbColumns.map((e,i)=>i)
      }


    },

    resultsColumns () {
      if (!this.resultsColumnsData || !this.resultsColumnsData.length)
        return this.customSortedColumns
      else
        return this.resultsColumnsData
    },

    filteredColumns () {

      try {
        var filteredColumns

        if (this.typesSelected.length > 0) {
          filteredColumns = this.resultsColumns.filter((column) => {
            return this.typesSelected.includes(column.stats.inferred_type.data_type) || !column.stats
          })
        } else {
          filteredColumns = this.resultsColumns
        }

        if (filteredColumns) {

          this.$nextTick(()=>{
            let _selected = []

            if (this.columns) {
              for (let i = 0; i < this.columns.length; i++) {
              const column = this.columns[i];
              if (this.selectedColumns[column.name]) {
                  _selected.push(i)
                }
              }
            }
            this.handleSelection( _selected, true )
          })
          return filteredColumns.map(col => {
            return { ...col, profilerDtype: col.stats ? col.stats.inferred_type.data_type : null }
          });
        }
      } catch (err) {
        console.error(err)
      }
      return []
    },

    _sortBy: {
      get () {
        return this.sortBy
      },
      set (v) {
        this.$emit('update:sortBy',v)
      }
    },
    _sortDesc: {
      get () {
        return this.sortDesc
      },
      set (v) {
        this.$emit('update:sortDesc',v)
      }
    },

    columnsHeader () {
			return this.columns.map((e) => {
				return e.name
			})
    },
  },

  beforeCreate () {
    this.isMounted = false
  },

  mounted () {

    try {
      this.getSelectionFromStore()
    } catch (error) {}

    this.$nextTick(()=>{
      this.isMounted = true
    })

  },

  methods: {

    async openDf (dfName) {
      try {
        this.loadingDf = true;
        this.$store.commit('setDfToTab', { dfName, go: true });
        await this.$store.dispatch('getProfiling', { payload: { dfName, socketPost: this.socketPost, partial: true, methods: this.commandMethods } });
        this.$store.commit('setDfToTab', { dfName, go: true });
      } catch (err) {
        console.error('Error opening dataset', err);
        this.$store.commit('unsetDf', { dfName });
      }
      this.loadingDf = false;
    },

    commandHandle (event) {
      this.$store.commit('commandHandle', event);
    },

    reloadInit () {
      this.$store.commit('setAppStatus', { status: 'workspace' });
    },

    async updateResults() {

      if (this.searchText) {
        this.resultsColumnsData = await this.$search(this.searchText, this.customSortedColumns, {
          shouldSort: true,
          threshold: 0.1,
          keys: ['name']
        })
      } else {
        this.resultsColumnsData = false
      }
    },

    updateSortedColumns(event) {
      this.$emit('sortDrag')
      this.sortedColumns = []
      this.sortedColumns = event
      this.updateResults()
    },

    getSelectionFromStore () {
      var selectedColumns = {}
      var storeSelectedColumns = this.currentSelection.columns

      if (storeSelectedColumns) {
        for (let index = 0; index < storeSelectedColumns.length; index++) {
          selectedColumns[storeSelectedColumns[index].name] = true
        }
      }

      if (JSON.stringify(selectedColumns) !== JSON.stringify(this.selectedColumns)) {
        this.selectedColumns = selectedColumns
      }

    },


    displaySelection: throttle( async function (item) {
			if (item) {
				this.scatterPlotDisplay = item.values
      }
    },100),

    watchSearchText: debounce( async function() {
      try {
        this.updateResults()
      } catch (err) {
        // console.error(err)
      }
    }, 800),

		toggleColumnsSelection () {
			let select = this.selectionStatus !== 1

			if (!select) {
				this.selectedColumns = {}
			} else {
				for (let i = 0; i < this.filteredColumns.length; i++) {
					const column = this.filteredColumns[i]
					this.$set(this.selectedColumns, column.name, select)
				}
			}

		},

		getSubTypes (item) {
			if (item.stats) {
				return Object.keys(item.stats)
					.filter((k) => {
						return (!!item.stats[k] && k !== 'string' && k !== 'missing' && k !== 'null')
					})
					.map((k) => {
						return item.stats[k] + ' ' + k
					})
			}
		},

		rowClicked (e) {
      this.toggleColumnSelection(e.name)
    },

    selectionEvent (selection) {
      this.handleSelection(selection,true)
    },

    handleSelection (selected, indices = false) {
      if (this.isMounted) {
        this.$emit('selection',{selected, indices})
      }
    },

		toggleColumnSelection (value) {
			if (this.selectedColumns[value]) {
				this.$set(this.selectedColumns, value, false)
				delete this.selectedColumns[value]
      }
      else {
        this.$set(this.selectedColumns, value, true)
      }
		},

		invertSelection () {
			for (let i = 0; i < this.filteredColumns.length; i++) {
				const column = this.filteredColumns[i]
				if (this.selectedColumns[column.name]) {
          // this.$delete(this.cSelectedColumn, column.name)
          this.$set(this.selectedColumns, column.name, false)
        }
        else {
          this.$set(this.selectedColumns, column.name, true)
        }
			}
		},

		toggleColumnsVisibility (value) {
      var hiddenColumns = {...this.hiddenColumns};
			for (let i = 0; i < this.filteredColumns.length; i++) {
				const column = this.filteredColumns[i]
				if (this.selectedColumns[column.name]) {
          hiddenColumns[column.name] = !value
        }
      }
      this.hiddenColumns = hiddenColumns;
    },

		restoreVisibility (value) {
      this.hiddenColumns = {};
		},

		toggleColumnVisibility (colName) {
      var hiddenColumns = {...this.hiddenColumns};
			hiddenColumns[colName] = !hiddenColumns[colName];
      this.hiddenColumns = hiddenColumns;
    },

    toggleUnselectedColumnsVisibility () {
      var hiddenColumns = {};
      var hidden = !this.everyUnselectedIsHidden;
			for (let i = 0; i < this.filteredColumns.length; i++) {
				const column = this.filteredColumns[i]
				if (!this.selectedColumns[column.name]) {
          hiddenColumns[column.name] = hidden
        }
      }
      this.hiddenColumns = hiddenColumns;
    }
  },

  watch: {

    currentDataset: {
      deep: true,
      handler (value) {
        this.sortedColumns = []
      },
    },

    noMatch (value) {
      this.$store.commit('mutation', { mutate: 'noMatch', payload: value });
    },

    showingColumnsLength: {
      immediate: true,
      handler (value) {
        this.$store.commit('mutation', { mutate: 'showingColumnsLength', payload: value });
      },
    },

    operationsActive () {
      this.$refs.bumblebeeTable && this.$refs.bumblebeeTable.checkVisibleColumns()
    },

    previewCode: {
      deep: true,
      async handler () {
        try {
          var currentCode = await getPropertyAsync(this.previewCode.code)
          if (this.loadedPreviewCode!==currentCode) {
            this.loadedPreviewCode = currentCode
            if (this.previewCode.load) {
              this.$store.commit('mutation', {mutate: 'loadingStatus', payload: 'Updating Preview' })

              var dfName = 'preview_df'
              var codePayload = deepCopy(this.previewCode.codePayload);

              codePayload.request = {
                ...this.previewCode.codePayload.request,
                type: 'preview',
                saveTo: dfName,
                isLoad: true,
                createsNew: true,
                sample: true,
                meta: true,
              }

              console.debug('[PREVIEW] Loading', codePayload)

              var response = await this.evalCode(codePayload);

              if (response.data.result && response.data.result.sample) {
                this.$store.commit('setLoadPreview', { sample: response.data.result.sample } )
              } else {
                throw response
              }

              if (response.data.result.meta) {
                this.$store.commit('setLoadPreview', { meta: response.data.result.meta } )
              }

              var pCodePayload = {
                command: 'profile_async',
                dfName,
                request: {
                  priority: -10,
                  isAsync: true,
                }
              }

              var pResponse = await this.evalCode(pCodePayload)

              if (!pResponse || !pResponse.data) {
                console.error(pResponse);
                throw new Error('Unknown error');
              }

              if (pResponse.data.status === 'error') {
                throw pResponse.data.error || new Error('Unknown error');
              }

              var profile = parseResponse(pResponse.data.result)

              this.$store.commit('setLoadPreview', { profile } )

              this.$store.commit('mutation', {mutate: 'loadingStatus', payload: false })

            }
          }
        } catch (err) {
          let _error = printError(err);
          this.$store.commit('setPreviewInfo', { error: _error });
          this.$store.commit('mutation', {mutate: 'loadingStatus', payload: false })
        }
      },
    },

    selectedColumns: {
      deep: true,
      handler (v) {
        this.handleSelection(Object.entries(v).filter((e)=>e[1]).map(e=>e[0]) || [], false)
      }
    },

    currentSelection() {
      this.getSelectionFromStore()
    },

    searchText: {
			immediate: true,
			handler: 'watchSearchText'
		}
	},


}
</script>
