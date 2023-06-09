<template>
  <v-navigation-drawer dark color="primary" permanent>
    <v-list-item>
      <v-list-item-content>
        <v-list-item-title class="text-h6">
          Filter
        </v-list-item-title>
      </v-list-item-content>
    </v-list-item>

    <v-divider style="margin-top: 10px; margin-bottom: 10px;"></v-divider>

    <v-row justify="center" >
      <v-expansion-panels accordion>
        <v-expansion-panel v-for="([icon, text, content], i) in items" :key="i">
          <v-expansion-panel-header dark color="primary">
            <v-row>
            <v-icon style="margin-left: 20px; margin-right: 20px;">{{icon}}</v-icon>
            {{text}}
            </v-row>
          </v-expansion-panel-header>
          <v-expansion-panel-content color="secondary">
            <component v-bind:is="content"></component>
          </v-expansion-panel-content>
        </v-expansion-panel>
      </v-expansion-panels>
    </v-row>
  </v-navigation-drawer>
</template>

<script>
import PatentFilters from "@/components/PatentFilters";
import PatentTypeFilter from "@/components/PatentTypeFilter";
import {bus} from "@/main";
import axios from "axios";
import countryList from "@/countryList.json";
import typelist from "@/types.json";
import ConflictTypeFilter from "@/components/ConflictInstigatorFilter";
import ConflictTargetFilter from "@/components/ConflictTargetFilter";
import conflictData from "../USD_data.json";

export default {
  name: "FilterBar",
  components: {
    PatentFilters
  },


  /**
   * Listen to relevant events and prepare data for initial visualization
   * @returns {Promise<void>}
   */
  async created() {

    bus.$on('selected-countries', (selCountries) => {
      this.selCountries = selCountries
      this.filterPatents()
      this.filterEvents()
    })
    bus.$on('change', (from, to) => {
      this.dateTo = to
      this.dateFrom = from
      this.filterEvents()
      this.filterPatents()
    })
    bus.$on('selected-types', (selSections) => {
      this.selSections = selSections
    })
    bus.$on('selected-instigagors', (selInstigators) => {
      this.selInstigators = selInstigators
      this.filterEvents()
    })
    bus.$on('selected-targets', (selTargets) => {
      this.selTargets = selTargets
      this.filterEvents()
    })

    conflictData.forEach((event) => {
      this.instigatorList.push(event.ACTOR1)
    })
    conflictData.forEach((event) => {
      this.targetList.push(event.TARGET1)
    })

    //Returns all conflict and patent data, as yet unfiltered, for the initial visualization.
    this.filterEvents()
    this.filterPatents()
  },

  data: () => ({
    baseUrl: "http://84.252.122.16:3000/",
    tlsUrl: "http://84.252.122.16:3000/tls209?",
    tls: "tls209?",
    geoc: "geoc_app?",
    request: "",
    countryList: countryList,
    instigatorList : [],
    targetList: [],
    selCountries:[],
    sectionList: typelist,
    selSections: [],
    sectionFilterIDs:[],
    countryFilterIDs: [],
    responseIDs: [],
    selTargets: [],
    selInstigators: [],
    filteredEvents:[],
    dateFrom: "1960-1-1",
    dateTo: "2016-12-31",
    joinedPatents: new Map,
    idStr: "",
    first: true,
    res: [],
    items: [
      ['mdi-briefcase', 'Patents', PatentTypeFilter],
      ['mdi-layers', 'Regions', PatentFilters],
      ['mdi-format-list-bulleted-type', 'Conflict Instigator', ConflictTypeFilter],
      ['mdi-human', 'Conflict Target', ConflictTargetFilter],
    ],
  }),

  //TODO: Somewhere some id filter list needs to be emptied.
  watch: {

    /**
     * Filters patents per patent type
     *
     * @returns {Promise<void>}
     */
    selSections: async function () {

      //Prepare request string
      this.typeSearchStr = ""
      this.sectionFilterIDs = []

      if (this.selSections.length !== 0) {
        this.typeSearchStr = "or=(ipc_class_symbol.like." + this.selSections[0] + "*"
        this.selSections.forEach((type) => {
          this.typeSearchStr += (",ipc_class_symbol.like." + type + "*")
        })
        this.typeSearchStr += ")"
      }

      //Filter out IDs that were selected in Region filter
      //Note: Some patents are however registered in multiple countries. Results for unfiltered Countries may therefore appear.
      if(this.selCountries.length > 0) this.typeSearchStr += this.formatIDfilter(this.countryFilterIDs)

      //Create Request
      this.request = this.baseUrl + this.tls + this.typeSearchStr

      //Send request and save filtered IDS for further filtering
      this.sendRequest(this.request).then((response) => {

        if (response !== undefined) {
          response.forEach((type) => {
            this.sectionFilterIDs.push(type.appln_id)
          })

          //get Patents from geoc table and get relevant geo data
          this.filterPatents()
        }
      })
    },
  },

  methods: {

    /**
     * Gets patents taking filters into consideration
     * Processes returned data for further use
     * Emits data in events
     */
    sendPatentFilterRequest: function () {

      //Create request string
      this.request = this.baseUrl + this.geoc + this.regionSearchStr + this.typeSearchStr + "&&filing_date=lte." + this.dateTo + "&&filing_date=gte." + this.dateFrom

      //Send request and use returned IDS for further filtering
      this.sendRequest(this.request).then(async (response) => {

        if (response !== undefined) {

          //Postgrest API does not allow all IDs in query at once, break it into chunks
          let i, temporary
          let slice = 5000

          for (i = 0; i < response.length ; i += slice) {

            this.idStr = ""
            temporary = response.slice(i, i + slice);

            //Get missing data for pie chart
            temporary.forEach((tmp, index) => {

              this.countryFilterIDs.push(tmp.appln_id)
              this.joinedPatents.set(tmp.appln_id, tmp)
              this.joinedPatents.get(tmp.appln_id).patentType = []

              //Create request string
              if (index < temporary.length - 1) this.idStr += tmp.appln_id + ","
              else if (temporary.length === 1) this.idStr = tmp.appln_id
              else this.idStr += tmp.appln_id
            })

            this.request2 = this.baseUrl + this.tls + "appln_id=in.(" + this.idStr + ")"

            //send request with max. 5000 ids
            this.sendRequest(this.request2).then(async (response2) => {

              if (response2 !== undefined) {
                // maps patent ID to json object with all necessary information from both tables for pie chart
                // some patents have the patent type missing
                response2.forEach((type) => {
                  if (this.joinedPatents.has(type.appln_id)) {
                    let tmp = this.joinedPatents.get(type.appln_id)
                    tmp.patentType.push(type.ipc_class_symbol)
                    this.joinedPatents.set(type.appln_id, tmp.valueOf())
                  }
                })

                if (i + slice > response.length) {
                  bus.$emit('filtered-map', this.joinedPatents)
                }
              }
            })
          }
          //emit filtered Patents
          bus.$emit('filtered-patents', response)
        }
      })
    },

    /**
     * Prepares request string for patents
     * @returns {Promise<void>}
     */
    filterPatents: async function() {
      //Prepare request string
      this.countryFilterIDs = []
      this.typeSearchStr = ""
      this.regionSearchStr = ""
      this.idStr = ""

      //Prepare request string
      if (this.selCountries.length !== 0) {
        this.regionSearchStr = "name_0=in.(" + this.selCountries[0]
        this.selCountries.forEach((country) => {if (this.selCountries[0]!== country) this.regionSearchStr += ("," + country)})
        this.regionSearchStr += (")")
      }

      //Add IDs (if any) that were selected in section filter to request string
      if(this.selSections.length > 0) {
        let i, temporary
        let slice = 5000
        //Postgrest API does not allow all IDs in query at once, break it into chunks
        for (i = 0; i < this.selSections.length; i += slice) {

          this.idStr = ""
          temporary = this.selSections.slice(i, i + slice);

          temporary.forEach(() => {
            if(this.selSections.length > 0) {
              this.typeSearchStr = this.formatIDfilter(this.temporary)
            }
          })
          this.sendPatentFilterRequest();
        }
      } else {
        this.sendPatentFilterRequest()
      }
    },

    /**
     * Filters USD conflict database
     * @returns {Promise<void>}
     */
    filterEvents: async function() {
      //map function to rename countries to match conflict dataset
      const mapConflictCountry = new Map([
        ["Côte d'Ivoire", "Cote d'Ivoire"],
        ["United Arab Emirates (the)", "United Arab Emirates"],
        ["Congo (the)", "Congo"],
        ["Venezuela (Bolivarian Republic of)", "Venezuela"],
        ["Viet Nam", "Vietnam"],
        ["Syrian Arab Republic", "Syria"],
        ["Tanzania, United Republic of", "Tanzania"],
        ["Sudan (the)", "Sudan"],
        ["Congo (the Democratic Republic of the)", "Congo, DRC"],
        ["Bolivia (Plurinational State of)", "Bolivia"],
        ["Philippines (the)", "Philippines"],
        ["Niger (the)", "Niger"],
        ["Dominican Republic (the)", "Dominican Republic"],
        ["Korea (the Republic of)", "South Korea"],
        ["Iran (Islamic Republic of)", "Iran"],
        ["Lao People's Democratic Republic (the)", "Laos"]
      ]);

      //Prepare filter values
      let str1, str2, str3
      let fromDate = new Date(this.dateFrom)
      let toDate = new Date(this.dateTo)

      if (this.selCountries.length < 1) str1 = this.countryList
      else str1 = this.selCountries
      if (this.selInstigators.length < 1) str2 = this.instigatorList
      else str2 = this.selInstigators
      if (this.selTargets.length < 1) str3 = this.targetList
      else str3 = this.selTargets

      //rename country list to work with conflict dataset
      str1 = str1.map(countryName => (mapConflictCountry.get(countryName) || countryName));

      //filter USD Database
      let data = conflictData,
          filterBy = { COUNTRY: str1, ACTOR1: str2, TARGET1: str3},
          result = data.filter(object => Object.keys(filterBy).every(key => filterBy[key].some(filter => object[key] === filter)));

      let filteredUSD = result.filter(function (event) {
        return event.BYEAR >= fromDate.getFullYear() && !(event.BYEAR > toDate.getFullYear());
      });

      //console.log("USD has " + filteredUSD.length + " matches");

      //Emit Filtered USD database entries
      bus.$emit('filtered-USD', filteredUSD)
    },

    /**
     * Formats request string
     * @param ids
     * @returns {string}
     */
    formatIDfilter(ids) {
      let str = ""
      if(ids !== undefined && ids.length > 0) {
        str += "&&appln_id=in.("
        ids.forEach((id, idx) => {
          str += id
          if (idx !== ids.length -1) str += ","
        })
        str += ")"
      }
      return str
    },

    /**
     * Takes a request string and sends a paginated get request. Returns request data
     * @param req
     * @param start
     * @param end
     * @param first
     * @returns {Promise<AxiosResponse<any>>}
     */
    sendRequest: async function(req, start=0, end, first = true) {

      //offset and limit for pagination
      //The limit and small offset are necessary since either the server can't keep up or the configuration isn't ideal.
      let offset = 2000
      let maxLimit = 8000

      //prepares header
      let config = {
        headers: {
          //"range-unit": "items",
          //"range": start + "-" + end,
          "prefer": "count=planned",
        }
      }

      //Prepare request string
      let reqString = req + "&&limit=" + offset + "&&offset=" + start

      return axios.get(reqString, config)
          .then( response => {

                //console.log("Fetched " + response.data.length + " results")

                let max = parseInt (response.headers["content-range"].split("/")[1])
                let curSt = parseInt(response.headers["content-range"].split("/")[0].split("-")[0])
                let curEnd = parseInt(response.headers["content-range"].split("/")[0].split("-")[1])

                if (first) this.res = response.data
                else this.res = [...this.res , ... response.data]

                curSt += offset
                curEnd += offset

                //Pagination. Recursively gets page ranges until row limit is reached
                //Otherwhise, the min operation can be removed and the offset made bigger.
                if((curEnd - 1)  <=  Math.min(max,maxLimit)) {
                  return this.sendRequest(req, curSt, "", false)
                }
                return this.res
              }
          )
          .catch(error => {
            this.errorMessage = error.message;
            console.error("There was an error!", error)
          })
    },
  },
}

</script>

<style scoped>

</style>
