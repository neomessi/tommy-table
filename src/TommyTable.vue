<template>
    <div>
		<div v-if="$data.actualCands.length">
			<button @click="clearSearch" title="Reset search/sort fields" style="font-size: 16px;">&nbsp;<span v-html="getHtmlEntity('clearSearch')"></span>&nbsp;</button>
			&nbsp;&nbsp;		
			<select v-model="loadNum" title="Batch size">
				<option v-for="(n, i) in $data.loadNums" :value="n" :key="i">{{ n }}</option>
				<option :value="0" key="a">All</option>
			</select>
			<br style="line-height: 28px" />
			<form ref="searchFields">
				<table class="table table-bordered table-condensed table-responsive">
					<thead>
						<tr>
							<th :key=i v-for="(k,i) in Object.keys($data.actualCands[0])" v-if="i > 0" nowrap>
								<a @click.stop="doSort(k);">{{ prettifyHeader(k) }}</a>
								<span v-if="getSortSeqKeyPos(k)>=0">							
									&nbsp;
									<span v-if="getKeySortSeqCycle(k)==0" v-html="getHtmlEntity('sortUp')"></span>
									<span v-if="getKeySortSeqCycle(k)==1"  v-html="getHtmlEntity('sortDown')"></span>							
									<span style="font-size:x-small;" v-if="getNumSortSeqs()>1">{{ getSortSeqKeyPos(k)+1 }}</span>
								</span>						
							</th>
						</tr>
					</thead>
					<tbody>
						<tr>
							<td :key=i v-for="(k,i) in Object.keys($data.actualCands[0])" v-if="i > 0" nowrap>
								
								<!-- text input (default) -->
								<span v-if="!(/bit|choice/i.test($data.useDataTypes[i]))">
									<input title="type to search" @keyup="updateSearchField( i, $event.target.value );">
								</span>

								<!-- bit -->
								<span v-if="/bit/i.test($data.useDataTypes[i])">								
									<label style="font-size: smaller"><input type="radio" :id="`tommyTable_radio_y_${i}`" :name="`tommyTable_radio_${i}`" value="1" @click="updateSearchField( i, $event.target.value );">&nbsp;{{ $data.useBitindicators[0] }}</label>
									<br />
									<label style="font-size: smaller"><input type="radio" :id="`tommyTable_radio_n_${i}`" :name="`tommyTable_radio_${i}`" value="0" @click="updateSearchField( i, $event.target.value );">&nbsp;{{ $data.useBitindicators[1] }}</label>
								</span>

								<!-- choice -->
								<span v-if="/choice/i.test($data.useDataTypes[i])">
									<select @change="updateSearchMethod( i, $props.choices[i].options[$event.target.value].criteria );" :id="`tommyTable_select_${i}`">
										<option v-for="(c, j) in $props.choices[i].options" :value="j" :key="j">{{ c.label }}</option>
									</select>
								</span>

							</td>
						</tr>
						<tr v-for="(m,i) in $data.searchCands" :key="i" v-if="i <= $data.shownNum">
							<td :key=j v-for="(k,j) in Object.keys(m)" v-if="j > 0">
								<span v-if="j === 1">
									<span v-if="parseInt(m[Object.keys($data.actualCands[0])[0]]) > 0">
										<a v-bind:href="`${$props.detaillink}${m[Object.keys($data.actualCands[0])[0]]}`" target="_blank">{{ formatValue(m[k], j) }}</a>
									</span>
									<span v-if="parseInt(m[Object.keys($data.actualCands[0])[0]]) <= 0">
										{{ formatValue(m[k], j) }}
									</span>
								</span>
								<span v-if="j > 1">
									{{ formatValue(m[k], j, $props.choices[j] ? $props.choices[j].dataTypeFormat : null) }}
								</span>
							</td>					
						</tr>
					</tbody>
				</table>
			</form>		
			<div v-if="!$data.searchCands.length && $data.actualCands.length" class="alert alert-warning">No results</div>
			<br />
			<div style="text-align: center;" v-if="shownNum < $data.searchCands.length">
				<a @click.stop="loadMore()" style="font-size: xxx-large; text-decoration: none; cursor: pointer;" title="Load more"><span v-html="getHtmlEntity('loadMore')"></span></a>
			</div>		
		</div>
		<div v-if="!$data.actualCands.length" class="alert alert-danger">No data</div>
    </div>	
</template>

<script>
	export default {
        props: {
        	batchsizes: { default: [25,50,100,200] },
			bitindicators: { default: ['Y','N'] },
			choices: { default: {} },
			debugmode: { default: false },
			defaultsearches: { default: [] },
			defaultsorts: { default: [] },
			detaillink: { default: '/' },
			datatypes: { default: [] },
			tabledatavar: { default: "window.tommyTableData" }
        },

        data: function() {
            return {
				loadNums: this.batchsizes,
				loadNum: this.batchsizes[0],
				shownNum: this.batchsizes[0],
				useDataTypes: this.datatypes,
				useBitindicators: this.bitindicators,
				useDefaultSearches: this.defaultsearches,
				useDefaultSorts: this.defaultsorts,

				searchFields: [],
				searchMethods: [],
				sortSeq: [], // sequence of sort fields.  These will be pushed/popped so can sort by multiple columns
				sortSeqCycle: [], // the current "cycle" corresponding to each sortSeq element above (0=asc, 1=desc, 2=remove).  the cycle is bumped every time field is clicked for sort
				actualCands: [], // the entire supplied JSON data
				searchCands: [], // actualCands are copied over here if any of criteria match (also removed from here if doesn't)

				defaultHtmlEntities: {
					clearSearch: '&#9851', // Universal Recycling Symbol
					loadMore: '&#9660;', // Down-Pointing Triangle
					sortDown: '&darr;', // DOWNWARDS ARROW
					sortUp: '&uarr;', // UPWARDS ARROW
				}
			}
        },

        created: function() {
            this.actualCands = [...(eval(this.tabledatavar) || [])];
			this.searchCands = [...this.actualCands];
			this.useDataTypes = [...this.useDataTypes, ...Array(Object.keys(this.actualCands[0]).length-this.useDataTypes.length).fill('string')];			
        },

		mounted: function() {
			this.initSearchFields();
			this.initSearchMethods();
			this.handleDefaultSearches();
			this.handleDefaultSorts();

			if (this.debugmode) {
				console.log(`** tommyTable intialized with the following: **`);
				console.log(`Total records: ${this.actualCands.length}`);
				console.log(`Records after search(s): ${this.searchCands.length}`);
				console.log(`Showing: ${this.shownNum}`);
				console.log(`Data types: ${this.useDataTypes}`);
				console.log(`Search fields: ${this.searchFields}`);
				console.log(`Search methods: ${this.searchMethods}`);
				console.log(`Sorting on: ${this.sortSeq}`);
				console.log(`Sort directions: ${this.sortSeqCycle.map((e) => e ? 'desc' : 'asc')}`);
			}
		},

		watch: {
			loadNum: {
				handler: function(v) {
					if (v > this.shownNum) {
						this.shownNum = v;
					}
					else if (v === 0) {
						this.shownNum = this.actualCands.length;
					}
				}
			}
		},

        methods: {
			/* sorting functions
			*/
			doSort(key, cycle) {
				const defaultCycle = typeof cycle === 'number' ? cycle : 0; // 0=asc
				var keypos =0;
				if ( typeof(key) != 'undefined' ) {
					keypos = this.sortSeq.indexOf(key);
					if ( keypos == -1 ) {
						this.sortSeqCycle.push(defaultCycle);
						this.sortSeq.push( key );
						keypos = this.sortSeq.length-1;
					}else {
						if ( ++this.sortSeqCycle[keypos] == 2 ) { // 2=remove
							this.sortSeqCycle.splice(keypos, 1);
							this.sortSeq.splice(keypos, 1);
							if ( keypos == this.sortSeq.length ) {
								return; // nothing to sort (removed last sort item)
							}else {
								//key = this.sortSeq[keypos++]; // set key to next sort item (removed in middle)								
							}
						}
					}
				}				

				for ( var k=keypos; k<this.sortSeq.length; k++ ) {
					var key = this.sortSeq[k];
					var arr = this.sortSeq.slice();
					var parentsortkey = ( k>0 ) ? arr[this.sortSeq.indexOf(key)-1] : ""; // for multiple field sorting						
					var useSortSeqCycle = this.sortSeqCycle[this.sortSeq.indexOf(key)];						
					
					if ( arr.length == 1) {
						this.searchCands.sort(function(a, b){
							var doNumericSort = !isNaN(a[key]) && !isNaN(b[key]);
							if ( useSortSeqCycle == 0 ) {
								//0=asc sort									
								if (doNumericSort) {
									return a[key] - b[key]
								}									
								return a[key] == b[key] ? 0 : a[key] < b[key] ? -1 : 1;
							}else {
								//1=desc sort
								if (doNumericSort) {
									return b[key] - a[key]
								}
								return a[key] == b[key] ? 0 : a[key] > b[key] ? -1 : 1;
							}								
						});
					}
					else {
						// break into several arrays, sort and put back together
						var searchcandstmp = [];
						var searchcandstmp2 = [];
						var lastparentval = null;
						for ( var i=0; i<=this.searchCands.length; i++ ) {							
							if ( i==this.searchCands.length || this.searchCands[i][parentsortkey] != lastparentval ) {
								searchcandstmp.sort(function(a, b){									
									if ( useSortSeqCycle == 0 ) { //0=asc sort
										return a[key] == b[key] ? 0 : a[key] < b[key] ? -1 : 1;
									}else { //1=desc sort
										return a[key] == b[key] ? 0 : a[key] > b[key] ? -1 : 1;
									}									
								});								
								searchcandstmp2 = searchcandstmp2.concat( searchcandstmp.slice() );								
								searchcandstmp = [];								
							}
							if ( i<this.searchCands.length ) {
								lastparentval = this.searchCands[i][parentsortkey];
								searchcandstmp.push( this.searchCands[i] );
							}
							
						}												
						this.searchCands = [];
						this.searchCands = searchcandstmp2.slice();						
					}					
				}
			},

			/* searching functions
			*/
			doSearch: function() {
				var arrMatch = [];
				for ( var i=0; i<this.actualCands.length; i++ ) {
					var j=0;
					for ( var key in this.actualCands[i] ) {
						var val=this.searchFields[j];
						var fn = this.searchMethods[j];
						if ( val != "" ) {
							var re = new RegExp(val, 'i');
							if ( !this.actualCands[i][key].toString().match(re) ) {
								break;
							}
						}
						else {
							if (!fn(this.actualCands[i][key].toString())) {
								break;
							}							
						}
						j++;
					}
					if ( j == Object.keys(this.actualCands[i]).length ) {
						arrMatch.push(this.actualCands[i]);
					}
				}
				this.searchCands = [];
				for ( var i=0; i<arrMatch.length; i++ ) {
					this.searchCands.push(arrMatch[i]);
				}
				this.doSort();
			},				
			updateSearchField: function(i, val) {
				this.searchFields[i]=val;
				this.doSearch();
			},
			updateSearchMethod: function(i, fn) {				
				this.searchMethods[i]=fn;
				this.doSearch();
			},
			clearSearch: function() {
				this.searchCands = this.actualCands.slice();
				this.initSearchMethods();
				this.resetSearchFields();
				this.sortSeq=[];
				this.sortSeqCycle=[];					
			},				
			resetSearchFields: function() {				
				this.initSearchFields();				
				this.$refs.searchFields.reset();
			},
			initSearchFields: function() {
				this.searchFields = Array(Object.keys(this.actualCands[0]).length).fill('');
			},
			initSearchMethods: function() {
				this.searchMethods = Array(Object.keys(this.actualCands[0]).length).fill(() => true);
			},
			handleDefaultSearches: function() {
				const _vue = this;
				this.useDefaultSearches.forEach(function(e, i) {
					if (/object/i.test(typeof e)) {
						_vue.updateSearchMethod(i, _vue.choices[i].options[e.choiceCriteraIndex] && _vue.choices[i].options[e.choiceCriteraIndex].criteria || function() { return true });

						// update DOM
						document.querySelector(`#tommyTable_select_${i}`).selectedIndex = e.choiceCriteraIndex;
					}
					else {
						_vue.updateSearchField(i, e.toString());

						// update DOM
						if (/bit/i.test(_vue.useDataTypes[i])) {
							document.querySelector(`#tommyTable_radio_y_${i}`).checked = !!e;
							document.querySelector(`#tommyTable_radio_n_${i}`).checked = !!!e;
						}						
					}					
				});
				this.doSearch();
			},
			handleDefaultSorts: function() {
				if (!this.useDefaultSorts.length) return;

				const _vue = this;
				this.useDefaultSorts.forEach(function(e, i) {					
					if (e && /^(asc|desc)$/i.test(e)) {
						_vue.doSort(Object.keys(_vue.actualCands[0])[i], /asc/i.test(e) ? 0 : 1);
					}
				});
			},

			/* getters
			*/
			getSortSeqKeyPos: function(key) {
				return this.sortSeq.indexOf(key);
			},
			getKeySortSeqCycle: function(key) {
				return this.sortSeqCycle[this.sortSeq.indexOf(key)];
			},
			getNumSortSeqs: function() {
				return this.sortSeq.length;
			},
			getHtmlEntity: function(k) {
				const h = this.defaultHtmlEntities[k];
				if (h) {
					return h;
				}
				throw (`HTML enitity '${k}' not found`);
			},

			/* misc
			*/
			loadMore: function() {
				this.shownNum += this.loadNum;
			},

			/* display functions
			*/
			formatValue: function(s, i, override) {				
				switch(override || this.useDataTypes[i]) {
					case 'money':
						return new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(s);

					case 'bit':
						return s ? this.useBitindicators[0] : this.useBitindicators[1];

					default: // 'string'/undefined (empty)
						return s;
				}					
			},
			prettifyHeader: (s) => s.replace(/_/g, ' ').replace(/\b([a-z]{1})/g, (s, m) => m.toUpperCase()),
		},
    }
</script>

<style scoped>
    tbody tr:nth-child(odd) {
        background-color: #FFF;
    }
    tbody tr:nth-child(even) {
        background-color: #f0f0f0;
    }
</style>