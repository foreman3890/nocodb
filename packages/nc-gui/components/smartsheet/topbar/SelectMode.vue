<script lang="ts" setup>
import { ActiveViewInj, inject, ref, storeToRefs, useViewsStore } from '#imports'

const activeView = inject(ActiveViewInj, ref())

const { openedViewsTab } = storeToRefs(useViewsStore())

const { onViewsTabChange } = useViewsStore()
</script>

<template>
  <div class="flex flex-row p-1 mx-3 mt-3 mb-3 bg-gray-100 rounded-lg gap-x-0.5 nc-view-sidebar-tab">
    <div
      class="tab"
      :class="{
        active: openedViewsTab === 'view',
      }"
      @click="onViewsTabChange('view')"
    >
      <GeneralViewIcon class="tab-icon" :meta="{ type: activeView?.type }" ignore-color />
      <div class="tab-title nc-tab">Data</div>
    </div>
    <div
      class="tab"
      :class="{
        active: openedViewsTab !== 'view',
      }"
      @click="onViewsTabChange(isEeUI ? 'field' : 'relation')"
    >
      <GeneralIcon
        icon="erd"
        class="tab-icon"
        :class="{}"
        :style="{
          fontWeight: 600,
        }"
      />
      <div class="tab-title nc-tab">Details</div>
    </div>
  </div>
</template>

<style scoped>
.tab {
  @apply flex flex-row items-center h-7.5 justify-center px-2 py-1 bg-gray-100 rounded-lg gap-x-1.5 text-gray-500 hover:text-black cursor-pointer transition-all duration-300 select-none;
}

.tab-icon {
  font-size: 1.1rem;
}
.tab .tab-title {
  @apply min-w-0;
  word-break: 'keep-all';
  white-space: 'nowrap';
  display: 'inline';
}

.active {
  @apply bg-white shadow text-brand-500 hover:text-brand-500;
}
</style>
