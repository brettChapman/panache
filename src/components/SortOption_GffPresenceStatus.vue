<template>
  <div>
<!--    <b-form-group label="Tagged input using dropdown" label-for="tags-with-dropdown">-->
    <b-form-group class="noMarginBottom">
      <b-form-tags id="tags-with-dropdown" v-model="tagNames" no-outer-focus class="revokeBootstrapCSS">
        <template v-slot="{ tags, disabled, addTag, removeTag }">
          <!-- SEARCH BAR -->
          <b-dropdown size="sm" variant="outline-secondary" class="mb-2" block menu-class="w-100">
            <template #button-content>
              <b-icon icon="tag-fill"></b-icon> Select annotation(s)
            </template>
            <b-dropdown-form @submit.stop.prevent="() => {}">
              <b-form-group
                  label="Search annotations"
                  label-for="tag-search-input"
                  label-cols-md="auto"
                  class="mb-2 displayBlock"
                  label-size="sm"
                  :disabled="disabled"
              >
                <b-form-input
                    v-model="search"
                    id="tag-search-input"
                    type="search"
                    size="sm"
                    autocomplete="off"
                ></b-form-input>
              </b-form-group>
            </b-dropdown-form>
            <b-dropdown-divider></b-dropdown-divider>
            <!-- AVAILABLE OPTIONS IN SEARCH BAR -->
            <b-dropdown-item-button
                v-for="option in matchingAnnotNames"
                :key="option"
                @click="onOptionClick({ option, addTag })"
            >
              <div class="ellipsis">{{ option }}</div>
            </b-dropdown-item-button>
            <!-- INFORMATIONS ABOUT YOUR RESEARCH -->
            <b-dropdown-text v-if="searchCriteria.length >= searchMinChar && matchingAnnotNames.length === 0">
              There are no tags available to select
            </b-dropdown-text>
            <b-dropdown-text v-else-if="searchCriteria.length < searchMinChar && matchingAnnotNames.length === 0">
              The research should be at least {{ searchMinChar }} characters long
            </b-dropdown-text>
          </b-dropdown>

          <!-- SORTING BUTTON -->
          <b-button v-show="tagNames.length > 0"
                    block
                    class="mb-2 buttonPanache"
                    size="sm"
                    @click="sortByTags">
            Sort with tags
          </b-button>

          <!-- POPUP OF INFORMATIONS ABOUT THE GENOMES -->
          <div v-show="hasSearchHappened">
            <b-button
                class="mb-2 buttonPanache"
                size="sm"
                block
                id="popover-target-1">
              Informations about search
            </b-button>
            <b-popover target="popover-target-1" triggers="hover" placement="top">
              <template #title>Search criteria percentages</template>
              <div v-for="genomeInfo in popupDiv" :key="genomeInfo">
                {{ genomeInfo }}
              </div>
            </b-popover>
          </div>

          <!-- LIST OF THE ANNOT TAGS THAT ARE SELECTED AS FILTER(S) -->
          <div :key="componentKey">
            <b-input-group v-for="tag in tags" :key="tag" size="sm" v-show="mapOfPresenceStatus.has(tag)" :prepend="mapOfPresenceStatus.get(tag) ? 'Presence' : 'Absence'" class="mb-2 tagCustomCSS">
              <b-input-group-prepend is-text>
                <!-- CHECKBOX TO CHANGE THE DESIRED PRESENCE STATUS OF THE ANNOT -->
                <b-form-checkbox
                    switch
                    class="mr-n2 noBorderLeft"
                    :class="'noBorderLeft'"
                    :checked="mapOfPresenceStatus.get(tag)"
                    @change="updateStatus(tag)"
                    :id="'switch_' + tag">
                </b-form-checkbox>
              </b-input-group-prepend>
              <b-form-input class="inputTag" disabled aria-label="Large text input with switch" :placeholder="tag"></b-form-input>
              <b-input-group-append is-text>
                <b-icon icon="x" @click="removeTag(tag)" class="cursorP"/>
              </b-input-group-append>
            </b-input-group>
          </div>
        </template>
      </b-form-tags>
    </b-form-group>
  </div>
</template>

<script>
import {mapActions, mapState} from "vuex";
import {nonReactiveDataStore} from '@/store/non-reactive-data';

export default {
  name: "SortOption_GffPresenceStatus",
  data() {
    return {
      search: '',
      tagNames: [], //TODO Rename, is it the possible names for annots? Name from bootstrap tags?
      searchMinChar: 3,
      mapOfPresenceStatus: new Map(),
      hasSearchHappened: false,
      popupDiv: [],
      componentKey: 0,
    }
  },
  computed: {
    ...mapState({
      annotMap: 'annotMap', //TODO change to annot in store
      genomeList: 'genomeListInDisplay',
    }),
    /**
     * Function that returns the research criteria.
     * @returns {string} = research criteria
     */
    searchCriteria() {
      // Compute the search criteria
      return this.search.trim().toLowerCase();
    },
    /**
     * Function that returns the named annotations matching to the research criteria.
     * @returns {Array} = list of all matching annotation names, empty if none found
     */
    matchingAnnotNames() {
      let criteria = this.searchCriteria;

      if (criteria.length >= this.searchMinChar) {
        // From annotMap, get names matching the search criteria
        let matchingNames = [...this.annotMap.keys()].filter(opt => opt.toLowerCase().indexOf(criteria) > -1);

        // Filter out already selected options
        return matchingNames.filter(opt => this.tagNames.indexOf(opt) === -1);

      } else { return [] }
    },
  },
  methods: {
    ...mapActions([
      'updateGenomesInDisplay',
    ]),
    /**
     * Add a named annot to tags (a Bootstrap.Vue list).
     * @param option = Named annot to add
     * @param addTag = function addTag of Bootstrap.Vue
     */
    onOptionClick({ option, addTag }) {
      addTag(option); // Add the annot to the tag list
      this.search = ''; // Empty the search bar
    },
    /**
     * Sort the genomes based on their scores.
     */
    sortByTags() {

      let scorePerGenome = this.applyScoresToGenomes();

       // Sort the ranking Map in function of their number of points.
      let rankGenomes = new Map([...scorePerGenome.entries()].sort((a, b) => b[1] - a[1]));

      // Update the popup displaying the scores
      this.popupDiv = []; // Clear previous info
      rankGenomes.forEach( (score, genome) => {
        this.popupDiv.push(`${genome} (${score}%)`);
      });

      // Update the store with the new genome order
      this.updateGenomesInDisplay([...rankGenomes.keys()]);
      this.hasSearchHappened = true; // Enables the info popup to appear on screen
    },
    /**
     * Update the status of research of an annotation.
     * @param annot = The annot to which you want to update the status
     */
    updateStatus(annot) {
      this.mapOfPresenceStatus.set(annot, !this.mapOfPresenceStatus.get(annot))
      this.hasSearchHappened = false; // Turn hasSearchHappened to false to not misinform the user and hide the popup
      this.forceRerender();
    },
    /**
     * Function that force area with componentKey as key to rerender.
     */
    forceRerender() {
      this.componentKey += 1;
      if (this.componentKey > 2) { // To avoid a large number
        this.componentKey = 0;
      }
    },
    /**
     * Function that parses all the annotName x presenceStatus filters and
     * computes a matching score for every genome, as a rounded percent of match.
     * @returns {Map} = mapOfScores[genome] --> score
     */
    applyScoresToGenomes() {

      // Map that stores the scores of each genome depending on how well they match the tags
      let mapOfScores = new Map();

      // Initializing at 0
      this.genomeList.forEach( geno => mapOfScores.set(geno, 0) );

      // Ranks genomes according to their matching score with the choosen tags
      this.mapOfPresenceStatus.forEach( (shouldBePresent, annotName) => {

        let annotStartStopChrom = this.annotMap.get(annotName);
        let searchCriteria = {
          'shouldBePresent': shouldBePresent,
          'annotStart': annotStartStopChrom[0], // Coordinate in nt of the start position of the annot
          'annotStop': annotStartStopChrom[1], // Coordinate in nt of the stop position of the annot
          'annotChrom': annotStartStopChrom[2], // Name of the chrom hosting the annot
        };

        // Find the panBlocks overlapping the annot and the breakpoints
        // that split the annot in multiple segments, each will have a part
        // of the overall matching score.
        let { spannedBlocks, arrayOfBreakpoints } = this.getBlocksSpannedByAnnotAndBkpts(searchCriteria);

        // Build mapOfBlocksPerBreakpoint that lists, for all breakpoints, which
        // panBlocks cover the subsequent interval
        //
        //Exemple:
        //  Annot |------------------------------------------------------------|
        //        |         |       |           |                    |
        //        <BKPT_A--><BKPT_B><BKPT_C----><BKPT_D-------------><BKPT_E--->
        //
        //    1   __________11111111111111111111________________________________
        //    2   222222222222222222222222222222222222222222222222222___________
        //    3   ______________________________333333333333333333333___________
        //    4   __________________444444444444444444444444444444444___________
        //
        // map.get(A) --> [2];
        // map.get(C) --> [1, 2, 4];
        // map.get(E) --> [];
        let mapOfBlocksPerBreakpoint = new Map();

        // For each breakpoint, stores the blocks present in the subsequent interval
        // It enables overlapping blocks
        arrayOfBreakpoints.forEach( bkpt => {
          mapOfBlocksPerBreakpoint.set(bkpt, []);
          spannedBlocks.forEach(block => {
            // Does the [breakpoint, nextBreakpoint[ interval has the block?
            if ( (block.FeatureStart <= bkpt) || (bkpt < block.FeatureStop ) ) {
              mapOfBlocksPerBreakpoint.get(bkpt).push(block)
            }
          });
        });

        this.increaseScorePerGenome(mapOfScores, arrayOfBreakpoints, mapOfBlocksPerBreakpoint, searchCriteria);

      });

      //console.log({mapOfScores});
      // Turn the score into a percentage of matches
      mapOfScores.forEach( (score, geno ) => mapOfScores.set(geno, Math.round(score / this.mapOfPresenceStatus.size * 100)));

      //console.log({mapOfScores});
      return mapOfScores;
    },
    /**
     * Function that returns all the panBlocks overlaped by a given annotation
     * as well as the coordinates of breakpoints within that annot, corresponding
     * to the borders of said panBlocks when they are contained within the annot.
     * @param searchCriteria = (object) Contains annotStart, annotStop, and annotChom
     * @returns {Array} = [panBlock_A, ..., panBlock_M]
     * @returns {Array} = [AnnotStart, BKPT_1, ..., BKPT_N, AnnotStop]
     */
    getBlocksSpannedByAnnotAndBkpts(searchCriteria) {

      let {annotStart, annotStop, annotChrom} = searchCriteria;

      let chromPavBlocks = [...nonReactiveDataStore.fullChromData[annotChrom]]; // Get the list of PAV blocks found in annotChrom

      // Find the blocks for which we should check the pav status
      // Here the condition is that at least a part of the block(s) should
      // overlap the annotation
      let setOfBreakpoints = new Set([annotStart]);
      let spannedBlocks = chromPavBlocks.filter( block => {

        let blockStartIsInAnnot = ( annotStart <= parseInt(block.FeatureStart) ) && ( parseInt(block.FeatureStart) <= annotStop );
        let blockStopIsInAnnot = ( annotStart <= parseInt(block.FeatureStop) ) && ( parseInt(block.FeatureStop) <= annotStop );
        let blockContainsAnnot = ( parseInt(block.FeatureStart) <= annotStart) && ( annotStop <= parseInt(block.FeatureStop) );

        let isMatching = (blockStartIsInAnnot || blockStopIsInAnnot || blockContainsAnnot);

        // Keep track of the breakpoints within the annotation, made
        // by the borders of PAV blocks
        if (isMatching) {
          setOfBreakpoints.add(Math.max(block.FeatureStart, annotStart));
          setOfBreakpoints.add(Math.min(block.FeatureStop, annotStop));
        }

        return isMatching
      });

      // Create the ordered array of breakpoints / intervals starting points
      let arrayOfBreakpoints = Array.from(setOfBreakpoints).sort((a, b) => a - b); // Sort in increasing order
      arrayOfBreakpoints.pop(); // Final breakpoint is useless since it does not start any interval to check
      arrayOfBreakpoints.reverse(); // Will be parsed from end to start later

      return {spannedBlocks, arrayOfBreakpoints};
    },
    /**
     * For each annotation chunk between breakpoints, computes the matching score
     * (ie. a proportion of the chunk's length depending on how well it matches)
     * based on the search criteria and the panBlocks found.
     * That score is then converted to a global score by cumulating the scores
     * of all chunks, returned by the function.
     * Increases the score obtained with previous search criterias / annot.
     * @param mapOfScores = (Map) map.get(geno) --> matchingScore, updated based on the current annot score
     * @param arrayOfBreakpoints = (Array) reverse ordered list of breakpoints
     * @param mapOfBlocksPerBreakpoint = (Map) with the list of panBlocks found within each segment
     * @param searchCriteria = (object) Contains shouldBePresent annotStop, and annotStart
     */
     increaseScorePerGenome(mapOfScores, arrayOfBreakpoints, mapOfBlocksPerBreakpoint, searchCriteria) {

      let {shouldBePresent, annotStart, annotStop} = searchCriteria;

      // Compute matching score for each genome
      this.genomeList.forEach( geno => {

        let cumulLenScore = 0;
        let nextBreakpoint = annotStop;
        let annotLength = annotStop - annotStart;

        // Parsing the breakpoints from right to left
        arrayOfBreakpoints.forEach( bkpt => {

          let intervalLength = nextBreakpoint - bkpt;
          let intervalScore = 0;
          let nbOfMatchingBlocks = 0;

          // Check PAV status of all candidate blocks
          mapOfBlocksPerBreakpoint.get(bkpt).forEach( block => {
            let pavStatus = parseInt(block[geno]);

            //Check if there is a match between desired and block presence/absence statuses
            let queryDoesMatch = (shouldBePresent && pavStatus > 0) || (!shouldBePresent && pavStatus === 0);

            if (queryDoesMatch) {
              nbOfMatchingBlocks += 1;
            }
          })

          // Apply the score to the length of that chunk
          let nbBlocksInInterval = mapOfBlocksPerBreakpoint.get(bkpt).length;
          if (nbBlocksInInterval > 0) {
            intervalScore = nbOfMatchingBlocks * intervalLength / nbBlocksInInterval;
          }

          cumulLenScore += intervalScore;
          nextBreakpoint = bkpt;
        })

        // The score is a float between 0 and 1, shows the proportion of the annot that matches the desired pav status
        let annotScore = cumulLenScore / annotLength;
        //console.log({cumulLenScore, annotLength, spannedBlocks})

        //console.log({geno, annotScore});
        mapOfScores.set(geno, mapOfScores.get(geno) + annotScore);

      });
    },
  },
  watch: {
    tagNames() {

      let currentTags = [...this.tagNames];

      // In Map, remove keys from deleted tags
      this.mapOfPresenceStatus.forEach( (val, key) => {

        if (currentTags.includes(key)) {
          let idxToDel = currentTags.indexOf(key);
          currentTags.splice(idxToDel, 1);

        } else {
          this.mapOfPresenceStatus.delete(key);
        }

      });

      // Add new (key, value) according to new tag
      currentTags.forEach( (annotName) => {
        this.mapOfPresenceStatus.set(annotName, true);
      });
    }
  }
}

</script>

<style scoped>

.width100P {
  width: 100%;
}

.revokeBootstrapCSS {
  padding: 0px;
  border: none;
}

.displayBlock {
  display: block;
}

.noMarginBottom {
  margin-bottom: 0;
}

.ellipsis {
  text-overflow: ellipsis;
  overflow: hidden;
}

.noBordeRight{
  border-right: none;
}

.noBorderLeft {
  border-left: none;
}

.cursorP {
  cursor: pointer;
}

.inputTag {
  color: #495057;
  background-color: white;
  height: 34px !important;
}

.inputTag::placeholder {
  color: #495057;
}

</style>

<style>

.tagCustomCSS .input-group-prepend:first-child .input-group-text {
  width: 78px !important;
}

</style>