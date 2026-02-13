<template>
  <v-container fluid class="pa-4">
    <v-card class="elevation-1" rounded="lg">
      <!-- HEADER - ELEGANT -->
      <v-card-title class="pa-5">
        <div class="d-flex align-center">
          <v-avatar color="warning" size="40" class="mr-4" variant="tonal">
            <v-icon icon="mdi-desktop-classic" color="warning" />
          </v-avatar>
          <div>
            <div class="d-flex align-center">
              <h2 class="text-h5 font-weight-medium text-warning-darken-1">Sign-In Logs</h2>
              <v-chip
                color="warning"
                variant="tonal"
                size="small"
                class="ml-4"
              >
                77 Columns
              </v-chip>
            </div>
            <div class="d-flex align-center mt-1">
              <span class="text-caption text-grey-darken-1 mr-3">
                Client-side pagination {{ visibleColumns.length }} visible
              </span>
              <v-chip
                size="x-small"
                color="warning"
                variant="tonal"
                prepend-icon="mdi-database"
                class="mr-2"
              >
                {{ allLogs.length }} records
              </v-chip>
            </div>
          </div>
        </div>
      </v-card-title>

      <!-- ACTION BAR - COLUMNS & REFRESH ON LEFT -->
      <v-card-text class="pa-4 pt-0">
        <v-row align="center" justify="space-between">
          <v-col cols="12" md="6" class="d-flex align-center">
            <!-- COLUMN VISIBILITY MENU -->
            <v-menu
              :close-on-content-click="false"
              location="bottom-start"
              width="380"
              offset="5"
            >
              <template #activator="{ props }">
                <v-btn
                  variant="outlined"
                  density="comfortable"
                  prepend-icon="mdi-table-column"
                  v-bind="props"
                  class="mr-3"
                  color="warning"
                  :disabled="allHeaders.length === 0"
                >
                  Columns
                  <v-chip size="x-small" color="warning" class="ml-2">{{ visibleColumns.length }}/{{ allHeaders.length || 77 }}</v-chip>
                </v-btn>
              </template>

              <v-card class="pa-4" elevation="4" rounded="lg" v-if="allHeaders.length > 0">
                <div class="d-flex align-center justify-space-between mb-3">
                  <span class="text-subtitle-2 font-weight-medium">Column Visibility ({{ allHeaders.length }} total)</span>
                  <div>
                    <v-btn size="x-small" variant="text" @click="showDefaultColumns" class="mr-1">Default (9)</v-btn>
                    <v-btn size="x-small" variant="text" @click="showAllColumns">All</v-btn>
                  </div>
                </div>
                <v-divider class="mb-3" />
                <div style="max-height: 400px; overflow-y: auto;">
                  <v-checkbox
                    v-for="col in columnOptions"
                    :key="col.value"
                    :label="col.title"
                    :model-value="col.selected"
                    @update:model-value="toggleColumn(col.value)"
                    density="compact"
                    hide-details
                    class="my-1"
                    color="warning"
                  />
                </div>
                <v-divider class="my-3" />
                <div class="text-caption text-grey-darken-1 text-center">
                  {{ visibleColumns.length }} columns currently visible
                </div>
              </v-card>
              <v-card class="pa-4" elevation="4" rounded="lg" v-else>
                <div class="text-caption text-center pa-4">
                  Loading...
                </div>
              </v-card>
            </v-menu>

            <v-btn
              variant="outlined"
              density="comfortable"
              prepend-icon="mdi-refresh"
              @click="fetchLogs"
              :loading="loading"
              color="warning"
            >
              Refresh
            </v-btn>
          </v-col>
          
          <v-col cols="12" md="6" class="text-right">
            <div v-if="hasActiveColumnFilters" class="d-inline-flex align-center">
              <v-icon icon="mdi-filter-variant" size="18" color="warning" class="mr-1" />
              <span class="text-caption text-grey-darken-1 mr-2">Filters active</span>
              <v-btn size="x-small" variant="text" @click="clearAllColumnFilters" color="warning">
                Clear all
              </v-btn>
            </div>
          </v-col>
        </v-row>
      </v-card-text>

      <!-- ACTIVE FILTERS CHIPS -->
      <v-card-text class="pa-4 pt-0" v-if="hasActiveColumnFilters">
        <div class="d-flex flex-wrap gap-2">
          <v-chip
            v-for="filter in activeColumnFilters"
            :key="filter.key"
            size="small"
            density="comfortable"
            color="orange-lighten-4"
            closable
            @click:close="clearColumnFilter(filter.key)"
            class="text-grey-darken-3"
          >
            <span class="font-weight-medium mr-1">{{ 
              getColumnTitle(filter.key)
            }}:</span>
            {{ truncateText(filter.value, 20) }}
          </v-chip>
        </div>
      </v-card-text>

      <v-divider />

      <!-- TABLE CONTAINER -->
      <div style="overflow-x: auto; overflow-y: hidden; position: relative;" v-if="headers.length > 0">
        <v-data-table-server
          :headers="headers"
          :items="pagedLogs"
          :items-length="filteredLogs.length"
          :loading="loading"
          :page="page"
          :items-per-page="pageSize"
          @update:page="page = $event"
          @update:items-per-page="pageSize = $event"
          class="elevation-0 elegant-table"
          :loading-text="'Loading audit logs...'"
          :no-data-text="'No sign-in logs found'"
          fixed-header
          height="65vh"
          density="compact"
        >
          <!-- CUSTOM HEADERS WITH FILTER ICONS -->
          <template #headers="{ columns }">
            <tr>
              <th v-for="col in columns" :key="col.key" class="elegant-header-cell">
                <div class="d-flex align-center">
                  <span class="elegant-header-text">{{ col.title }}</span>
                  
                  <!-- FILTER ICON -->
                  <v-menu
                    :close-on-content-click="false"
                    v-model="filterMenu[col.key]"
                    location="bottom"
                    offset="4"
                  >
                    <template #activator="{ props }">
                      <v-btn
                        icon
                        size="x-small"
                        variant="text"
                        v-bind="props"
                        :color="columnFilters[col.key] ? 'warning' : 'grey-lighten-1'"
                        density="compact"
                        class="filter-btn"
                      >
                        <v-icon :size="columnFilters[col.key] ? 16 : 14">
                          {{ columnFilters[col.key] ? 'mdi-filter' : 'mdi-filter-outline' }}
                        </v-icon>
                      </v-btn>
                    </template>

                    <v-card class="pa-3" width="260" elevation="8" rounded="lg">
                      <div class="text-body-2 font-weight-medium mb-2">
                        Filter by {{ col.title }}
                      </div>
                      <v-text-field
                        v-model="columnFilters[col.key]"
                        density="compact"
                        variant="outlined"
                        hide-details
                        :placeholder="`Search...`"
                        :prepend-inner-icon="columnFilters[col.key] ? 'mdi-filter-check' : 'mdi-filter'"
                        clearable
                        @click:clear="clearColumnFilter(col.key)"
                      />
                      <div class="text-caption text-grey-darken-1 mt-1">
                        <v-icon icon="mdi-information-outline" size="14" class="mr-1" />
                       Filtering
                      </div>
                    </v-card>
                  </v-menu>
                </div>
              </th>
            </tr>
          </template>

          <!-- DYNAMIC CELL RENDERING WITH TOOLTIPS -->
          <template #item="{ item }">
            <tr class="hover-row">
              <td v-for="col in headers" :key="col.key" class="elegant-cell">
                <template v-if="item[col.key] !== undefined && item[col.key] !== null && item[col.key] !== ''">
                  <!-- Date fields -->
                  <v-tooltip v-if="isDateField(col.key)" location="top" :text="formatFullDateTime(item[col.key])">
                    <template #activator="{ props }">
                      <div v-bind="props" class="d-inline-block">
                        <div class="text-caption font-weight-medium text-grey-darken-3">
                          {{ formatDate(item[col.key]) }}
                        </div>
                        <div class="text-caption text-grey-darken-1">
                          {{ formatTime(item[col.key]) }}
                        </div>
                      </div>
                    </template>
                  </v-tooltip>
                  
                  <!-- ID/GUID fields -->
                  <v-tooltip v-else-if="isIdField(col.key)" location="top" :text="item[col.key]">
                    <template #activator="{ props }">
                      <span v-bind="props" class="text-caption font-family-monospace text-grey-darken-3">
                        {{ truncateText(item[col.key], 12) }}
                      </span>
                    </template>
                  </v-tooltip>
                  
                  <!-- Email/UPN fields -->
                  <v-tooltip v-else-if="isEmailField(col.key)" location="top" :text="item[col.key]">
                    <template #activator="{ props }">
                      <span v-bind="props" class="text-body-2 text-truncate text-grey-darken-3 d-inline-block" 
                            style="max-width: 200px;">
                        {{ item[col.key] }}
                      </span>
                    </template>
                  </v-tooltip>
                  
                  <!-- Boolean fields -->
                  <v-chip
                    v-else-if="typeof item[col.key] === 'boolean'"
                    size="x-small"
                    density="compact"
                    :color="item[col.key] ? 'green-lighten-4' : 'grey-lighten-3'"
                    :class="item[col.key] ? 'text-green-darken-3' : 'text-grey-darken-2'"
                    label
                  >
                    {{ item[col.key] ? 'Yes' : 'No' }}
                  </v-chip>
                  
                  <!-- Status/Risk fields -->
                  <v-tooltip v-else-if="isStatusField(col.key)" location="top" :text="item[col.key]">
                    <template #activator="{ props }">
                      <v-chip
                        v-bind="props"
                        size="x-small"
                        density="compact"
                        :color="col.key.toLowerCase().includes('risk') ? getRiskColor(item[col.key]) : getStatusColor(item[col.key])"
                        :class="col.key.toLowerCase().includes('risk') ? getRiskTextColor(item[col.key]) : getStatusTextColor(item[col.key])"
                        label
                      >
                        {{ item[col.key] }}
                      </v-chip>
                    </template>
                  </v-tooltip>
                  
                  <!-- Number fields -->
                  <v-tooltip v-else-if="typeof item[col.key] === 'number'" location="top" :text="item[col.key].toLocaleString()">
                    <template #activator="{ props }">
                      <span v-bind="props" class="text-caption text-grey-darken-3 font-family-monospace">
                        {{ item[col.key].toLocaleString() }}
                      </span>
                    </template>
                  </v-tooltip>
                  
                  <!-- Default text fields -->
                  <v-tooltip v-else location="top" :text="String(item[col.key])">
                    <template #activator="{ props }">
                      <span v-bind="props" class="text-body-2 text-truncate text-grey-darken-3 d-inline-block" 
                            style="max-width: 180px;">
                        {{ truncateText(item[col.key], 25) }}
                      </span>
                    </template>
                  </v-tooltip>
                </template>
                <span v-else class="text-caption text-grey-lighten-1">—</span>
              </td>
            </tr>
          </template>

          <!-- LOADING STATE -->
          <template #loading>
            <tr>
              <td :colspan="headers.length || 1" class="pa-10 text-center">
                <v-progress-circular
                  indeterminate
                  color="warning"
                  size="40"
                  width="2"
                />
                <div class="text-body-2 text-grey-darken-1 mt-4">
                  Loading 500 records...
                  <span v-if="hasActiveColumnFilters" class="d-block mt-1">Applying filters</span>
                </div>
              </td>
            </tr>
          </template>

          <!-- NO DATA STATE -->
          <template #no-data>
            <tr>
              <td :colspan="headers.length || 1" class="pa-10 text-center">
                <v-icon 
                  :icon="hasActiveColumnFilters ? 'mdi-filter-off' : 'mdi-file-search-outline'" 
                  size="56" 
                  class="text-grey-lighten-2 mb-4" 
                />
                <h4 class="text-h6 font-weight-regular mb-2 text-grey-darken-2">
                  {{ hasActiveColumnFilters ? 'No matching records' : 'No sign-in logs' }}
                </h4>
                <p class="text-body-2 text-grey-darken-1 mb-4">
                  {{ hasActiveColumnFilters ? 'Try adjusting your filters' : 'Data will appear here once available' }}
                </p>
                <div>
                  <v-btn
                    v-if="hasActiveColumnFilters"
                    variant="outlined"
                    @click="clearAllColumnFilters"
                    prepend-icon="mdi-filter-remove"
                    class="mr-2"
                    color="warning"
                  >
                    Clear Filters
                  </v-btn>
                  <v-btn
                    variant="tonal"
                    @click="fetchLogs"
                    prepend-icon="mdi-refresh"
                    color="warning"
                  >
                    Refresh
                  </v-btn>
                </div>
              </td>
            </tr>
          </template>

          <!-- TABLE FOOTER -->
          <template #bottom>
            <div class="text-caption text-grey-darken-1 px-4 py-3 border-t d-flex justify-space-between">
              <div>
                <span v-if="hasActiveColumnFilters" class="mr-3">
                  <v-icon icon="mdi-filter-check" size="16" class="mr-1" color="warning" />
                  <span class="text-warning-darken-1">Client-side filtered</span>
                </span>
                Showing {{ showingFrom }}-{{ showingTo }} of {{ filteredLogs.length }} records
                <span class="text-grey-lighten-1 mx-2">|</span>
                <span class="text-grey-darken-1">{{ allLogs.length }}/500 loaded</span>
              </div>
              <div class="d-flex align-center">
                <span class="mr-2">Page {{ page }} of {{ totalPages }}</span>
                <span class="text-grey-lighten-1 mx-2">|</span>
                <span>{{ headers.length }} columns</span>
              </div>
            </div>
          </template>
        </v-data-table-server>
      </div>

      <!-- EMPTY STATE WHEN NO HEADERS -->
      <div v-else-if="!loading && allLogs.length === 0" class="pa-10 text-center">
        <v-icon icon="mdi-file-search-outline" size="56" class="text-grey-lighten-2 mb-4" />
        <h4 class="text-h6 font-weight-regular mb-2 text-grey-darken-2">No sign-in logs</h4>
        <p class="text-body-2 text-grey-darken-1 mb-4">Data will appear here once available</p>
        <v-btn variant="tonal" @click="fetchLogs" prepend-icon="mdi-refresh" color="warning">
          Refresh
        </v-btn>
      </div>

      <!-- PAGINATION CONTROLS - ONLY SHOW WHEN THERE'S DATA -->
      <template v-if="headers.length > 0 && filteredLogs.length > 0">
        <v-divider />
        <v-card-actions class="pa-4">
          <v-row align="center" justify="space-between" dense>
            <v-col cols="12" md="3">
              <div class="d-flex align-center">
                <span class="text-caption text-grey-darken-1 mr-3">
                  Rows per page:
                </span>
                <v-select
                  v-model="pageSize"
                  :items="[10, 25, 50, 100]"
                  density="compact"
                  variant="outlined"
                  hide-details
                  style="width: 110px"
                />
              </div>
            </v-col>
            
            <v-col cols="12" md="6" class="text-center">
              <v-pagination
                v-model="page"
                :length="totalPages"
                :total-visible="7"
                density="comfortable"
                color="warning"
                rounded
              />
            </v-col>
            
            <v-col cols="12" md="3" class="text-right">
              <div class="text-caption text-grey-darken-1">
                {{ pagedLogs.length }} rows shown
              </div>
            </v-col>
          </v-row>
        </v-card-actions>
      </template>
    </v-card>
  </v-container>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import axios from 'axios'

/* ---------- CONFIG ---------- */
const API_BASE = import.meta.env.VITE_API_BASE_URL

/* ---------- STATE ---------- */
const loading = ref(false)
const allLogs = ref([])
const page = ref(1)
const pageSize = ref(25)
const fromDate = ref(null)
const toDate = ref(null)
const filterMenu = ref({})

/* ---------- DEFAULT VISIBLE COLUMNS (9 COLUMNS) ---------- */
const defaultVisibleColumns = [
  'createdDateTime',
  'userDisplayName',
  'userPrincipalName',
  'appDisplayName',
  'status',
  'ipaddress',
  'location',
  'clientAppUsed',
  'riskLevelAggregated'
]

/* ---------- ALL 77 COLUMNS (DYNAMIC) ---------- */
const allHeaders = ref([])
const visibleColumns = ref([...defaultVisibleColumns])

/* ---------- HEADERS FOR DISPLAY ---------- */
const headers = computed(() => {
  return allHeaders.value.filter(h => visibleColumns.value.includes(h.key))
})

/* ---------- COLUMN FILTERS ---------- */
const columnFilters = ref({})

// Initialize filters for default columns first
defaultVisibleColumns.forEach(key => {
  columnFilters.value[key] = ''
})

/* ---------- COMPUTED PROPERTIES ---------- */
const hasActiveColumnFilters = computed(() =>
  Object.values(columnFilters.value).some(v => v && v.trim() !== '')
)

const activeColumnFilters = computed(() => {
  return Object.entries(columnFilters.value)
    .filter(([_, value]) => value && value.trim() !== '')
    .map(([key, value]) => ({ key, value }))
})

const showingFrom = computed(() => {
  if (filteredLogs.value.length === 0) return 0
  return (page.value - 1) * pageSize.value + 1
})

const showingTo = computed(() => {
  const to = page.value * pageSize.value
  return Math.min(to, filteredLogs.value.length)
})

const columnOptions = computed(() => {
  return allHeaders.value.map(h => ({
    title: h.title,
    value: h.key,
    selected: visibleColumns.value.includes(h.key)
  }))
})

/* ---------- API - FETCH ALL 500 RECORDS ---------- */
const fetchLogs = async () => {
  loading.value = true
  try {
    const res = await axios.get(`${API_BASE}/api/client/signin-logs`, {
      params: {
        fromDate: fromDate.value,
        toDate: toDate.value
      }
    })

    allLogs.value = res.data?.data || []
    
    // Generate headers from the first row of data
    if (allLogs.value.length > 0) {
      generateHeaders(allLogs.value[0])
      
      // Initialize filters for all columns from the data
      Object.keys(allLogs.value[0]).forEach(key => {
        if (columnFilters.value[key] === undefined) {
          columnFilters.value[key] = ''
        }
      })
    }
    
    page.value = 1
  } catch (err) {
    console.error("API ERROR:", err)
    allLogs.value = []
  } finally {
    loading.value = false
  }
}

/* ---------- HEADER GENERATION ---------- */
const formatHeader = (key) => {
  return key
    .replace(/([A-Z])/g, ' $1')
    .replace(/^./, str => str.toUpperCase())
    .replace('Id', 'ID')
    .replace('Url', 'URL')
    .replace('Ip', 'IP')
    .replace('Api', 'API')
    .replace('Sso', 'SSO')
    .replace('Mfa', 'MFA')
}

const generateHeaders = (firstRow) => {
  if (!firstRow) return
  
  allHeaders.value = Object.keys(firstRow).map(key => ({
    title: formatHeader(key),
    key,
    width: '160px',
    sortable: true
  }))
}

/* ---------- HELPER TO GET COLUMN TITLE ---------- */
const getColumnTitle = (key) => {
  const header = allHeaders.value.find(h => h.key === key)
  return header?.title || key
}

/* ---------- COLUMN VISIBILITY MANAGEMENT ---------- */
const toggleColumn = (key) => {
  if (visibleColumns.value.includes(key)) {
    visibleColumns.value = visibleColumns.value.filter(k => k !== key)
  } else {
    visibleColumns.value = [...visibleColumns.value, key]
  }
}

const showDefaultColumns = () => {
  visibleColumns.value = [...defaultVisibleColumns]
}

const showAllColumns = () => {
  if (allHeaders.value.length > 0) {
    visibleColumns.value = allHeaders.value.map(h => h.key)
  }
}

/* ---------- CLIENT-SIDE FILTERING ---------- */
const filteredLogs = computed(() => {
  if (allLogs.value.length === 0) return []
  
  return allLogs.value.filter(log => {
    return Object.entries(columnFilters.value).every(([key, filter]) => {
      if (!filter || filter.trim() === '') return true
      const value = log[key]
      if (value === undefined || value === null) return false
      return value.toString().toLowerCase().includes(filter.toLowerCase())
    })
  })
})

/* ---------- CLIENT-SIDE PAGINATION ---------- */
const pagedLogs = computed(() => {
  if (filteredLogs.value.length === 0) return []
  const start = (page.value - 1) * pageSize.value
  return filteredLogs.value.slice(start, start + pageSize.value)
})

const totalPages = computed(() =>
  Math.max(1, Math.ceil(filteredLogs.value.length / pageSize.value))
)

/* ---------- FILTER MANAGEMENT ---------- */
const clearColumnFilter = (key) => {
  if (columnFilters.value[key] !== undefined) {
    columnFilters.value[key] = ''
  }
}

const clearAllColumnFilters = () => {
  Object.keys(columnFilters.value).forEach(key => {
    columnFilters.value[key] = ''
  })
}

/* ---------- FORMATTERS & HELPERS ---------- */
const formatDate = (d) => {
  if (!d) return '—'
  try {
    return new Date(d).toLocaleDateString()
  } catch {
    return d
  }
}

const formatTime = (d) => {
  if (!d) return ''
  try {
    return new Date(d).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
  } catch {
    return ''
  }
}

const formatFullDateTime = (d) => {
  if (!d) return '—'
  try {
    return new Date(d).toLocaleString()
  } catch {
    return d
  }
}

const truncateText = (text, length = 25) => {
  if (!text) return '—'
  if (typeof text !== 'string') return String(text)
  return text.length > length ? text.substring(0, length) + '…' : text
}

const getInitials = (name) => {
  if (!name) return '??'
  return name.split(' ').map(w => w[0]).join('').toUpperCase().substring(0, 2)
}

/* ---------- FIELD TYPE DETECTION ---------- */
const isDateField = (key) => {
  const datePatterns = ['date', 'time', 'created', 'modified', 'updated', 'timestamp']
  return datePatterns.some(pattern => key.toLowerCase().includes(pattern))
}

const isIdField = (key) => {
  return key.toLowerCase().includes('id') || 
         key.toLowerCase().includes('guid') || 
         key.toLowerCase().includes('uuid')
}

const isEmailField = (key) => {
  return key.toLowerCase().includes('mail') || 
         key.toLowerCase().includes('principal') || 
         key.toLowerCase().includes('email')
}

const isStatusField = (key) => {
  return key.toLowerCase().includes('status') || 
         key.toLowerCase().includes('risk') || 
         key.toLowerCase().includes('level')
}

/* ---------- STATUS/RISK COLORS ---------- */
const getStatusColor = (status) => {
  if (!status) return 'grey-lighten-3'
  status = status.toLowerCase()
  if (status.includes('success')) return 'green-lighten-4'
  if (status.includes('failed')) return 'red-lighten-4'
  if (status.includes('interrupted') || status.includes('warning')) return 'orange-lighten-4'
  return 'grey-lighten-3'
}

const getStatusTextColor = (status) => {
  if (!status) return 'text-grey-darken-3'
  status = status.toLowerCase()
  if (status.includes('success')) return 'text-green-darken-3'
  if (status.includes('failed')) return 'text-red-darken-3'
  if (status.includes('interrupted') || status.includes('warning')) return 'text-orange-darken-3'
  return 'text-grey-darken-3'
}

const getRiskColor = (risk) => {
  if (!risk) return 'grey-lighten-3'
  risk = risk.toLowerCase()
  if (risk.includes('high')) return 'red-lighten-4'
  if (risk.includes('medium')) return 'orange-lighten-4'
  if (risk.includes('low')) return 'green-lighten-4'
  if (risk.includes('none')) return 'blue-lighten-4'
  return 'grey-lighten-3'
}

const getRiskTextColor = (risk) => {
  if (!risk) return 'text-grey-darken-3'
  risk = risk.toLowerCase()
  if (risk.includes('high')) return 'text-red-darken-3'
  if (risk.includes('medium')) return 'text-orange-darken-3'
  if (risk.includes('low')) return 'text-green-darken-3'
  if (risk.includes('none')) return 'text-blue-darken-3'
  return 'text-grey-darken-3'
}

/* ---------- WATCHERS ---------- */
watch(pageSize, () => (page.value = 1))

/* ---------- LIFECYCLE ---------- */
onMounted(fetchLogs)
</script>

<style scoped>
/* ELEGANT HEADER STYLES */
.elegant-header-cell {
  padding: 14px 12px !important;
  background-color: #ffffff !important;
  border-bottom: 1px solid #eaeef2 !important;
  font-weight: 500 !important;
  transition: background-color 0.2s ease;
}

.elegant-header-cell:hover {
  background-color: #fff8e7 !important;
}

.elegant-header-text {
  font-size: 12px;
  font-weight: 600;
  letter-spacing: 0.3px;
  text-transform: uppercase;
  color: #5f6b7a;
  margin-right: 6px;
}

.filter-btn {
  opacity: 0.6;
  transition: opacity 0.2s ease, color 0.2s ease;
}

.filter-btn:hover {
  opacity: 1;
  background-color: #fff2e0 !important;
}

/* ELEGANT CELL STYLES */
.elegant-cell {
  padding: 8px 12px !important;
  border-bottom: 1px solid #f0f3f7 !important;
  transition: background-color 0.2s ease;
}

.hover-row:hover .elegant-cell {
  background-color: #fef8e7 !important;
}

/* CUSTOM SCROLLBAR */
:deep(::-webkit-scrollbar) {
  width: 8px;
  height: 8px;
}

:deep(::-webkit-scrollbar-track) {
  background: #f5f7fa;
  border-radius: 4px;
}

:deep(::-webkit-scrollbar-thumb) {
  background: #ffffff;
  border-radius: 4px;
  transition: background 0.2s ease;
}

:deep(::-webkit-scrollbar-thumb:hover) {
  background: #ffffff;
}

/* TABLE BASE STYLES */
:deep(.elegant-table .v-data-table-header) {
  background-color: #ffffff !important;
}

:deep(.elegant-table .v-data-table__th) {
  border-bottom: 1px solid #eaeef2 !important;
}

:deep(.elegant-table .v-data-table__td) {
  border-bottom: 1px solid #f0f3f7 !important;
}

.font-family-monospace {
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
}

.d-flex.flex-wrap {
  display: flex;
  flex-wrap: wrap;
}

.gap-2 {
  gap: 8px;
}

.border-t {
  border-top: 1px solid #eaeef2;
}

:deep(.v-chip) {
  color: inherit !important;
}

:deep(.v-data-table-server__wrapper) {
  overflow-x: auto;
}

/* ELEGANT TOOLTIPS */
:deep(.v-tooltip .v-overlay__content) {
  background-color: #1e293b !important;
  color: white !important;
  font-size: 11px !important;
  padding: 6px 12px !important;
  border-radius: 6px !important;
  font-weight: 400 !important;
  letter-spacing: 0.2px !important;
}
</style>