<script setup lang="ts">
import { computed } from 'vue'
import { Search, Filter } from 'lucide-vue-next'
import feedbackImage from '@/assets/images/analytics/analytics-green.png'
import { Input } from '@/components/ui/input'
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select'
import { useFeedback } from '../composables/useFeedback'

const feedbackStore = useFeedback()

// Computed properties to safely access store values
const filterStatus = computed({
  get: () => feedbackStore.filterStatus.value,
  set: (value: string) => feedbackStore.setFilterStatus(value)
})

const searchTerm = computed({
  get: () => feedbackStore.searchTerm.value,
  set: (value: string) => feedbackStore.setSearchTerm(value)
})
</script>

<template>
  <div class="mb-8 space-y-6">
    <div class="mb-8 bg-white rounded-lg shadow-sm p-4 sm:p-6 space-y-6">
      <!-- Header Content -->
      <div class="flex flex-col md:flex-row items-center gap-6">
        <div class="flex-1 text-center md:text-left">
          <h1 class="text-lg md:text-2xl font-medium text-green-800 mb-2 text-center md:text-left">
            Welcome to the Feedback Center
          </h1>
          <p class="text-gray-600 text-sm sm:text-md md:text-lg">
            Help us make your brand management experience even better. Share your thoughts, suggestions, or report any
            issues you encounter while using our platform. Your feedback goes directly to our admin team and helps shape
            future improvements.
          </p>
        </div>
        <div class="w-full max-w-xs sm:max-w-sm md:w-28 md:h-28 flex-shrink-0 relative">
          <img :src="feedbackImage" alt="Feedback Illustration" class="w-full h-full object-contain" />
        </div>
      </div>
    </div>

    <!-- Filters -->
    <div class="flex flex-col sm:flex-row justify-between items-center gap-4 bg-white rounded-lg shadow-sm p-4">
      <div class="flex flex-col space-y-4 w-full sm:max-w-44">
        <div class="relative">
          <Filter class="absolute left-3 top-1/2 -translate-y-1/2 h-5 w-5 text-gray-500 pointer-events-none" />
          <select v-model="filterStatus"
            class="w-full bg-white border border-gray-300 rounded-md pl-10 pr-3 py-2 focus:outline-none focus:ring-2 focus:ring-green-500 focus:border-green-500 appearance-none">
            <option value="all">All Statuses</option>
            <option value="pending">Pending</option>
            <option value="reviewed">Under Review</option>
            <option value="resolved">Resolved</option>
          </select>
        </div>
      </div>

      <div class="relative w-full sm:w-auto">
        <div class="flex flex-col space-y-4">
          <div class="relative">
            <Search class="absolute left-3 top-1/2 -translate-y-1/2 h-5 w-5 text-gray-500 pointer-events-none" />
            <input v-model="searchTerm" type="text" placeholder="Search feedback..."
              class="w-full pl-10 pr-4 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-green-500 focus:border-green-500 disabled:bg-gray-50 disabled:cursor-not-allowed" />
          </div>
        </div>
        <Search class="h-5 w-5 absolute left-3 top-1/2 transform -translate-y-1/2 text-gray-400" />
      </div>
    </div>
  </div>
</template>