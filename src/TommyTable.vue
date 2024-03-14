<template>
    <div>
		<div v-if="actualCands.length">
			<button @click="clearSearch" title="Reset search/sort fields" style="font-size: 16px;">&nbsp;<span v-html="getHtmlEntity('clearSearch')"></span>&nbsp;</button>
			&nbsp;&nbsp;		
			<select v-model="loadNum" title="Batch size">
				<option v-for="(n, i) in loadNums" :value="n" :key="i">{{ n }}</option>
				<option :value="0" key="a">All</option>
			</select>
			<br style="line-height: 28px" />
			<form ref="searchFieldsRef">
				<table class="table table-bordered table-condensed table-responsive">
					<thead>
						<tr>
                            <template v-for="(k,i) in Object.keys(actualCands[0])">
                                <th :key=i v-if="i > 0" nowrap>
                                    <a @click.stop="doSort(k);">{{ prettifyHeader(k) }}</a>
                                    <span v-if="getSortSeqKeyPos(k)>=0">							
                                        &nbsp;
                                        <span v-if="getKeySortSeqCycle(k)==0" v-html="getHtmlEntity('sortUp')"></span>
                                        <span v-if="getKeySortSeqCycle(k)==1"  v-html="getHtmlEntity('sortDown')"></span>							
                                        <span style="font-size:x-small;" v-if="getNumSortSeqs()>1">{{ getSortSeqKeyPos(k)+1 }}</span>
                                    </span>						
                                </th>
                            </template>
						</tr>
					</thead>
					<tbody>
						<tr>
                            <template v-for="(k,i) in Object.keys(actualCands[0])">
							<td :key=i v-if="i > 0" nowrap>
                                <!-- text input (default) -->
								<span v-if="!(/bit|choice/i.test(useDataTypes[i]))">
									<input title="type to search" @keyup="updateSearchField( i, $event.target.value );">
								</span>

								<!-- bit -->
								<span v-if="/bit/i.test(useDataTypes[i])">
									<label style="font-size: smaller"><input type="radio" :id="`tommyTable_radio_y_${i}`" :name="`tommyTable_radio_${i}`" value="1" @click="updateSearchField( i, $event.target.value );">&nbsp;{{ bitindicators[0] }}</label>
									<br />
									<label style="font-size: smaller"><input type="radio" :id="`tommyTable_radio_n_${i}`" :name="`tommyTable_radio_${i}`" value="0" @click="updateSearchField( i, $event.target.value );">&nbsp;{{ bitindicators[1] }}</label>
								</span>

								<!-- choice -->
								<span v-if="/choice/i.test(useDataTypes[i])">
									<select @change="updateSearchMethod( i, choices[i].options[$event.target.value].criteria );" :id="`tommyTable_select_${i}`">
										<option v-for="(c, j) in choices[i].options" :value="j" :key="j">{{ c.label }}</option>
									</select>
								</span>
							</td>
                            </template>
						</tr>
                        <template v-for="(m,i) in searchCandsRef">
						<tr :key="i" v-if="i < shownNum">
                            <template v-for="(k,j) in Object.keys(m)">
							<td :key=j  v-if="j > 0">
								<span v-if="j === 1">
									<span v-if="parseInt(m[Object.keys(actualCands[0])[0]]) > 0">
										<a v-bind:href="`${detaillink}${m[Object.keys(actualCands[0])[0]]}`" target="_blank">{{ formatValue(m[k], j) }}</a>
									</span>
									<span v-if="parseInt(m[Object.keys(actualCands[0])[0]]) <= 0">
										{{ formatValue(m[k], j) }}
									</span>
								</span>
								<span v-if="j > 1">
									{{ formatValue(m[k], j, choices[j] ? choices[j].dataTypeFormat : null) }}
								</span>
							</td>	
                            </template>
						</tr>
                        </template>
					</tbody>
				</table>
			</form>		
			<div v-if="!searchCandsRef.length && actualCands.length" class="alert alert-warning">No results</div>
			<br />
			<div style="text-align: center;" v-if="shownNum < searchCandsRef.length">
				<a @click.stop="loadMore()" style="font-size: xxx-large; text-decoration: none; cursor: pointer;" title="Load more"><span v-html="getHtmlEntity('loadMore')"></span></a>
			</div>		
		</div>
		<div v-if="!actualCands.length" class="alert alert-danger">No data</div>
    </div>	
</template>

<script setup>
    import { onMounted, ref, watch } from "vue";

	    const {
                batchsizes,
                bitindicators: useBitindicators,
                choices,
                debugmode,
                defaultsearches: useDefaultSearches,
                defaultsorts: useDefaultSorts,
                detaillink,
                datatypes,
                tabledatavar
            } = defineProps({
        	batchsizes: { type: Array, default: [25,50,100,200] },
			bitindicators: { type: Array,  default: ['Y','N'] },
			choices: { type: Array,  default: [] },
			debugmode: { type: Boolean,  default: true }, // false
			defaultsearches: { type: Array,  default: [] },
			defaultsorts: { type: Array,  default: [] },
			detaillink: { type: String,  default: '/' },
			datatypes: { type: Array, default: [] },
			tabledatavar: { type: String,  default: "window.tommyTableData" }
        });

        
        const loadNums = batchsizes;
        const loadNum = ref(batchsizes[0]);
        const shownNum = ref(batchsizes[0]);
        let searchFields = [];
        let searchMethods = [];
        const sortSeqRef = ref([]); // sequence of sort fields.  These will be pushed/popped so can sort by multiple columns
        const sortSeqCycleRef = ref([]); // the current "cycle" corresponding to each sortSeq element above (0=asc, 1=desc, 2=remove).  the cycle is bumped every time field is clicked for sort
        let actualCands = []; // the entire supplied JSON data
        const searchCandsRef = ref([]);  // actualCands are copied over here if any of criteria match (also removed from here if doesn't)
        const useDataTypes = ref(datatypes);
        
        const searchFieldsRef = ref(null);

        const defaultHtmlEntities = {
            clearSearch: '&#9851', // Universal Recycling Symbol
            loadMore: '&#9660;', // Down-Pointing Triangle
            sortDown: '&darr;', // DOWNWARDS ARROW
            sortUp: '&uarr;', // UPWARDS ARROW
        }
        
        actualCands = [...(eval(tabledatavar) || [{}])];
        searchCandsRef.value = [...actualCands];

        const getAllKeys = function() {
				return actualCands[0] && Object.keys(actualCands[0]) || {};
			};

        useDataTypes.value = [...useDataTypes.value, ...Array(getAllKeys().length-useDataTypes.value.length).fill('string')];
        
		onMounted(() => {
			initSearchFields();
			initSearchMethods();
			handleDefaultSearches();
			handleDefaultSorts();

			if (debugmode) {
				console.log(`** tommyTable intialized with the following: **`);
				console.log(`Total records: ${actualCands.length}`);
				console.log(`Records after search(s): ${searchCandsRef.value.length}`);
				console.log(`Showing: ${shownNum.value}`);
				console.log(`Data types: ${useDataTypes.value}`);
				console.log(`Search fields: ${searchFields}`);
				console.log(`Search methods: ${searchMethods}`);
				console.log(`Sorting on: ${sortSeqRef.value}`);
				console.log(`Sort directions: ${sortSeqCycleRef.value.map((e) => e ? 'desc' : 'asc')}`);
			}
		});
        
			// sorting
			
			const doSort = function(key, cycle) {
                const searchCands = searchCandsRef.value;
                const sortSeq = sortSeqRef.value;
                const sortSeqCycle = sortSeqCycleRef.value;

				const defaultCycle = typeof cycle === 'number' ? cycle : 0; // 0=asc
				var keypos =0;
				if ( typeof(key) != 'undefined' ) {
					keypos = sortSeq.indexOf(key);
					if ( keypos == -1 ) {
						sortSeqCycle.push(defaultCycle);
						sortSeq.push( key );
						keypos = sortSeq.length-1;
					}else {
						if ( ++sortSeqCycle[keypos] == 2 ) { // 2=remove
							sortSeqCycle.splice(keypos, 1);
							sortSeq.splice(keypos, 1);
							if ( keypos == sortSeq.length ) {
								return; // nothing to sort (removed last sort item)
							}else {
								//key = sortSeq[keypos++]; // set key to next sort item (removed in middle)								
							}
						}
					}
				}				

				for ( var k=keypos; k<sortSeq.length; k++ ) {
					var key = sortSeq[k];
					var arr = sortSeq.slice();
					var parentsortkey = ( k>0 ) ? arr[sortSeq.indexOf(key)-1] : ""; // for multiple field sorting						
					var useSortSeqCycle = sortSeqCycle[sortSeq.indexOf(key)];						
					
					if ( arr.length == 1) {
						searchCands.sort(function(a, b){
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
						for ( var i=0; i<=searchCands.length; i++ ) {							
							if ( i==searchCands.length || searchCands[i][parentsortkey] != lastparentval ) {
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
							if ( i<searchCands.length ) {
								lastparentval = searchCands[i][parentsortkey];
								searchcandstmp.push( searchCands[i] );
							}

						}
						searchCandsRef.value = [];
						searchCandsRef.value = searchcandstmp2.slice();						
					}
				}
                sortSeqRef.value = sortSeq;
                sortSeqCycleRef.value = sortSeqCycle;
			};


			// searching
			
			const doSearch = function() {
                var arrMatch = [];
				for ( var i=0; i<actualCands.length; i++ ) {
                    var j=0;
					for ( var key in actualCands[i] ) {
                        var val=searchFields[j];
						var fn = searchMethods[j];
						if ( val != "" ) {
							var re = new RegExp(val, 'i');
							if ( !actualCands[i][key].toString().match(re) ) {
								break;
							}
						}
						else {
							if (!fn(actualCands[i][key].toString())) {
								break;
							}							
						}
						j++;
					}
					if ( j == Object.keys(actualCands[i]).length ) {
						arrMatch.push(actualCands[i]);
					}
				}
                searchCandsRef.value = [];
				for ( var i=0; i<arrMatch.length; i++ ) {
					searchCandsRef.value.push(arrMatch[i]);
				}
                doSort();
			};				
			const updateSearchField = function(i, val) {
                searchFields[i]=val;
				doSearch();
			};
			const updateSearchMethod = function(i, fn) {		
                searchMethods[i]=fn;
				doSearch();
			};
			const clearSearch = function() {
				searchCandsRef.value = actualCands.slice();
				initSearchMethods();
				resetSearchFields();
				sortSeqRef.value=[];
				sortSeqCycleRef.value=[];					
			};				
			const resetSearchFields = function() {				
				initSearchFields();				
				searchFieldsRef.value.reset();
			};
            
			const initSearchFields = function() {
				searchFields = Array(getAllKeys().length).fill('');
			};
			const initSearchMethods = function() {
				searchMethods = Array(getAllKeys().length).fill(() => true);
			};
			const handleDefaultSearches = function() {
				useDefaultSearches.forEach(function(e, i) {
					if (/object/i.test(typeof e)) {
						updateSearchMethod(i, choices[i].options[e.choiceCriteraIndex] && choices[i].options[e.choiceCriteraIndex].criteria || function() { return true });

						// update DOM
						document.querySelector(`#tommyTable_select_${i}`).selectedIndex = e.choiceCriteraIndex;
					}
					else {
						updateSearchField(i, e.toString());

						// update DOM
                        if (/bit/i.test(useDataTypes.value[i])) {
							document.querySelector(`#tommyTable_radio_y_${i}`).checked = !!e;
							document.querySelector(`#tommyTable_radio_n_${i}`).checked = !!!e;
						}						
					}					
				});
				doSearch();
			};
			const handleDefaultSorts = function() {
				if (!useDefaultSorts.length) return;

				useDefaultSorts.forEach(function(e, i) {
					if (e && /^(asc|desc)$/i.test(e)) {
                        doSort(Object.keys(actualCands[0])[i], /asc/i.test(e) ? 0 : 1);
					}
				});
			};


			// getters
			
			const getSortSeqKeyPos = function(key) {
				return sortSeqRef.value.indexOf(key);
			};
			const getKeySortSeqCycle = function(key) {
				return sortSeqCycleRef.value[sortSeqRef.value.indexOf(key)];
			};
			const getNumSortSeqs = function() {
				return sortSeqRef.value.length;
			};
			const getHtmlEntity = function(k) {
				const h = defaultHtmlEntities[k];
				if (h) {
					return h;
				}
				throw (`HTML enitity '${k}' not found`);
			};
			

			// loading
			
			const loadMore = function() {
				shownNum.value += loadNum.value;
			};

            watch(loadNum, (v) => {
                if (v > shownNum.value) {
                    shownNum.value = v;
                }
                else if (v === 0) {
                    shownNum.value = actualCands.length;
                }
		    });


			// display
			
			const formatValue = function(s, i, override) {				
                switch(override || useDataTypes.value[i]) {
					case 'money':
						return new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(s);

					case 'bit':
						return s ? useBitindicators[0] : useBitindicators[1];

					default: // 'string'/undefined (empty)
						return s;
				}					
			};

			const prettifyHeader = (s) => s.replace(/_/g, ' ').replace(/\b([a-z]{1})/g, (s, m) => m.toUpperCase());    
</script>

<style scoped>
    tbody tr:nth-child(odd) {
        background-color: #FFF;
    }
    tbody tr:nth-child(even) {
        background-color: #f0f0f0;
    }
</style>