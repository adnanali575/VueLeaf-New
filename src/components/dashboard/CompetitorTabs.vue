<script setup lang="ts">
import { computed } from 'vue'
import type { CompetitorData } from '@/services/competitors'
import { useCompetitorStore } from '@/stores/competitors'

interface Props {
  competitors: CompetitorData[]
}

const props = defineProps<Props>()
const store = useCompetitorStore()

const selectedCompetitor = computed(() => store.selectedCompetitor)

const handleSelect = (competitor: string) => {
  store.setSelectedCompetitor(competitor)
}
</script>

<template>
  <div class="flex gap-2 border-b border-gray-200 pb-6 overflow-x-auto">
    <button v-for="competitor in competitors" :key="competitor.name" @click="handleSelect(competitor.name)"
      class="whitespace-nowrap" :class="[
        'px-4 py-2 text-sm font-medium rounded-lg',
        selectedCompetitor === competitor.name
          ? 'bg-green-50 text-green-700 border-b-2 border-green-500'
          : 'text-gray-500 hover:text-gray-700 hover:bg-gray-50'
      ]">
      {{ competitor.name }}
    </button>
  </div>
</template>
