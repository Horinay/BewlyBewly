<script setup lang="ts">
import { Icon } from '@iconify/vue'

import { useBewlyApp } from '~/composables/useAppProvider'
import { useDark } from '~/composables/useDark'
import { AppPage } from '~/enums/appEnums'
import { settings } from '~/logic'
import type { DockItem } from '~/stores/mainStore'
import { useMainStore } from '~/stores/mainStore'

import Tooltip from '../Tooltip.vue'
import type { HoveringDockItem } from './types'

const props = defineProps<{
  activatedPage: AppPage
}>()

// const emit = defineEmits(['pageChange', 'settingsVisibilityChange', 'refresh', 'backToTop'])
const emit = defineEmits<{
  (e: 'dockItemClick', dockItem: DockItem): void
  (e: 'settingsVisibilityChange'): void
  (e: 'refresh'): void
  (e: 'backToTop'): void
}>()

const mainStore = useMainStore()
const { isDark, toggleDark } = useDark()
const { reachTop } = useBewlyApp()

const hideDock = ref<boolean>(false)
const hoveringDockItem = reactive<HoveringDockItem>({
  themeMode: false,
  settings: false,
})
const currentDockItems = ref<DockItem[]>([])
const activatedDockItem = ref<DockItem>()

const tooltipPlacement = computed(() => {
  if (settings.value.dockPosition === 'left')
    return 'right'
  else if (settings.value.dockPosition === 'right')
    return 'left'
  else if (settings.value.dockPosition === 'bottom')
    return 'top'
  return 'right'
})

/**
 * Whether to show the back to top or refresh button
 */
const showBackToTopOrRefreshButton = computed((): boolean => {
  const dockItemConfig = settings.value.dockItemsConfig.find(e => e.page === props.activatedPage)
  if (dockItemConfig && dockItemConfig.useOriginalBiliPage) {
    return false
  }

  return settings.value.moveBackToTopOrRefreshButtonToDock
    && props.activatedPage !== AppPage.Search
})

watch(() => settings.value.autoHideDock, (newValue) => {
  hideDock.value = newValue
}, { immediate: true })

// use Json stringify to watch the changes of the array item properties
watch(() => JSON.stringify(settings.value.dockItemsConfig), () => {
  currentDockItems.value = computeDockItem()
}, { immediate: true })

function computeDockItem(): DockItem[] {
  // Transfer the data from dockItemVisibilityList into dockItemsConfig
  if (settings.value.dockItemVisibilityList.length > 0 && settings.value.dockItemsConfig.length === 0) {
    settings.value.dockItemsConfig = settings.value.dockItemVisibilityList.map(item =>
      ({
        page: item.page,
        visible: item.visible,
        openInNewTab: false,
        useOriginalBiliPage: false,
      }))
  }

  // if dockItemVisibilityList not fresh , set it to default
  if (!settings.value.dockItemsConfig.length || settings.value.dockItemsConfig.length !== mainStore.dockItems.length)
    settings.value.dockItemsConfig = mainStore.dockItems.map(dock => ({ page: dock.page, visible: true, openInNewTab: false, useOriginalBiliPage: false }))

  const targetDockItems: DockItem[] = []

  settings.value.dockItemsConfig.forEach((item) => {
    const foundItem = mainStore.dockItems.find(defaultItem => defaultItem.page === item.page)
    item.visible && targetDockItems.push({
      i18nKey: foundItem?.i18nKey || '',
      icon: foundItem?.icon || '',
      iconActivated: foundItem?.iconActivated || '',
      page: foundItem?.page || AppPage.Home,
      openInNewTab: item.openInNewTab,
      useOriginalBiliPage: item.useOriginalBiliPage || false,
      url: foundItem?.url || '',
    })
  })
  return targetDockItems
}

function toggleHideDock(hide: boolean) {
  if (settings.value.autoHideDock)
    hideDock.value = hide
  else
    hideDock.value = false
}

function handleDockItemClick(dockItem: DockItem) {
  activatedDockItem.value = dockItem
  emit('dockItemClick', dockItem)
}

function handleBackToTopOrRefresh() {
  if (reachTop.value)
    emit('refresh')
  else
    emit('backToTop')
}
</script>

<template>
  <aside
    class="dock-wrap"
    pos="absolute top-0" flex="~ col justify-center items-center" w-full h-full
    z-1 pointer-events-none
  >
    <!-- Edge Div -->
    <div
      v-if="settings.autoHideDock && hideDock"
      class="dock-edge"
      :class="`dock-edge-${settings.dockPosition}`"
      @mouseenter="toggleHideDock(false)"
      @mouseleave="toggleHideDock(true)"
    />

    <!-- Dock Content -->
    <div
      class="dock-content"
      :class="{
        left: settings.dockPosition === 'left',
        right: settings.dockPosition === 'right',
        bottom: settings.dockPosition === 'bottom',
        hide: hideDock,
      }"
      @mouseenter="toggleHideDock(false)"
      @mouseleave="toggleHideDock(true)"
    >
      <div
        class="dock-content-inner"
      >
        <template v-for="dockItem in currentDockItems" :key="dockItem.page">
          <Tooltip :content="$t(dockItem.i18nKey)" :placement="tooltipPlacement">
            <button
              class="dock-item group"
              :class="{
                'active': activatedPage === dockItem.page,
                'inactive': hoveringDockItem.themeMode && isDark,
                'disable-glowing-effect': settings.disableDockGlowingEffect,
              }"
              @click="handleDockItemClick(dockItem)"
            >
              <div
                v-show="activatedPage !== dockItem.page"
                :class="dockItem.icon"
                text-xl
              />
              <div
                v-show="activatedPage === dockItem.page"
                :class="dockItem.iconActivated"
                text-xl
              />
            </button>
          </Tooltip>
        </template>

        <!-- dividing line -->
        <div class="divider" />

        <Tooltip
          v-if="!settings.disableLightDarkModeSwitcherOnDock"
          :content="isDark ? $t('dock.dark_mode') : $t('dock.light_mode')" :placement="tooltipPlacement"
          class="group"
          pointer-events-none
        >
          <!-- moon -->
          <div
            v-if="isDark"
            pos="absolute top-0 left-0 group-hover:top-2px group-hover:left--4px"
            w-full h-full bg-white rounded="1/2"
            z--2 pointer-events-none
            shadow="group-hover:[-8px_4px_160px_20px_hsla(226deg,85%,77%,1),-8px_4px_100px_12px_hsla(226deg,85%,77%,0.8),-8px_4px_60px_10px_hsla(226deg,85%,77%,0.6),-8px_4px_20px_4px_hsla(226deg,85%,77%,0.4),-4px_2px_8px_0_hsla(226deg,85%,77%,0.8)]"
            opacity-0 group-hover:opacity-100
            duration-600
          />

          <button
            class="dock-item"
            bg="!dark-hover:$bew-bg" transform="!dark-hover:scale-100" shadow="!dark-hover:[inset_4px_-2px_8px_hsla(226deg,85%,77%,1)]"
            pointer-events-auto
            @click="toggleDark"
            @mouseenter="hoveringDockItem.themeMode = true"
            @mouseleave="hoveringDockItem.themeMode = false"
          >
            <Transition name="fade">
              <div v-show="hoveringDockItem.themeMode" absolute>
                <Icon v-if="isDark" icon="line-md:sunny-outline-to-moon-loop-transition" />
                <Icon v-else icon="line-md:moon-alt-to-sunny-outline-loop-transition" />
              </div>
            </Transition>
            <Transition name="fade">
              <div v-show="!hoveringDockItem.themeMode" absolute>
                <Icon v-if="isDark" icon="line-md:sunny-outline-to-moon-transition" />
                <Icon v-else icon="line-md:moon-to-sunny-outline-transition" />
              </div>
            </Transition>
          </button>
        </Tooltip>

        <Tooltip :content="$t('dock.settings')" :placement="tooltipPlacement">
          <button
            class="dock-item group"
            :class="{
              inactive: hoveringDockItem.themeMode && isDark,
            }"
            @click="emit('settingsVisibilityChange')"
          >
            <div i-mingcute:settings-3-line text-xl group-hover:rotate-180 transition="all 2000 ease-out" />
          </button>
        </Tooltip>
      </div>

      <button
        v-if="showBackToTopOrRefreshButton"
        class="back-to-top-or-refresh-btn"
        :class="{
          inactive: hoveringDockItem.themeMode && isDark,
        }"
        @click="handleBackToTopOrRefresh"
      >
        <Transition name="fade">
          <Icon
            v-if="reachTop"
            icon="line-md:rotate-270"
            shrink-0 rotate-90 absolute text-2xl
          />
          <Icon
            v-else
            icon="line-md:arrow-small-up"
            shrink-0 absolute text-2xl
          />
        </Transition>
      </button>
    </div>
  </aside>
</template>

<style lang="scss" scoped>
.dock-wrap {
  > * {
    --uno: "pointer-events-auto";
  }
}

.dock-edge {
  &-left,
  &-right,
  &-bottom {
    --uno: "absolute z--1";
  }

  &-left {
    --uno: "left-0 top-0 w-14px h-full hover:w-60px";
  }

  &-right {
    --uno: "right-0 top-0 w-14px h-full hover:w-60px";
  }

  &-bottom {
    --uno: "left-0 bottom-0 w-full h-14px hover-h-60px";
  }
}

.dock-content {
  --uno: "absolute flex justify-center items-center duration-300";

  &.left {
    --uno: "left-2 after:right--4px";
  }
  &.left.hide {
    --uno: "opacity-0 translate-x--100%";
  }

  &.right {
    --uno: "right-2 after:left--4px";
  }
  &.right.hide {
    --uno: "opacity-0 translate-x-100%";
  }

  &.bottom {
    --uno: "top-unset bottom-0";
  }
  &.bottom.hide {
    --uno: "opacity-0 translate-y-100%";
  }

  .divider {
    --uno: "my-1 mx-3 h-3px bg-$bew-border-color rounded-4";
  }

  &.bottom .divider {
    --uno: "w-3px h-auto my-3 mx-1";
  }

  .dock-content-inner {
    --uno: "duration-300 ease-in-out transform-gpu";
    --uno: "p-2 m-2 bg-$bew-content-alt dark:bg-$bew-elevated";
    --uno: "flex flex-col gap-2 shrink-0";
    --uno: "rounded-full border-1 border-$bew-border-color";
    box-shadow: var(--bew-shadow-edge-glow-1), var(--bew-shadow-2);
    backdrop-filter: var(--bew-filter-glass-1);
  }

  &.bottom .dock-content-inner {
    --uno: "flex-row";
  }

  .back-to-top-or-refresh-btn {
    --uno: "absolute lg:bottom--45px bottom--35px";
    --uno: "transform active:important-scale-90 hover:scale-110";
    --uno: "lg:w-45px w-35px lg:h-45px h-35px";
    --uno: "grid place-items-center";
    --uno: "filter-$bew-filter-glass-1";
    --uno: "bg-$bew-elevated hover:bg-$bew-content-hover";
    --uno: "rounded-full shadow-$bew-shadow-2 border-1 border-$bew-border-color";

    backdrop-filter: var(--bew-filter-glass-1);
    transition:
      transform 300ms cubic-bezier(0.34, 2, 0.6, 1),
      background 300ms ease,
      color 300ms ease,
      box-shadow 300ms ease,
      opacity 600ms ease;
    box-shadow: var(--bew-shadow-edge-glow-1), var(--bew-shadow-2);

    &.active {
      --uno: "important-bg-$bew-theme-color-auto text-$bew-text-auto";
      --uno: "shadow-$shadow-active dark:shadow-$shadow-dark";
      --uno: "active:shadow-$shadow-active-active dark-active:shadow-$shadow-dark-active";
    }

    &.inactive {
      --uno: "opacity-80 !shadow-none";
    }
  }

  &.bottom .back-to-top-or-refresh-btn {
    --uno: "bottom-unset lg:right--45px right--35px";
  }
}

.dock-item {
  --shadow-dark: 0 4px 30px 4px rgba(255, 255, 255, 0.6);
  --shadow-active: 0 4px 30px var(--bew-theme-color-70);
  --shadow-dark-active: 0 4px 20px rgba(255, 255, 255, 0.8);
  --shadow-active-active: 0 4px 20px var(--bew-theme-color-90);

  --uno: "relative transform active:important-scale-90 hover:scale-110";
  --uno: "lg:w-45px w-35px";
  --uno: "lg:lh-45px lh-35px";
  --uno: "p-0 flex items-center justify-center";
  --uno: "aspect-square relative";
  --uno: "leading-0";
  --uno: "rounded-60px antialiased";
  --uno: "bg-$bew-fill-alt hover:bg-$bew-fill-2 cursor-pointer";
  --uno: "dark:bg-$bew-fill-1 dark-hover:bg-$bew-fill-4";

  box-shadow: var(--bew-shadow-edge-glow-1), var(--bew-shadow-1);
  transition:
    transform 300ms cubic-bezier(0.34, 2, 0.6, 1),
    background 300ms ease,
    color 300ms ease,
    box-shadow 600ms ease,
    opacity 600ms ease;

  &:hover {
    box-shadow:
      var(--bew-shadow-edge-glow-1),
      0 0 0 2px var(--bew-fill-2),
      var(--bew-shadow-2);
  }

  &.disable-glowing-effect {
    box-shadow: var(--bew-shadow-edge-glow-1), var(--bew-shadow-1) !important;
  }

  &.active {
    --uno: "important-bg-$bew-theme-color-auto text-$bew-text-auto dark:text-$bew-theme-color";
    --uno: "shadow-$shadow-active dark:shadow-$shadow-dark";
    --uno: "active:shadow-$shadow-active-active dark-active:shadow-$shadow-dark-active";
  }

  &.inactive {
    --uno: "opacity-80 !shadow-none";
  }

  svg {
    --uno: "lg:w-22px w-18px lg:h-22px h-18px block align-middle";
  }
}
</style>
