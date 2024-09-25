<template>
  <div class="container mx-auto p-4">
    <section class="w-full max-w-screen-md shadow-xl mx-auto">
      <fwb-tabs v-model="activeTab" variant="underline" class="p-5">
        <fwb-tab name="search" title="Search">
          <form @submit.prevent="search">
            <fwb-input v-model="searchQuery" size="lg">
              <template #prefix>
                <svg aria-hidden="true" class="w-5 h-5 text-gray-500 dark:text-gray-400" fill="none"
                  stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
                  <path d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z" stroke-linecap="round" stroke-linejoin="round"
                    stroke-width="2" />
                </svg>
              </template>
              <template #suffix>
                <fwb-button :disabled="isLoading">
                  <span v-show="!isLoading">Search</span>
                  <fwb-spinner color="yellow" v-show="isLoading" />
                </fwb-button>
              </template>
            </fwb-input>

            <div class="flex mt-4 me-2">
              <fwb-radio v-model="searchType" label="Search All" value="all" />
              <fwb-radio v-model="searchType" label="Individuals" value="individual" />
              <fwb-radio v-model="searchType" label="Entities" value="entity" />
            </div>

            <div class="flex me-2 me-2">
              <fwb-radio v-model="searchCountry" label="Search All" value="all" />
              <fwb-radio v-model="searchCountry" label="Australia" value="au" />
              <fwb-radio v-model="searchCountry" label="New Zealand" value="nz" />
            </div>
          </form>
        </fwb-tab>
        <fwb-tab name="smartSearch" title="SmartSearch">
          <form @submit.prevent="smartSearch">
            <fwb-textarea v-model="smartSearchQuery" :rows="10" custom label="">
              <template #footer>
                <div class="flex items-center justify-between">
                  <fwb-button :disabled="isLoading">
                    <span v-show="!isLoading">Search</span>
                    <fwb-spinner color="yellow" v-show="isLoading" />
                  </fwb-button>
                  <div class="flex">
                    <div class="flex">
                      <fwb-radio class="mx-4" v-model="smartSearchType" label="Individuals" value="individual" />
                      <fwb-radio class="mx-4" v-model="smartSearchType" label="Entities" value="entity" />
                    </div>
                  </div>
                </div>
              </template>
            </fwb-textarea>
          </form>
        </fwb-tab>
      </fwb-tabs>
    </section>

    <section class="w-full max-w-screen-md mx-auto mt-8" v-if="searchData.length && activeTab === 'search'">
      <fwb-table striped>
        <fwb-table-head>
          <fwb-table-head-cell>Match</fwb-table-head-cell>
          <fwb-table-head-cell>Country</fwb-table-head-cell>
          <fwb-table-head-cell>Type</fwb-table-head-cell>
        </fwb-table-head>
        <fwb-table-body>
          <fwb-table-row v-for="(match, index) in searchData">
            <fwb-table-cell v-html="match.nameFormatted"></fwb-table-cell>
            <fwb-table-cell
              v-text="match.country === 'au' ? 'Australia' : match.country === 'nz' ? 'New Zealand' : 'Unknown'"></fwb-table-cell>
            <fwb-table-cell
              v-text="match.type === 'individual' ? 'Individual' : match.type === 'entity' ? 'Entity' : 'Unknown'"></fwb-table-cell>
          </fwb-table-row>
        </fwb-table-body>
      </fwb-table>
    </section>

    <section class="w-full max-w-screen-md mx-auto mt-8" v-if="smartSearchData.length && activeTab === 'smartSearch'">
      <template v-for="(match, index) in smartSearchData">
        <h2 class="my-4 font-bold">{{ match.q }}</h2>

        <fwb-alert class="border-t-4 rounded-none my-4" icon type="warning" v-show="!match.candidates.length">
          No results.
        </fwb-alert>

        <fwb-table striped v-show="match.candidates.length">
          <fwb-table-head>
            <fwb-table-head-cell>Match</fwb-table-head-cell>
            <fwb-table-head-cell>Country</fwb-table-head-cell>
          </fwb-table-head>
          <fwb-table-body>
            <fwb-table-row v-for="(candidate, index) in match.candidates">
              <fwb-table-cell v-html="candidate.nameFormatted"></fwb-table-cell>
              <fwb-table-cell
                v-text="candidate.country === 'au' ? 'Australia' : candidate.country === 'nz' ? 'New Zealand' : 'Unknown'"></fwb-table-cell>
            </fwb-table-row>
          </fwb-table-body>
        </fwb-table>
      </template>
    </section>

    <section class="w-full max-w-screen-md mx-auto mt-8" v-if="noResults">
      <fwb-alert class="border-t-4 rounded-none" icon type="warning">
        No results.
      </fwb-alert>
    </section>
  </div>
</template>

<script lang="ts" setup>
import { ref } from 'vue'
import axios from 'axios'
import {
  FwbTab,
  FwbTabs,
  FwbButton,
  FwbInput,
  FwbRadio,
  FwbSpinner,
  FwbAlert,
  FwbTable,
  FwbTableBody,
  FwbTableCell,
  FwbTableHead,
  FwbTableHeadCell,
  FwbTableRow,
  FwbTextarea
} from 'flowbite-vue'

const client = axios.create({
  baseURL: 'https://k4gvf03cfb.execute-api.ap-southeast-2.amazonaws.com',
  timeout: 30000
})

const activeTab = ref('search')

const searchQuery = ref('')
const searchType = ref('all')
const searchCountry = ref('all')

const smartSearchQuery = ref('')
const smartSearchType = ref('individual')

const searchData = ref([])
const smartSearchData = ref([])

const noResults = ref(false)
const isLoading = ref(false)

const search = async () => {
  const q = searchQuery.value.trim()

  if (!q) {
    return
  }

  searchData.value = []
  isLoading.value = true
  noResults.value = false

  const filter = {}

  if (searchType.value !== 'all') {
    filter.type = searchType.value
  }

  if (searchCountry.value != 'all') {
    filter.country = searchCountry.value
  }

  const { data: { results } } = await client.post('/search', { q, filter }).finally(() => isLoading.value = false)

  searchData.value = results
  noResults.value = !results.length
}

const smartSearch = async () => {
  const q = smartSearchQuery.value.trim()

  if (!q) {
    return
  }

  smartSearchData.value = []
  isLoading.value = true
  noResults.value = false

  const filter = { type: smartSearchType.value }
  const { data: { results } } = await client.post('/smart-search', { q, filter }).finally(() => isLoading.value = false)

  smartSearchData.value = results
  noResults.value = !results.length
}
</script>

<style>
em {
  color: rgb(14 116 144);
}
</style>