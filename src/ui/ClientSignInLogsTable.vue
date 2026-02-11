<template>
  <v-container fluid class="pa-4">
    <v-card class="elevation-1" rounded="lg">
      <!-- HEADER -->
      <v-card-title class="pa-5 bg-grey-lighten-5">
        <div class="d-flex align-center justify-space-between w-100">
          <div class="d-flex align-center">
            <v-icon icon="mdi-shield-account" color="primary" class="mr-3" />
            <div>
              <h3 class="text-h6 font-weight-medium mb-1">Sign-In Audit Logs</h3>
              <div class="text-caption text-grey-darken-1">
                Client-side pagination · Azure API
              </div>
            </div>
          </div>
          <div class="d-flex align-center">
            <v-chip
              color="warning"
              variant="tonal"
              size="small"
              prepend-icon="mdi-desktop-classic"
              class="mr-3"
            >
              Client-side · Capped @ 500
            </v-chip>
            <v-btn
              variant="outlined"
              prepend-icon="mdi-refresh"
              @click="fetchLogs"
              :loading="loading"
              density="comfortable"
            >
              Refresh
            </v-btn>
          </div>
        </div>
      </v-card-title>

      <!-- ACTIVE FILTERS STATUS BAR -->
      <v-card-text class="pa-4" v-if="hasActiveColumnFilters">
        <div class="d-flex align-center">
          <v-icon icon="mdi-filter-variant" size="18" color="primary" class="mr-2" />
          <span class="text-caption text-grey-darken-1 mr-3">Active column filters:</span>
          <div class="d-flex flex-wrap gap-2">
            <v-chip
              v-for="filter in activeColumnFilters"
              :key="filter.key"
              size="small"
              density="comfortable"
              color="blue-lighten-4"
              closable
              @click:close="clearColumnFilter(filter.key)"
              class="text-grey-darken-3"
            >
              <span class="font-weight-medium mr-1">{{ 
                headers.find(h => h.key === filter.key)?.title 
              }}:</span>
              {{ filter.value }}
            </v-chip>
            <v-btn
              size="x-small"
              variant="text"
              @click="clearAllColumnFilters"
              class="ml-2"
            >
              Clear all columns
            </v-btn>
          </div>
        </div>
      </v-card-text>

      <v-divider />

      <!-- TABLE -->
      <v-data-table-server
        :headers="headers"
        :items="pagedLogs"
        :items-length="filteredLogs.length"
        :loading="loading"
        :page="page"
        :items-per-page="pageSize"
        @update:page="page = $event"
        @update:items-per-page="pageSize = $event"
        class="elevation-0"
        :loading-text="'Loading audit logs...'"
        :no-data-text="'No sign-in logs found'"
      >

        <!-- CUSTOM HEADERS WITH FILTER ICONS -->
        <template #headers="{ columns }">
          <tr class="bg-grey-lighten-4">
            <th v-for="col in columns" :key="col.key" class="pa-3">
              <div class="d-flex align-center justify-space-between">
                <span class="text-uppercase text-caption font-weight-medium text-grey-darken-2">
                  {{ col.title }}
                </span>
                
                <!-- FILTER ICON WITH MENU -->
                <v-menu
                  :close-on-content-click="false"
                  location="bottom"
                >
                  <template #activator="{ props }">
                    <v-btn
                      icon
                      size="x-small"
                      variant="text"
                      v-bind="props"
                      :color="columnFilters[col.key] ? 'primary' : 'grey-darken-1'"
                      density="compact"
                      class="ml-1"
                    >
                      <v-icon size="18">
                        {{ columnFilters[col.key] ? 'mdi-filter' : 'mdi-filter-outline' }}
                      </v-icon>
                    </v-btn>
                  </template>

                  <v-card class="pa-3" width="260">
                    <div class="text-caption font-weight-medium mb-2">
                      Filter by {{ col.title }}
                    </div>
                    <v-text-field
                      v-model="columnFilters[col.key]"
                      density="compact"
                      variant="outlined"
                      hide-details
                      :placeholder="`Search ${col.title.toLowerCase()}...`"
                      :prepend-inner-icon="columnFilters[col.key] ? 'mdi-filter-check' : 'mdi-filter'"
                      clearable
                      @click:clear="clearColumnFilter(col.key)"
                    />
                    <div class="text-caption text-grey-darken-1 mt-1">
                      <v-icon icon="mdi-information-outline" size="14" class="mr-1" />
                      Client-side filtering (instant)
                    </div>
                  </v-card>
                </v-menu>
              </div>
            </th>
          </tr>
        </template>

        <!-- CUSTOM ROW TEMPLATE -->
        <template #item="{ item }">
          <tr class="hover-row">
            <!-- Time -->
            <td class="pa-3">
              <div class="text-caption font-weight-medium text-grey-darken-3">
                {{ formatDate(item.createdDateTime) }}
              </div>
              <div class="text-caption text-grey-darken-1">
                {{ formatTime(item.createdDateTime) }}
              </div>
            </td>

            <!-- User -->
            <td class="pa-3">
              <div class="d-flex align-center">
                <v-avatar size="28" color="blue-lighten-5" class="mr-3">
                  <span class="text-caption font-weight-medium text-blue-darken-2">
                    {{ getInitials(item.userDisplayName) }}
                  </span>
                </v-avatar>
                <div class="text-body-2 font-weight-medium text-grey-darken-3">
                  {{ item.userDisplayName || '—' }}
                </div>
              </div>
            </td>

            <!-- UPN -->
            <td class="pa-3">
              <div class="text-body-2 text-truncate text-grey-darken-3" style="max-width: 180px">
                {{ item.userPrincipalName || '—' }}
              </div>
            </td>

            <!-- Application -->
            <td class="pa-3">
              <v-chip
                size="small"
                density="comfortable"
                variant="tonal"
                color="indigo"
                label
                class="text-grey-darken-3"
              >
                {{ item.appDisplayName || '—' }}
              </v-chip>
            </td>

            <!-- Status -->
            <td class="pa-3">
              <v-chip
                size="small"
                density="comfortable"
                :color="getStatusColor(item.status)"
                :class="getStatusTextColor(item.status)"
                label
                class="font-weight-medium"
              >
                {{ item.status || '—' }}
              </v-chip>
            </td>

            <!-- IP Address -->
            <td class="pa-3">
              <div class="text-caption font-family-monospace text-grey-darken-3">
                {{ item.ipaddress || '—' }}
              </div>
            </td>

            <!-- Location -->
            <td class="pa-3">
              <div class="text-body-2 text-grey-darken-3">
                {{ item.location || '—' }}
              </div>
            </td>

            <!-- Client -->
            <td class="pa-3">
              <div class="text-body-2 text-grey-darken-3">
                {{ item.clientAppUsed || '—' }}
              </div>
            </td>

            <!-- Risk -->
            <td class="pa-3">
              <v-chip
                size="small"
                density="comfortable"
                :color="getRiskColor(item.riskLevelAggregated)"
                :class="getRiskTextColor(item.riskLevelAggregated)"
                label
                class="font-weight-medium"
              >
                {{ item.riskLevelAggregated || '—' }}
              </v-chip>
            </td>
          </tr>
        </template>

        <!-- LOADING STATE -->
        <template #loading>
          <tr>
            <td colspan="9" class="pa-10 text-center">
              <v-progress-circular
                indeterminate
                color="primary"
                size="32"
                width="2"
              />
              <div class="text-caption text-grey-darken-1 mt-4">
                Loading audit logs from Azure API...
              </div>
            </td>
          </tr>
        </template>

        <!-- NO DATA STATE -->
        <template #no-data>
          <tr>
            <td colspan="9" class="pa-10 text-center">
              <v-icon 
                :icon="hasActiveColumnFilters ? 'mdi-filter-remove' : 'mdi-file-search-outline'" 
                size="48" 
                class="text-grey-lighten-1 mb-4" 
              />
              <h4 class="text-h6 font-weight-regular mb-2 text-grey-darken-2">
                {{ hasActiveColumnFilters ? 'No matching records found' : 'No sign-in logs available' }}
              </h4>
              <p class="text-body-2 text-grey-darken-1 mb-4">
                {{ hasActiveColumnFilters ? 'Try adjusting your filters' : 'Data will appear here once available' }}
              </p>
              <v-btn
                v-if="hasActiveColumnFilters"
                variant="outlined"
                @click="clearAllColumnFilters"
                prepend-icon="mdi-filter-remove"
                class="mr-2"
              >
                Clear Filters
              </v-btn>
              <v-btn
                variant="tonal"
                @click="fetchLogs"
                prepend-icon="mdi-refresh"
              >
                Refresh
              </v-btn>
            </td>
          </tr>
        </template>

        <!-- TABLE FOOTER -->
        <template #bottom>
          <div class="text-caption text-grey-darken-1 px-4 py-3 border-t d-flex justify-space-between">
            <div>
              <span v-if="hasActiveColumnFilters" class="mr-3">
                <v-icon icon="mdi-filter-check" size="16" class="mr-1" />
                Client-side filtered
              </span>
              Showing {{ showingFrom }} to {{ showingTo }} of {{ filteredLogs.length }} records
            </div>
            <div>
              Page {{ page }} of {{ totalPages }}
            </div>
          </div>
        </template>

      </v-data-table-server>

      <!-- PAGINATION CONTROLS -->
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
              color="primary"
            />
          </v-col>
          
          <v-col cols="12" md="3" class="text-right">
            <div class="text-caption text-grey-darken-1">
              {{ pagedLogs.length }} rows shown (client-side)
            </div>
          </v-col>
        </v-row>
      </v-card-actions>
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

/* ---------- HEADERS ---------- */
const headers = [
  { title: 'Time', key: 'createdDateTime', width: '180px' },
  { title: 'User', key: 'userDisplayName', width: '180px' },
  { title: 'UPN', key: 'userPrincipalName', width: '250px' },
  { title: 'Application', key: 'appDisplayName', width: '200px' },
  { title: 'Status', key: 'status', width: '120px' },
  { title: 'IP Address', key: 'ipaddress', width: '150px' },
  { title: 'Location', key: 'location', width: '180px' },
  { title: 'Client', key: 'clientAppUsed', width: '180px' },
  { title: 'Risk', key: 'riskLevelAggregated', width: '120px' }
]

/* ---------- COLUMN FILTERS ---------- */
const columnFilters = ref({
  createdDateTime: '',
  userDisplayName: '',
  userPrincipalName: '',
  appDisplayName: '',
  status: '',
  ipaddress: '',
  location: '',
  clientAppUsed: '',
  riskLevelAggregated: ''
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

/* ---------- API ---------- */
const fetchLogs = async () => {
  loading.value = true
  try {
    const res = await axios.get(`${API_BASE}/api/signin-logs`, {
      params: {
        page: 1,
        pageSize: 500,
        fromDate: fromDate.value,
        toDate: toDate.value
      }
    })

    allLogs.value = res.data?.data || []
    page.value = 1
  } catch (err) {
    console.error("API ERROR:", err)
    allLogs.value = []
  } finally {
    loading.value = false
  }
}

/* ---------- CLIENT-SIDE FILTERING ---------- */
const filteredLogs = computed(() => {
  return allLogs.value.filter(log =>
    Object.keys(columnFilters.value).every(key => {
      const filter = columnFilters.value[key]
      if (!filter) return true
      return log[key]?.toString().toLowerCase().includes(filter.toLowerCase())
    })
  )
})

/* ---------- CLIENT-SIDE PAGINATION ---------- */
const pagedLogs = computed(() => {
  const start = (page.value - 1) * pageSize.value
  return filteredLogs.value.slice(start, start + pageSize.value)
})

const totalPages = computed(() =>
  Math.max(1, Math.ceil(filteredLogs.value.length / pageSize.value))
)

/* ---------- FILTER MANAGEMENT ---------- */
const clearColumnFilter = (key) => {
  columnFilters.value[key] = ''
}

const clearAllColumnFilters = () => {
  Object.keys(columnFilters.value).forEach(key => {
    columnFilters.value[key] = ''
  })
}

/* ---------- FORMATTERS ---------- */
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

const getInitials = (name) => {
  if (!name) return '??'
  return name.split(' ').map(w => w[0]).join('').toUpperCase().substring(0, 2)
}

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
.hover-row:hover {
  background-color: #f8f9fa;
}

.font-family-monospace {
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
}

:deep(.v-data-table-header) {
  background-color: #f5f7fa;
}

:deep(.v-data-table__th) {
  border-bottom: 2px solid #e0e0e0;
  font-weight: 500;
}

.d-flex.flex-wrap {
  display: flex;
  flex-wrap: wrap;
}

.gap-2 {
  gap: 8px;
}

.border-t {
  border-top: 1px solid #e0e0e0;
}

/* Ensure text is visible in chips */
:deep(.v-chip) {
  color: inherit !important;
}
</style>