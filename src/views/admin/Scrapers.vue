<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, computed, watch, nextTick } from 'vue'
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table'
import { Alert, AlertTitle, AlertDescription } from '@/components/ui/alert'
import UiSwitch from '@/components/ui/switch/index.vue'

// Raw logs types
interface LogFile {
  scraper: string
  filename: string
  modified: number
}

interface LogEntry {
  time: string
  level: string
  name: string
  msg: string
  url: string
  operation: string
}

// Raw logs state
const rawLogScrapers = ref<LogFile[]>([])
const selectedLogScraper = ref('')
const rawLogs = ref<LogEntry[]>([])
const logTimer = ref<number | null>(null)

// Add source filter state
const selectedSource = ref('')
import {
  Activity,
  ClipboardList,
  ChevronLeft,
  ChevronRight,
  Timer,
  Cpu,
  HardDrive,
  Network,
  AlertTriangle,
  CheckCircle,
  RefreshCw,
  RotateCcw,
  AlertOctagon
} from 'lucide-vue-next'
import axios from '@/lib/axios'
import { isAxiosError } from 'axios'

interface ScraperStatus {
  total_posts: number
  posts_by_sentiment: Record<string, number>
  posts_by_source: Record<string, number>
  recent_logs: Array<{
    id: number
    keyword_term: string
    source: string
    status: string
    started_at: string
    completed_at: string | null
    posts_found: number
    error_message: string | null
    is_running: boolean
  }>
}

// Format memory size
const formatMemory = (mb: number) => {
  if (!mb) return '0 MB'
  return `${mb.toFixed(1)} MB`
}

// Format percentage
const formatPercent = (value: number) => {
  if (!value) return '0%'
  return `${value.toFixed(1)}%`
}

// Format network speed
const formatSpeed = (kbps: number) => {
  if (!kbps) return '0 KB/s'
  if (kbps < 1024) return `${kbps.toFixed(1)} KB/s`
  return `${(kbps / 1024).toFixed(1)} MB/s`  // Convert to MB/s if over 1024 KB/s
}

// Format bytes
const formatBytes = (bytes: number) => {
  if (!bytes) return '0 KB'
  const kb = bytes / 1024
  if (kb < 1024) return `${kb.toFixed(1)} KB`
  const mb = kb / 1024
  return `${mb.toFixed(1)} MB`
}

// Format CPU time
const formatTime = (ms: number) => {
  if (!ms) return '0ms'
  if (ms < 1000) return `${ms.toFixed(2)}ms`
  return `${(ms / 1000).toFixed(1)}s`
}

// Import store
import { useScraperScheduleStore } from '@/stores/scraperSchedule'

interface ScraperHealth {
  source: string
  total_tasks: number
  failed_tasks: number
  success_rate: number
  needs_attention: boolean
  next_run?: string | null
  is_enabled: boolean
}

// Initialize with default values
const defaultScraperHealth: Partial<ScraperHealth> = {
  is_enabled: true,
  total_tasks: 0,
  failed_tasks: 0,
  success_rate: 0,
  needs_attention: false
}

const scraperHealth = ref<ScraperHealth[]>([])
const scraperScheduleStore = useScraperScheduleStore()

const stats = ref<ScraperStatus & {
  // Performance metrics
  avgDuration: number
  activeInstances: number
  successRate: number
  // Memory metrics
  peakMemory: number
  instanceMemory: number
  availableMemory: number
  totalMemory: number
  // CPU metrics
  systemCpu: number
  scraperCpu: number
  browserCpu: number
  cpuTime: number
  // Network metrics
  bytesSent: number
  bytesReceived: number
  requestCount: number
  throughput: number
}>({
  total_posts: 0,
  posts_by_sentiment: {},
  posts_by_source: {},
  recent_logs: [],
  // Initialize new metrics
  avgDuration: 0,
  activeInstances: 0,
  successRate: 0,
  peakMemory: 0,
  instanceMemory: 0,
  availableMemory: 0,
  totalMemory: 0,
  systemCpu: 0,
  scraperCpu: 0,
  browserCpu: 0,
  cpuTime: 0,
  bytesSent: 0,
  bytesReceived: 0,
  requestCount: 0,
  throughput: 0
})

const loading = ref(false)
const error = ref<string | null>(null)

// Pagination
const currentPage = ref(1)
const itemsPerPage = 10
const allLogs = ref<ScraperStatus['recent_logs']>([])
const totalItems = computed(() => allLogs.value.length)
const totalPages = computed(() => Math.ceil(totalItems.value / itemsPerPage))

// Watch for page changes
watch(currentPage, () => {
  // Update paginated logs when page changes
  stats.value = {
    ...stats.value,
    recent_logs: allLogs.value.slice((currentPage.value - 1) * itemsPerPage, currentPage.value * itemsPerPage)
  }
})

// Fetch scraper health stats
const fetchScraperHealth = async () => {
  try {
    // Fetch schedules first
    await scraperScheduleStore.fetchSchedules()
    
    // Then fetch health stats
    const response = await axios.get('/logs/scraper-health')
    
    // Map health stats with schedule data
    scraperHealth.value = response.data.stats.map((stat: ScraperHealth) => ({
      ...defaultScraperHealth,
      ...stat,
      next_run: scraperScheduleStore.getNextRun(stat.source)
    }))
  } catch (err) {
    console.error('Error fetching scraper health:', err)
    if (isAxiosError(err)) {
      error.value = err.response?.data?.error || 'Failed to load scraper health statistics'
    } else {
      error.value = 'Failed to load scraper health statistics'
    }
  }
}

// Fetch scraper status and stats
/**
 * Fetch scraper stats and logs
 */
const fetchStats = async () => {
  try {
    loading.value = true
    error.value = null

    // Add source to logs query if selected
    const logsUrl = selectedSource.value
      ? `/logs/?page=${currentPage.value}&ordering=-started_at&page_size=${itemsPerPage}&source=${selectedSource.value}`
      : `/logs/?page=${currentPage.value}&ordering=-started_at&page_size=${itemsPerPage}`

    // Fetch stats and logs in parallel
    const [statsResponse, logsResponse] = await Promise.all([
      axios.get('/logs/status/'),
      axios.get(logsUrl)
    ])
    
    // Store all logs
    allLogs.value = logsResponse.data
    
    // Update stats with paginated logs
    stats.value = {
      ...statsResponse.data,
      recent_logs: allLogs.value.slice(
        (currentPage.value - 1) * itemsPerPage,
        currentPage.value * itemsPerPage
      )
    }
  } catch (err) {
    console.error('Error fetching scraper stats:', err)
    if (isAxiosError(err)) {
      error.value = err.response?.data?.error || 'Failed to load scraper statistics'
    } else {
      error.value = 'Failed to load scraper statistics'
    }
  } finally {
    loading.value = false
  }
}

// Types
interface ResetTasksResponse {
  status: 'success'
  message: string
  tasks_updated: number
}

// Reset failed tasks
const resetFailedTasks = async (): Promise<void> => {
  try {
    loading.value = true;
    error.value = null;
    
    const { data } = await axios.post<ResetTasksResponse>('/logs/reset-failed/');
    
    if (data.status === 'success') {
      // Show success message
      error.value = `Successfully ${data.message}`;
      
      // Log success for monitoring
      console.info(`Reset tasks operation completed: ${data.tasks_updated} tasks updated`);
      
      // Refresh data
      await Promise.all([
        fetchStats(),
        fetchScraperHealth()
      ]);
      
      // Clear success message after 3 seconds
      setTimeout(() => {
        if (error.value?.startsWith('Successfully')) {
          error.value = null;
        }
      }, 3000);
    }
  } catch (err) {
    console.error('Error resetting failed tasks:', err);
    if (isAxiosError(err)) {
      error.value = err.response?.data?.error || 'Failed to reset tasks'
    } else {
      error.value = 'Failed to reset tasks'
    }
  } finally {
    loading.value = false;
  }
};

/**
 * Refresh all scraper data
 * Fetches health stats first to ensure enabled states are set,
 * then fetches performance stats and logs
 */
const refreshAll = async () => {
  if (loading.value) return

  try {
    loading.value = true
    error.value = null

    // Fetch data in sequence (health must be first)
    await fetchScraperHealth()
    await fetchStats()
    await fetchRawLogs()
  } catch (err) {
    console.error('Error refreshing data:', err)
    if (isAxiosError(err)) {
      error.value = err.response?.data?.error || 'Failed to refresh data'
    } else {
      error.value = 'Failed to refresh data'
    }
  } finally {
    loading.value = false
  }
}

// Format timestamp
const formatDate = (date: string) => {
  // Create date in UTC
  const d = new Date(date)
  // Use toLocaleString with Manila timezone
  return d.toLocaleString('en-US', {
    timeZone: 'Asia/Manila',
    month: 'short',
    day: 'numeric',
    hour: 'numeric',
    minute: '2-digit',
    hour12: true,
    formatMatcher: 'basic'
  }).replace(',', ' -')
}

// Get source badge color
const getSourceColor = (source: string) => {
  switch (source) {
    // Green shades
    case 'rollitup.org':
      return 'bg-emerald-100 text-emerald-800'
    case 'overgrow.com':
      return 'bg-green-100 text-green-800'
    case 'growweedeasy.com':
      return 'bg-lime-100 text-lime-800'
      
    // Blue shades
    case 'icmag.com':
      return 'bg-blue-100 text-blue-800'
    case 'uk420.com':
      return 'bg-sky-100 text-sky-800'
    case 'beanbasement.nl':
      return 'bg-cyan-100 text-cyan-800'
      
    // Purple/Pink shades
    case 'thcfarmer.com':
      return 'bg-purple-100 text-purple-800'
    case 'phenohunter.com':
      return 'bg-fuchsia-100 text-fuchsia-800'
    case 'percysgrowroom.com':
      return 'bg-pink-100 text-pink-800'
      
    // Orange/Red shades
    case '420magazine.com':
      return 'bg-orange-100 text-orange-800'
    case 'reddit.com':
      return 'bg-red-100 text-red-800'
    case 'marijuanapassion.com':
      return 'bg-rose-100 text-rose-800'
      
    // Yellow/Brown shades
    case 'autoflower.org':
      return 'bg-yellow-100 text-yellow-800'
    case 'growersnetwork.org':
      return 'bg-amber-100 text-amber-800'
    case '420sa.co.za':
      return 'bg-orange-100 text-orange-800'
      
    // Unique shades
    case 'ilgmforum.com':
      return 'bg-indigo-100 text-indigo-800'
    case 'homegrowncannabisco.community':
      return 'bg-teal-100 text-teal-800'
      
    default:
      return 'bg-gray-100 text-gray-800'
  }
}

// Get status badge color
const getStatusColor = (status: string) => {
  const colors = {
    completed: 'bg-green-100 text-green-800',
    failed: 'bg-red-100 text-red-800',
    started: 'bg-blue-100 text-blue-800'
  }
  return colors[status as keyof typeof colors] || 'bg-gray-100 text-gray-800'
}

// Track which scrapers are being toggled
const togglingScrapers = ref<Record<string, boolean>>({})
const emergencyStopActive = ref(false)

// Toggle emergency stop
const toggleEmergencyStop = async () => {
  if (loading.value) return
  
  try {
    loading.value = true
    const response = await axios.post('/scraper-control/emergency-stop/')
    emergencyStopActive.value = response.data.emergency_stop
  } catch (err) {
    if (isAxiosError(err)) {
      error.value = err.response?.data?.error || 'Failed to toggle emergency stop'
    } else {
      error.value = 'Failed to toggle emergency stop'
    }
  } finally {
    loading.value = false
  }
}

/**
 * Toggle a scraper's enabled state
 * @param source The scraper source (e.g., 'thcfarmer.com')
 */
const toggleScraper = async (source: string) => {
  // Prevent multiple toggles
  if (togglingScrapers.value[source]) return

  try {
    // Set loading state
    togglingScrapers.value[source] = true

    // Extract base name (e.g., 'thcfarmer.com' -> 'thcfarmer')
    const base_source = source.split('.')[0]
    
    // Find scraper to update
    const scraper = scraperHealth.value.find(s => s.source === source)
    if (!scraper) {
      throw new Error(`Scraper ${source} not found`)
    }
    
    // Toggle state via API
    const response = await axios.post<{ status: string; enabled: boolean }>(
      `/scraper-control/${base_source}/toggle/`
    )
    
    // Update local state with confirmed state from backend
    scraper.is_enabled = response.data.enabled
  } catch (err) {
    // Handle errors
    if (isAxiosError(err)) {
      error.value = err.response?.data?.error || 'Failed to toggle scraper'
    } else {
      error.value = 'Failed to toggle scraper'
    }
  } finally {
    // Clear loading state
    togglingScrapers.value[source] = false
  }
}

// Raw logs methods
async function fetchRawLogs() {
  try {
    const url = selectedLogScraper.value
      ? `/logs/raw-logs/?scraper=${selectedLogScraper.value}`
      : '/logs/raw-logs/'
    
    const { data } = await axios.get(url)
    
    if (selectedLogScraper.value) {
      const wasEmpty = rawLogs.value.length === 0
      rawLogs.value = data
      // Only auto-scroll on initial load
      if (wasEmpty) {
        nextTick(() => {
          const container = document.querySelector('.raw-logs-container')
          if (container) {
            container.scrollTop = container.scrollHeight
          }
        })
      }
    } else {
      rawLogScrapers.value = data
    }
  } catch (error) {
    console.error('Failed to fetch raw logs:', error)
  }
}

// Start polling when scraper selected
watch(selectedLogScraper, (scraper) => {
  if (logTimer.value) {
    window.clearInterval(logTimer.value)
    logTimer.value = null
  }
  
  rawLogs.value = []
  
  if (scraper) {
    fetchRawLogs()
    // Poll every 30 seconds for new logs
    logTimer.value = window.setInterval(fetchRawLogs, 30000)
  }
})

// Cleanup on unmount
onBeforeUnmount(() => {
  if (logTimer.value) {
    window.clearInterval(logTimer.value)
  }
})

// Initial data fetch
onMounted(async () => {
  console.log('Scrapers component mounted')
  try {
    // Fetch scraper health first to ensure enabled states are set
    await fetchScraperHealth()
    await fetchStats()
    // Initial raw logs fetch
    await fetchRawLogs()
  } catch (err) {
    console.error('Error during initial data fetch:', err)
  }
})
</script>

<template>
  <div class="flex-1 p-6">
    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8">
      <!-- Welcome Section -->
      <div class="mb-8 bg-white rounded-lg shadow-sm p-6">
        <div class="flex flex-col md:flex-row items-center gap-6">
          <div class="flex-1">
            <h1 class="text-2xl md:text-3xl font-bold text-green-800 mb-2">
              Scraper Monitoring
            </h1>
            <p class="text-gray-600 text-lg">
              Monitor automated data collection processes. Track scraping performance, view logs, and analyze collected data metrics.
            </p>
          </div>
          <div class="flex gap-4">
            <button
              @click="refreshAll"
              class="inline-flex items-center gap-2 px-4 py-2 bg-green-600 text-white rounded-md hover:bg-green-700 transition-colors"
            >
              <RefreshCw class="w-5 h-5" />
              Refresh All Stats
            </button>
            <button
              @click="resetFailedTasks"
              class="inline-flex items-center gap-2 px-4 py-2 bg-yellow-600 text-white rounded-md hover:bg-yellow-700 transition-colors"
              :disabled="loading"
            >
              <RotateCcw class="w-5 h-5" />
              Reset Failed Tasks
            </button>
            <button
              @click="toggleEmergencyStop"
              class="inline-flex items-center gap-2 px-4 py-2 rounded-md transition-colors"
              :class="[
                emergencyStopActive
                  ? 'bg-gray-600 hover:bg-gray-700 text-white'
                  : 'bg-red-600 hover:bg-red-700 text-white'
              ]"
              :disabled="loading"
            >
              <AlertOctagon class="w-5 h-5" />
              {{ emergencyStopActive ? 'Clear Emergency Stop' : 'Emergency Stop' }}
            </button>
          </div>
        </div>
      </div>

      <!-- Error Alert -->
      <Alert v-if="error" variant="destructive" class="mb-8">
        <AlertTitle>Error</AlertTitle>
        <AlertDescription>{{ error }}</AlertDescription>
      </Alert>

      <!-- Resource Monitoring Cards -->
      <div class="grid grid-cols-1 md:grid-cols-4 gap-6 mb-8">
        <template v-if="loading">
          <div v-for="i in 4" :key="i" class="bg-white rounded-lg shadow-sm p-6">
            <div class="flex items-center gap-2 mb-4">
              <div class="bg-emerald-50 p-2 rounded-lg">
                <div class="h-5 w-5 bg-gray-200 rounded animate-pulse"></div>
              </div>
              <div class="h-5 bg-gray-200 rounded animate-pulse w-32"></div>
            </div>
            <div class="space-y-2">
              <div class="flex justify-between items-center">
                <div class="h-4 bg-gray-200 rounded animate-pulse w-24"></div>
                <div class="h-4 bg-gray-200 rounded animate-pulse w-16"></div>
              </div>
              <div class="flex justify-between items-center">
                <div class="h-4 bg-gray-200 rounded animate-pulse w-20"></div>
                <div class="h-4 bg-gray-200 rounded animate-pulse w-12"></div>
              </div>
              <div class="flex justify-between items-center">
                <div class="h-4 bg-gray-200 rounded animate-pulse w-28"></div>
                <div class="h-4 bg-gray-200 rounded animate-pulse w-14"></div>
              </div>
              <div class="flex justify-between items-center">
                <div class="h-4 bg-gray-200 rounded animate-pulse w-24"></div>
                <div class="h-4 bg-gray-200 rounded animate-pulse w-16"></div>
              </div>
            </div>
          </div>
        </template>
        <template v-else>
          <!-- Performance Card -->
          <div class="bg-white rounded-lg shadow-sm p-6">
            <div class="flex items-center gap-2 mb-4">
              <div class="bg-emerald-50 p-2 rounded-lg">
                <Timer class="w-5 h-5 text-emerald-500" />
              </div>
              <h3 class="text-lg font-semibold text-green-800">Performance Metrics</h3>
            </div>
            <div class="space-y-2">
              <div class="flex justify-between items-center">
                <span>Avg. Duration</span>
                <span class="font-semibold">{{ formatTime(stats.avgDuration) }}</span>
              </div>
              <div class="flex justify-between items-center">
                <span>Active Instances</span>
                <span class="font-semibold">{{ stats.activeInstances }}</span>
              </div>
              <div class="flex justify-between items-center">
                <span>Success Rate</span>
                <span class="font-semibold">{{ formatPercent(stats.successRate) }}</span>
              </div>
            </div>
          </div>

          <!-- Memory Usage Card -->
          <div class="bg-white rounded-lg shadow-sm p-6">
            <div class="flex items-center gap-2 mb-4">
              <div class="bg-emerald-50 p-2 rounded-lg">
                <HardDrive class="w-5 h-5 text-emerald-500" />
              </div>
              <h3 class="text-lg font-semibold text-green-800">Memory Usage</h3>
            </div>
            <div class="space-y-2">
              <div class="flex justify-between items-center">
                <span>Peak Memory</span>
                <span class="font-semibold">{{ stats.activeInstances > 0 ? formatMemory(stats.peakMemory) : '0 MB' }}</span>
              </div>
              <div class="flex justify-between items-center">
                <span>Per Instance</span>
                <span class="font-semibold">{{ stats.activeInstances > 0 ? formatMemory(stats.instanceMemory) : '0 MB' }}</span>
              </div>
              <div class="flex justify-between items-center">
                <span>Available</span>
                <span class="font-semibold">{{ formatMemory(stats.availableMemory) }}</span>
              </div>
              <div class="flex justify-between items-center">
                <span>Total Used</span>
                <span class="font-semibold">{{ formatMemory(stats.totalMemory) }}</span>
              </div>
            </div>
          </div>

          <!-- CPU Usage Card -->
          <div class="bg-white rounded-lg shadow-sm p-6">
            <div class="flex items-center gap-2 mb-4">
              <div class="bg-emerald-50 p-2 rounded-lg">
                <Cpu class="w-5 h-5 text-emerald-500" />
              </div>
              <h3 class="text-lg font-semibold text-green-800">CPU Usage</h3>
            </div>
            <div class="space-y-2">
              <div class="flex justify-between items-center">
                <span>System CPU</span>
                <span class="font-semibold">{{ formatPercent(stats.systemCpu) }}</span>
              </div>
              <div class="flex justify-between items-center">
                <span>Scraper CPU</span>
                <span class="font-semibold">{{ stats.activeInstances > 0 ? formatPercent(stats.scraperCpu) : '0%' }}</span>
              </div>
              <div class="flex justify-between items-center">
                <span>Browser CPU</span>
                <span class="font-semibold">{{ stats.activeInstances > 0 ? formatPercent(stats.browserCpu) : '0%' }}</span>
              </div>
              <div class="flex justify-between items-center">
                <span>CPU Time</span>
                <span class="font-semibold">{{ stats.activeInstances > 0 ? formatTime(stats.cpuTime) : '0ms' }}</span>
              </div>
            </div>
          </div>

          <!-- Network Usage Card -->
          <div class="bg-white rounded-lg shadow-sm p-6">
            <div class="flex items-center gap-2 mb-4">
              <div class="bg-emerald-50 p-2 rounded-lg">
                <Network class="w-5 h-5 text-emerald-500" />
              </div>
              <h3 class="text-lg font-semibold text-green-800">Network Statistics</h3>
            </div>
            <div class="space-y-2">
              <div class="flex justify-between items-center">
                <span>Total Data Sent</span>
                <span class="font-semibold">{{ formatBytes(stats.bytesSent) }}</span>
              </div>
              <div class="flex justify-between items-center">
                <span>Total Data Received</span>
                <span class="font-semibold">{{ formatBytes(stats.bytesReceived) }}</span>
              </div>
              <div class="flex justify-between items-center">
                <span>Total Packets</span>
                <span class="font-semibold">{{ stats.requestCount }}</span>
              </div>
              <div class="flex justify-between items-center">
                <span>Current Speed</span>
                <span class="font-semibold">{{ formatSpeed(stats.throughput) }}</span>
              </div>
            </div>
          </div>
        </template>
      </div>

      <!-- Scraper Health Stats -->
      <div v-if="loading" class="bg-white rounded-lg shadow-sm overflow-hidden mb-8">
        <!-- Table Header -->
        <div class="p-4 border-b border-gray-200">
          <div class="flex items-center gap-4">
            <div class="bg-emerald-50 p-2 rounded-lg">
              <Activity class="w-5 h-5 text-emerald-500" />
            </div>
            <div>
              <h2 class="text-xl font-semibold text-gray-800">Scraper Health (Last 24 Hours)</h2>
              <p class="text-sm text-gray-500 mt-1">Monitor scraper performance and scheduling status</p>
            </div>
          </div>
        </div>
        
        <div class="overflow-x-auto">
          <Table class="text-base border-collapse w-full">
            <TableHeader>
              <TableRow>
                <TableHead class="text-base border-b border-gray-200 px-4 py-2 bg-gray-50">Source</TableHead>
                <TableHead class="text-base border-b border-gray-200 px-4 py-2 bg-gray-50">Total Tasks</TableHead>
                <TableHead class="text-base border-b border-gray-200 px-4 py-2 bg-gray-50">Failed Tasks</TableHead>
                <TableHead class="text-base border-b border-gray-200 px-4 py-2 bg-gray-50">Success Rate</TableHead>
                <TableHead class="text-base border-b border-gray-200 px-4 py-2 bg-gray-50">Next Run</TableHead>
                <TableHead class="text-base border-b border-gray-200 px-4 py-2 bg-gray-50">Status</TableHead>
              </TableRow>
            </TableHeader>
            <TableBody>
              <TableRow v-for="i in 5" :key="i">
                <TableCell class="text-base border-b border-gray-100 px-4">
                  <div class="h-6 bg-gray-200 rounded animate-pulse w-32"></div>
                </TableCell>
                <TableCell class="text-base border-b border-gray-100 px-4">
                  <div class="h-6 bg-gray-200 rounded animate-pulse w-16"></div>
                </TableCell>
                <TableCell class="text-base border-b border-gray-100 px-4">
                  <div class="h-6 bg-gray-200 rounded animate-pulse w-16"></div>
                </TableCell>
                <TableCell class="text-base border-b border-gray-100 px-4">
                  <div class="h-6 bg-gray-200 rounded animate-pulse w-16"></div>
                </TableCell>
                <TableCell class="text-base border-b border-gray-100 px-4">
                  <div class="h-6 bg-gray-200 rounded animate-pulse w-32"></div>
                </TableCell>
                <TableCell class="text-base border-b border-gray-100 px-4">
                  <div class="h-6 bg-gray-200 rounded animate-pulse w-24"></div>
                </TableCell>
              </TableRow>
            </TableBody>
          </Table>
        </div>
      </div>

      <!-- Scraper Health Stats Content -->
      <div v-else class="bg-white rounded-lg shadow-sm overflow-hidden mb-8">
        <!-- Table Header -->
        <div class="p-4 border-b border-gray-200">
          <div class="flex items-center gap-4">
            <div class="bg-emerald-50 p-2 rounded-lg">
              <Activity class="w-5 h-5 text-emerald-500" />
            </div>
            <div>
              <h2 class="text-xl font-semibold text-gray-800">Scraper Health (Last 24 Hours)</h2>
              <p class="text-sm text-gray-500 mt-1">Monitor scraper performance and scheduling status</p>
            </div>
          </div>
        </div>
        
        <div class="overflow-x-auto">
          <Table class="text-base border-collapse">
            <TableHeader>
              <TableRow>
                <TableHead class="text-base border-b border-gray-200 px-4 py-2 bg-gray-50">Source</TableHead>
                <TableHead class="text-base border-b border-gray-200 px-4 py-2 bg-gray-50">Total Tasks</TableHead>
                <TableHead class="text-base border-b border-gray-200 px-4 py-2 bg-gray-50">Failed Tasks</TableHead>
                <TableHead class="text-base border-b border-gray-200 px-4 py-2 bg-gray-50">Success Rate</TableHead>
                <TableHead class="text-base border-b border-gray-200 px-4 py-2 bg-gray-50">Next Run</TableHead>
                <TableHead class="text-base border-b border-gray-200 px-4 py-2 bg-gray-50">Status</TableHead>
              </TableRow>
            </TableHeader>
            <TableBody>
              <TableRow v-for="stat in scraperHealth" :key="stat.source" :class="{ 'bg-red-50': stat.needs_attention, 'hover:bg-gray-50': !stat.needs_attention }">
                <TableCell class="text-base border-b border-gray-100 px-4">
                  <span :class="['px-2 py-1 rounded-md', getSourceColor(stat.source)]">
                    {{ stat.source }}
                  </span>
                </TableCell>
                <TableCell class="text-base border-b border-gray-100 px-4">{{ stat.failed_tasks }}</TableCell>
                <TableCell class="text-base border-b border-gray-100 px-4">{{ stat.success_rate }}%</TableCell>
                <TableCell class="text-base border-b border-gray-100 px-4">{{ stat.next_run ? formatDate(stat.next_run) : '-' }}</TableCell>
                <TableCell class="text-base border-b border-gray-100 px-4">
                  <span v-if="stat.needs_attention" class="text-red-600 font-medium">
                    <AlertTriangle class="w-5 h-5" />
                  </span>
                  <span v-else class="text-green-600 font-medium">
                    <CheckCircle class="w-5 h-5" />
                  </span>
                </TableCell>
                <TableCell class="text-base border-b border-gray-100 px-4">
                  <!-- Scraper Toggle Switch -->
                  <UiSwitch
                    :model-value="Boolean(stat.is_enabled)"
                    :disabled="emergencyStopActive || togglingScrapers[stat.source]"
                    @click="toggleScraper(stat.source)"
                    class="transition-opacity duration-200"
                    :title="
                      emergencyStopActive
                        ? 'Emergency stop active'
                        : togglingScrapers[stat.source]
                          ? 'Updating...'
                          : stat.is_enabled
                            ? 'Click to disable'
                            : 'Click to enable'
                    "
                  />
                </TableCell>
              </TableRow>
            </TableBody>
          </Table>
        </div>
      </div>

      <!-- Loading State -->
      <div v-if="loading" class="space-y-4">
        <div v-for="i in 3" :key="i" class="bg-white rounded-lg border border-gray-200 shadow-sm p-6">
          <div class="flex items-start justify-between mb-4">
            <div class="space-y-2">
              <div class="h-6 bg-gray-200 rounded animate-pulse w-48"></div>
              <div class="h-4 bg-gray-200 rounded animate-pulse w-32"></div>
            </div>
            <div class="h-8 bg-gray-200 rounded animate-pulse w-24"></div>
          </div>
          <div class="space-y-2 mb-4">
            <div class="h-4 bg-gray-200 rounded animate-pulse w-full"></div>
            <div class="h-4 bg-gray-200 rounded animate-pulse w-3/4"></div>
          </div>
        </div>
      </div>

      <!-- Statistics and Logs -->
      <div v-else>
        <!-- Logs Table -->
        <div class="bg-white rounded-lg shadow-sm overflow-hidden mb-8 text-base">
          <!-- Table Header -->
          <div class="p-4 border-b border-gray-200">
            <div class="flex items-center justify-between gap-4">
              <div class="flex items-center gap-4">
                <div class="bg-emerald-50 p-2 rounded-lg">
                  <ClipboardList class="w-5 h-5 text-emerald-500" />
                </div>
                <div>
                  <h2 class="text-xl font-semibold text-gray-800">Recent Scraper Logs</h2>
                  <p class="text-sm text-gray-500 mt-1">Overview of recent data collection activities</p>
                </div>
              </div>
              
              <!-- Source filter -->
              <select
                v-model="selectedSource"
                class="border rounded px-3 py-1"
                @change="currentPage = 1; fetchStats()"
              >
                <option value="">All Sources</option>
                <option
                  v-for="stat in scraperHealth"
                  :key="stat.source"
                  :value="stat.source"
                >
                  {{ stat.source }}
                </option>
              </select>
            </div>
          </div>
          <Table class="text-base border-collapse">
            <TableHeader>
              <TableRow>
                <TableHead class="text-base border-b border-gray-200 px-4 py-2 bg-gray-50">Keyword</TableHead>
                <TableHead class="text-base border-b border-gray-200 px-4 py-2 bg-gray-50">Source</TableHead>
                <TableHead class="text-base border-b border-gray-200 px-4 py-2 bg-gray-50">Status</TableHead>
                <TableHead class="text-base border-b border-gray-200 px-4 py-2 bg-gray-50">Started</TableHead>
                <TableHead class="text-base border-b border-gray-200 px-4 py-2 bg-gray-50">Posts Found</TableHead>
                <TableHead class="text-base border-b border-gray-200 px-4 py-2 bg-gray-50">Error</TableHead>
              </TableRow>
            </TableHeader>
            <TableBody>
              <TableRow v-for="log in stats.recent_logs" :key="log.id" class="hover:bg-gray-50">
                <TableCell class="text-base border-b border-gray-100 px-4">
                  <span class="px-2 py-1 rounded-md text-base bg-gray-50 text-gray-700">
                    {{ log.keyword_term }}
                  </span>
                </TableCell>
                <TableCell class="text-base border-b border-gray-100 px-4">
                  <span :class="['px-2 py-1 rounded-md text-base', getSourceColor(log.source)]">
                    {{ log.source }}
                  </span>
                </TableCell>
                <TableCell class="text-base border-b border-gray-100 px-4">
                  <span :class="['px-2 py-1 rounded-md text-base', getStatusColor(log.status)]">
                    {{ log.status }}
                  </span>
                </TableCell>
                <TableCell class="text-base border-b border-gray-100 px-4">{{ formatDate(log.started_at) }}</TableCell>
                <TableCell class="text-base border-b border-gray-100 px-4">{{ log.posts_found }}</TableCell>
                <TableCell class="text-base border-b border-gray-100 px-4 max-w-xs truncate" :title="log.error_message">
                  {{ log.error_message || '-' }}
                </TableCell>
              </TableRow>
              <TableRow v-if="stats.recent_logs.length === 0">
                <TableCell colspan="6" class="text-center py-4 text-gray-500">
                  No recent scraper logs found
                </TableCell>
              </TableRow>
            </TableBody>
          </Table>
          <!-- Pagination -->
          <div class="border-t border-gray-200 px-4 py-3 flex items-center justify-between">
            <span class="text-sm text-gray-500">
              Showing {{ stats.recent_logs.length }} of {{ totalItems }} logs
            </span>
            <div class="flex items-center gap-2">
              <button class="p-1 rounded" :disabled="currentPage === 1" @click="currentPage--">
                <ChevronLeft class="w-5 h-5" :class="currentPage === 1 ? 'text-gray-400' : 'text-gray-600'" />
              </button>
              <span class="text-sm text-gray-600">Page {{ currentPage }} of {{ totalPages }}</span>
              <button class="p-1 rounded" :disabled="currentPage === totalPages" @click="currentPage++">
                <ChevronRight class="w-5 h-5" :class="currentPage === totalPages ? 'text-gray-400' : 'text-gray-600'" />
              </button>
            </div>
          </div>
        </div>
      </div>

      <!-- Raw Log Viewer -->
      <div class="bg-white rounded-lg shadow-sm overflow-hidden mb-8 text-base">
        <div class="p-4 border-b border-gray-200">
          <div class="flex items-center justify-between gap-4">
            <div class="flex items-center gap-4">
              <div class="bg-emerald-50 p-2 rounded-lg">
                <ClipboardList class="w-5 h-5 text-emerald-500" />
              </div>
              <div>
                <h2 class="text-xl font-semibold text-gray-800">Live Scraper Logs</h2>
                <p class="text-sm text-gray-500 mt-1">View raw scraper logs in real-time</p>
              </div>
            </div>
            
            <!-- Scraper selector -->
            <select
              v-model="selectedLogScraper"
              class="border rounded px-3 py-1"
            >
              <option value="">Select Scraper</option>
              <option
                v-for="scraper in rawLogScrapers"
                :key="scraper.filename"
                :value="scraper.scraper"
              >
                {{ scraper.scraper }}
              </option>
            </select>
          </div>
        </div>
        
        <!-- Log display -->
        <div
          v-if="selectedLogScraper"
          class="raw-logs-container h-96 overflow-y-auto font-mono text-base bg-gray-50 p-4 space-y-1"
        >
          <div
            v-for="log in rawLogs"
            :key="log.time"
            class="py-1.5 border-b border-gray-100 last:border-0"
          >
            <span class="text-gray-400 font-mono mr-2">{{ log.time.substring(5, 16) }}</span>
            <span
              class="px-1.5 py-0.5 rounded text-xs font-medium mr-2"
              :class="{
                'bg-blue-100 text-blue-700': log.level === 'INFO',
                'bg-gray-100 text-gray-700': log.level === 'DEBUG',
                'bg-yellow-100 text-yellow-700': log.level === 'WARNING',
                'bg-red-100 text-red-700': log.level === 'ERROR'
              }"
            >{{ log.level }}</span>
            <span class="text-gray-700">{{ log.msg }}</span>
          </div>
        </div>
        
        <!-- Placeholder -->
        <div
          v-else
          class="h-96 flex items-center justify-center text-gray-500"
        >
          Select a scraper to view live logs
        </div>
      </div>
    </div>
  </div>
</template>
