<script setup lang="ts">
import { useI18n } from 'vue-i18n'

import { useBewlyApp } from '~/composables/useAppProvider'
import { TOP_BAR_VISIBILITY_CHANGE } from '~/constants/globalEvents'
import type { GridLayoutType } from '~/logic'
import { settings } from '~/logic'
import type { List as RankingVideoItem, RankingResult } from '~/models/video/ranking'
import type { List as RankingPgcItem, RankingPgcResult } from '~/models/video/rankingPgc'
import api from '~/utils/api'
import emitter from '~/utils/mitt'

import type { RankingType } from '../types'

const props = defineProps<{
  gridLayout: GridLayoutType
}>()

const emit = defineEmits<{
  (e: 'beforeLoading'): void
  (e: 'afterLoading'): void
}>()

const { t } = useI18n()
const { handleBackToTop, handlePageRefresh } = useBewlyApp()

const gridValue = computed((): string => {
  if (props.gridLayout === 'adaptive') {
    // eslint-disable-next-line ts/no-use-before-define
    if (!activatedRankingType.value.seasonType)
      return '~ 2xl:cols-4 xl:cols-3 lg:cols-2 md:cols-1 gap-5'
    else
      return '~ 2xl:cols-5 xl:cols-4 lg:cols-3 md:cols-2 gap-5'
  }

  if (props.gridLayout === 'twoColumns')
    return '~ cols-1 xl:cols-2 gap-4'
  return '~ cols-1 gap-4'
})

const rankingTypes = computed((): RankingType[] => {
  return [
    { id: 1, name: t('ranking.all'), rid: 0 },
    { id: 2, name: t('topbar.logo_dropdown.anime'), seasonType: 1 },
    { id: 3, name: t('topbar.logo_dropdown.chinese_anime'), seasonType: 4 },
    { id: 4, name: t('ranking.chinese_anime_related'), rid: 168 },
    { id: 5, name: t('topbar.logo_dropdown.documentary_films'), seasonType: 3 },
    { id: 6, name: t('topbar.logo_dropdown.animations'), rid: 1 },
    { id: 7, name: t('topbar.logo_dropdown.music'), rid: 3 },
    { id: 8, name: t('topbar.logo_dropdown.dance'), rid: 129 },
    { id: 9, name: t('topbar.logo_dropdown.gaming'), rid: 4 },
    { id: 10, name: t('topbar.logo_dropdown.knowledge'), rid: 36 },
    { id: 11, name: t('topbar.logo_dropdown.technology'), rid: 188 },
    { id: 12, name: t('topbar.logo_dropdown.sports'), rid: 234 },
    { id: 13, name: t('topbar.logo_dropdown.cars'), rid: 223 },
    { id: 14, name: t('topbar.logo_dropdown.life'), rid: 160 },
    { id: 15, name: t('topbar.logo_dropdown.foods'), rid: 211 },
    { id: 16, name: t('topbar.logo_dropdown.animals'), rid: 217 },
    { id: 17, name: t('topbar.logo_dropdown.kichiku'), rid: 119 },
    { id: 18, name: t('topbar.logo_dropdown.fashion'), rid: 155 },
    { id: 19, name: t('topbar.logo_dropdown.showbiz'), rid: 5 },
    { id: 20, name: t('topbar.logo_dropdown.cinephile'), rid: 181 },
    { id: 21, name: t('topbar.logo_dropdown.movies'), seasonType: 2 },
    { id: 22, name: t('topbar.logo_dropdown.tv_shows'), seasonType: 5 },
    { id: 23, name: t('topbar.logo_dropdown.variety_shows'), seasonType: 7 },
    { id: 24, name: t('ranking.original_content'), rid: 0, type: 'origin' },
    { id: 25, name: t('ranking.debut_work'), rid: 0, type: 'rookie' },
  ]
})

const isLoading = ref<boolean>(false)
const activatedRankingType = ref<RankingType>({ ...rankingTypes.value[0] })
const videoList = reactive<RankingVideoItem[]>([])
const PgcList = reactive<RankingPgcItem[]>([])
const shouldMoveAsideUp = ref<boolean>(false)

watch(() => activatedRankingType.value.id, () => {
  handleBackToTop(settings.value.useSearchPageModeOnHomePage ? 510 : 0)

  initData()
})

onMounted(() => {
  emitter.on(TOP_BAR_VISIBILITY_CHANGE, (val) => {
    shouldMoveAsideUp.value = false

    // Allow moving tabs up only when the top bar is not hidden & is set to auto-hide
    // This feature is primarily designed to compatible with the Bilibili Evolved's top bar
    // Even when the BewlyBewly top bar is hidden, the Bilibili Evolved top bar still exists, so not moving up
    if (settings.value.autoHideTopBar && settings.value.showTopBar) {
      if (val)
        shouldMoveAsideUp.value = false

      else
        shouldMoveAsideUp.value = true
    }
  })

  initData()
  initPageAction()
})

onActivated(() => {
  initPageAction()
})

function initPageAction() {
  handlePageRefresh.value = async () => {
    if (isLoading.value)
      return
    initData()
  }
}

function initData() {
  videoList.length = 0
  PgcList.length = 0
  getData()
}

function getData() {
  if ('seasonType' in activatedRankingType.value)
    getRankingPgc()
  else
    getRankingVideos()
}

onBeforeUnmount(() => {
  emitter.off(TOP_BAR_VISIBILITY_CHANGE)
})

function getRankingVideos() {
  videoList.length = 0
  emit('beforeLoading')
  isLoading.value = true
  api.ranking.getRankingVideos({
    rid: activatedRankingType.value.rid,
    type: 'type' in activatedRankingType.value ? activatedRankingType.value.type : 'all',
  }).then((response: RankingResult) => {
    if (response.code === 0) {
      const { list } = response.data
      Object.assign(videoList, list)
    }
  }).finally(() => {
    isLoading.value = false
    emit('afterLoading')
  })
}

function getRankingPgc() {
  PgcList.length = 0
  isLoading.value = true
  api.ranking.getRankingPgc({
    season_type: activatedRankingType.value.seasonType,
  }).then((response: RankingPgcResult) => {
    if (response.code === 0)
      Object.assign(PgcList, response.data.list)
  }).finally(() => isLoading.value = false)
}

defineExpose({ initData })
</script>

<template>
  <div flex="~ gap-40px">
    <aside
      pos="sticky top-150px" h="[calc(100vh-140px)]" w-200px shrink-0 duration-300
      ease-in-out
      :class="{ hide: shouldMoveAsideUp }"
    >
      <OverlayScrollbarsComponent h-inherit p-20px m--20px defer>
        <ul flex="~ col gap-2">
          <li v-for="rankingType in rankingTypes" :key="rankingType.id">
            <a
              :class="{ active: activatedRankingType.id === rankingType.id }"
              px-4 lh-30px h-30px hover:bg="$bew-fill-2" w-inherit
              block rounded="$bew-radius" cursor-pointer transition="all 300 ease-out"
              hover:scale-105 un-text="$bew-text-1"
              @click="activatedRankingType = rankingType"
            >{{ rankingType.name }}</a>
          </li>
        </ul>
      </OverlayScrollbarsComponent>
    </aside>

    <!-- By directly using predefined unocss grid properties, it is possible to dynamically set the grid attribute -->
    <div hidden grid="~ 2xl:cols-5 xl:cols-4 lg:cols-3 md:cols-2 gap-5" />
    <div hidden grid="~ 2xl:cols-4 xl:cols-3 lg:cols-2 md:cols-1 gap-5" />
    <div hidden grid="~ cols-1 xl:cols-2 gap-4" />
    <div hidden grid="~ cols-1 gap-4" />

    <main w-full :grid="gridValue">
      <template v-if="!('seasonType' in activatedRankingType)">
        <VideoCard
          v-for="(video, index) in videoList"
          :key="video.aid"
          :video="{
            id: Number(video.aid),
            duration: video.duration,
            title: video.title,
            desc: video.desc,
            cover: video.pic,
            author: {
              name: video.owner.name,
              authorFace: video.owner.face,
              mid: video.owner.mid,
            },
            view: video.stat.view,
            danmaku: video.stat.danmaku,
            publishedTimestamp: video.pubdate,
            bvid: video.bvid,
            rank: index + 1,
            cid: video.cid,
          }"
          show-preview
          :horizontal="gridLayout !== 'adaptive'"
          w-full
        />
      </template>
      <template v-else>
        <BangumiCard
          v-for="pgc in PgcList"
          :key="pgc.url"
          :bangumi="{
            url: pgc.url,
            cover: pgc.cover,
            title: pgc.title,
            desc: pgc.new_ep.index_show,
            view: pgc.stat.view,
            follow: pgc.stat.follow,
            rank: pgc.rank,
            capsuleText: pgc.rating.replace('分', ''),
            badge: {
              text: pgc.badge_info.text || '',
              bgColor: pgc.badge_info.bg_color || '',
              bgColorDark: pgc.badge_info.bg_color_night || '',
            },
          }"
          :horizontal="gridLayout !== 'adaptive'"
        />
      </template>

      <!-- skeleton -->
      <template v-if="isLoading">
        <template v-if="!('seasonType' in activatedRankingType)">
          <VideoCardSkeleton
            v-for="item in 30" :key="item"
            :horizontal="gridLayout !== 'adaptive'"
          />
        </template>
        <template v-else>
          <BangumiCardSkeleton
            v-for="item in 30" :key="item"
            :horizontal="gridLayout !== 'adaptive'"
          />
        </template>
      </template>
    </main>
  </div>
</template>

<style lang="scss" scoped>
.active {
  --uno: "scale-110 bg-$bew-theme-color-auto text-$bew-text-auto shadow-$bew-shadow-2";
}

.hide {
  --uno: "h-[calc(100vh-70)] translate-y--70px";
}
</style>
