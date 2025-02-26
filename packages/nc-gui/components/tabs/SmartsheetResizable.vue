<script lang="ts" setup>
import { Pane, Splitpanes } from 'splitpanes'
import 'splitpanes/dist/splitpanes.css'

const {
  isRightSidebarOpen,
  isLeftSidebarOpen,
  leftSidebarWidthPercent,
  rightSidebarSize: sideBarSize,
} = storeToRefs(useSidebarStore())
const wrapperRef = ref<HTMLDivElement>()
const splitpaneWrapperRef = ref()

const contentSize = computed(() => 100 - sideBarSize.value.current)
const isSidebarShort = ref(!isRightSidebarOpen.value)
const animationDuration = 300
const contentDomWidth = ref(window.innerWidth)
const isMouseOverShowSidebarZone = ref(false)
const isAnimationEndAfterSidebarHide = ref(!isRightSidebarOpen.value)
const isStartHideSidebarAnimation = ref(false)
const isLeftSidebarAnimating = ref(false)

const sidebarWidth = computed(() => (sideBarSize.value.old * contentDomWidth.value) / 100)
const currentSidebarSize = computed({
  get: () => sideBarSize.value.current,
  set: (val) => {
    sideBarSize.value.current = val
    sideBarSize.value.old = val
  },
})

const isSidebarHidden = ref(!isRightSidebarOpen.value)

watch(isRightSidebarOpen, () => {
  sideBarSize.value.current = sideBarSize.value.old

  if (isRightSidebarOpen.value) {
    setTimeout(() => {
      isSidebarShort.value = true

      isSidebarHidden.value = false
    }, 0)

    setTimeout(() => {
      isSidebarShort.value = false
    }, animationDuration / 2)
  } else {
    sideBarSize.value.old = sideBarSize.value.current

    isSidebarShort.value = true
    isAnimationEndAfterSidebarHide.value = false

    setTimeout(() => {
      sideBarSize.value.current = 0
      isSidebarHidden.value = true
      isAnimationEndAfterSidebarHide.value = true
    }, animationDuration * 1.75)
  }
})

function handleMouseMove(e: MouseEvent) {
  if (!wrapperRef.value) return

  if (isRightSidebarOpen.value && !isSidebarHidden.value && !isMouseOverShowSidebarZone.value) return
  if (isRightSidebarOpen.value) {
    isSidebarHidden.value = false
    isMouseOverShowSidebarZone.value = false
    return
  }

  const viewportWidth = window.innerWidth

  if (e.clientX > viewportWidth - 14) {
    isSidebarHidden.value = false
    isMouseOverShowSidebarZone.value = true
  } else if (e.clientX < viewportWidth - (sidebarWidth.value + 10) && !isSidebarHidden.value) {
    isSidebarHidden.value = true
    isMouseOverShowSidebarZone.value = false
    isAnimationEndAfterSidebarHide.value = false

    setTimeout(() => {
      isAnimationEndAfterSidebarHide.value = true
    }, animationDuration * 1.75)
  }
}

function onWindowResize() {
  contentDomWidth.value = ((100 - leftSidebarWidthPercent.value) / 100) * window.innerWidth
}

onMounted(() => {
  contentDomWidth.value = ((100 - leftSidebarWidthPercent.value) / 100) * window.innerWidth
  document.addEventListener('mousemove', handleMouseMove)

  window.addEventListener('resize', onWindowResize)
})

onBeforeUnmount(() => {
  document.removeEventListener('mousemove', handleMouseMove)
  window.removeEventListener('resize', onWindowResize)
})

watch(leftSidebarWidthPercent, () => {
  contentDomWidth.value = ((100 - leftSidebarWidthPercent.value) / 100) * window.innerWidth
})

watch(isLeftSidebarOpen, () => {
  if (isLeftSidebarOpen.value) {
    contentDomWidth.value = ((100 - leftSidebarWidthPercent.value) / 100) * window.innerWidth
  } else {
    contentDomWidth.value = window.innerWidth
  }

  isLeftSidebarAnimating.value = true
  setTimeout(() => {
    isLeftSidebarAnimating.value = false
  }, 700)
})

watch(
  () => !isRightSidebarOpen.value && isSidebarShort.value,
  (value) => {
    if (value) {
      setTimeout(() => {
        isStartHideSidebarAnimation.value = true
      }, animationDuration / 2)
    } else {
      isStartHideSidebarAnimation.value = false
    }
  },
)
</script>

<template>
  <Splitpanes
    ref="splitpaneWrapperRef"
    style="height: calc(100vh - var(--topbar-height))"
    class="smartsheet-resizable-wrapper w-full"
    :class="{
      'smartsheet-sidebar-short': isSidebarShort,
    }"
    @resize="currentSidebarSize = $event[1].size"
  >
    <Pane :size="contentSize">
      <slot name="content" />
    </Pane>
    <Pane
      min-size="15%"
      :size="currentSidebarSize"
      max-size="40%"
      class="nc-smartsheet-sidebar-splitpane !bg-transparent relative !overflow-visible !-ml-0.5"
    >
      <div
        ref="wrapperRef"
        class="nc-smartsheet-sidebar-wrapper relative z-10 !bg-transparent"
        :class="{
          'open': isRightSidebarOpen,
          'close': !isRightSidebarOpen,
          'absolute': isMouseOverShowSidebarZone,
          'smartsheet-sidebar-short': isSidebarShort,
          'hide-sidebar': isStartHideSidebarAnimation && !isMouseOverShowSidebarZone,
          'mouseover-show-sidebar-zone':
            isSidebarShort && !isRightSidebarOpen && isMouseOverShowSidebarZone && isAnimationEndAfterSidebarHide,
        }"
        :style="{
          width: isLeftSidebarAnimating
            ? '100%'
            : isAnimationEndAfterSidebarHide && isSidebarHidden
            ? '0px'
            : `${sidebarWidth}px`,
          overflow: isMouseOverShowSidebarZone ? 'visible' : undefined,
          translate: isMouseOverShowSidebarZone ? 'translateX(-100%)' : undefined,
        }"
      >
        <slot name="sidebar" />
      </div>
    </Pane>
  </Splitpanes>
</template>

<style lang="scss">
.smartsheet-resizable-wrapper > {
  .splitpanes__splitter {
    width: 0 !important;
    position: relative;
    overflow: visible;
  }
  .splitpanes__splitter:before {
    @apply bg-transparent;
    width: 1px;
    content: '';
    position: absolute;
    left: -2px;
    top: 0;
    height: 100%;
    z-index: 40;
  }

  .splitpanes__splitter:hover:before {
    @apply bg-scrollbar;
    z-index: 40;
    width: 4px !important;
    left: -4px;
  }

  .splitpanes--dragging .splitpanes__splitter:before {
    @apply bg-scrollbar;
    z-index: 40;
    width: 10px !important;
    left: -6px;
  }
}

.splitpanes--dragging > .splitpanes__splitter::before {
  @apply w-1 mr-0 bg-scrollbar;
  z-index: 40;
  width: 4px !important;
  left: -4px;
}

.splitpanes--dragging {
  cursor: col-resize;
}

.smartsheet-sidebar-short > .splitpanes__splitter {
  display: none !important;
  background-color: transparent !important;
}

.nc-smartsheet-sidebar-wrapper {
  @apply flex flex-col h-full justify-center min-w-36;
}

.nc-smartsheet-sidebar-wrapper.close {
  > * {
    height: 80vh;
  }
}

.nc-smartsheet-sidebar-wrapper.smartsheet-sidebar-short {
  > * {
    height: 80h !important;
    padding-bottom: 0.35rem;
    margin-top: -3.25rem;
    margin-left: 0.25rem;
  }
}

.nc-smartsheet-sidebar-wrapper.open {
  height: calc(100vh - var(--topbar-height));
  > * {
    height: calc(100vh - var(--topbar-height));
  }
}

.nc-smartsheet-sidebar-wrapper > * {
  height: calc(100% - var(--topbar-height));
}

.nc-smartsheet-sidebar-wrapper > * {
  width: 100%;
  transition: all 0.3s ease-in-out;
}

.nc-smartsheet-sidebar-wrapper.hide-sidebar > * {
  position: absolute;
  opacity: 0;
  transform: translateX(100%);
}
.mouseover-show-sidebar-zone > * {
  transform: translateX(-100%);
}
.nc-smartsheet-sidebar-wrapper.smartsheet-sidebar-short > * {
  @apply !(rounded-l-lg border-1 border-gray-200 shadow-lg);
}
</style>
