<script setup lang="ts">
import type { Ref } from 'vue'

import { useBewlyApp } from '~/composables/useAppProvider'
import type { GridLayoutType } from '~/logic'
import type { DataItem as MomentItem, MomentResult } from '~/models/moment/moment'
import api from '~/utils/api'

// https://github.com/starknt/BewlyBewly/blob/fad999c2e482095dc3840bb291af53d15ff44130/src/contentScripts/views/Home/components/ForYou.vue#L16
interface VideoElement {
  uniqueId: string
  item?: MomentItem
}

const props = defineProps<{
  gridLayout: GridLayoutType
}>()

const emit = defineEmits<{
  (e: 'beforeLoading'): void
  (e: 'afterLoading'): void
}>()

const gridValue = computed((): string => {
  if (props.gridLayout === 'adaptive')
    return '~ 2xl:cols-5 xl:cols-4 lg:cols-3 md:cols-2 gap-5'
  if (props.gridLayout === 'twoColumns')
    return '~ cols-1 xl:cols-2 gap-4'
  return '~ cols-1 gap-4'
})

const videoList = ref<VideoElement[]>([])
const isLoading = ref<boolean>(false)
const needToLoginFirst = ref<boolean>(false)
const containerRef = ref<HTMLElement>() as Ref<HTMLElement>
const offset = ref<string>('')
const updateBaseline = ref<string>('')
const noMoreContent = ref<boolean>(false)
const noMoreContentWarning = ref<boolean>(false)
const { handleReachBottom, handlePageRefresh, haveScrollbar } = useBewlyApp()

onMounted(async () => {
  initData()
  initPageAction()
})

onActivated(() => {
  initPageAction()
})

async function initData() {
  offset.value = ''
  updateBaseline.value = ''
  videoList.value.length = 0
  noMoreContent.value = false
  noMoreContentWarning.value = false

  await getData()
}

async function getData() {
  emit('beforeLoading')
  isLoading.value = true

  try {
    for (let i = 0; i < 3; i++)
      await getFollowedUsersVideos()
  }
  finally {
    isLoading.value = false
    emit('afterLoading')
  }
}

function initPageAction() {
  handleReachBottom.value = async () => {
    if (isLoading.value)
      return
    if (noMoreContent.value) {
      noMoreContentWarning.value = true
      return
    }
    getData()
  }
  handlePageRefresh.value = async () => {
    if (isLoading.value)
      return
    if (isLoading.value)
      return
    initData()
  }
}

async function getFollowedUsersVideos() {
  if (noMoreContent.value)
    return

  if (offset.value === '0') {
    noMoreContent.value = true
    return
  }

  try {
    let i = 0
    // https://github.com/starknt/BewlyBewly/blob/fad999c2e482095dc3840bb291af53d15ff44130/src/contentScripts/views/Home/components/ForYou.vue#L208
    const pendingVideos: VideoElement[] = Array.from({ length: 30 }, () => ({
      uniqueId: `unique-id-${(videoList.value.length || 0) + i++})}`,
    } satisfies VideoElement))
    let lastVideoListLength = videoList.value.length
    videoList.value.push(...pendingVideos)

    const response: MomentResult = await api.moment.getMoments({
      type: 'pgc',
      offset: Number(offset.value),
      update_baseline: updateBaseline.value,
    })

    if (response.code === -101) {
      noMoreContent.value = true
      needToLoginFirst.value = true
      return
    }

    if (response.code === 0) {
      offset.value = response.data.offset
      updateBaseline.value = response.data.update_baseline

      const resData = [] as MomentItem[]

      response.data.items.forEach((item: MomentItem) => {
        resData.push(item)
      })

      // when videoList has length property, it means it is the first time to load
      if (!videoList.value.length) {
        videoList.value = resData.map(item => ({ uniqueId: `${item.id_str}`, item }))
      }
      else {
        resData.forEach((item) => {
          videoList.value[lastVideoListLength++] = {
            uniqueId: `${item.id_str}`,
            item,
          }
        })
      }

      if (!haveScrollbar() && !noMoreContent.value) {
        getFollowedUsersVideos()
      }
    }
    else if (response.code === -101) {
      needToLoginFirst.value = true
    }
  }
  finally {
    videoList.value = videoList.value.filter(video => video.item)
  }
}

function jumpToLoginPage() {
  location.href = 'https://passport.bilibili.com/login'
}

defineExpose({ initData })
</script>

<template>
  <div>
    <!-- By directly using predefined unocss grid properties, it is possible to dynamically set the grid attribute -->
    <div hidden grid="~ 2xl:cols-5 xl:cols-4 lg:cols-3 md:cols-2 gap-5" />
    <div hidden grid="~ cols-1 xl:cols-2 gap-4" />
    <div hidden grid="~ cols-1 gap-4" />

    <Empty v-if="needToLoginFirst" mt-6 :description="$t('common.please_log_in_first')">
      <Button type="primary" @click="jumpToLoginPage()">
        {{ $t('common.login') }}
      </Button>
    </Empty>
    <Empty v-if="videoList.length === 0 && !needToLoginFirst" mt-6 :description="$t('common.no_more_content')">
      <Button type="primary" @click="initData()">
        {{ $t('common.operation.refresh') }}
      </Button>
    </Empty>
    <div
      v-else
      ref="containerRef"
      m="b-0 t-0" relative w-full h-full
      :grid="gridValue"
    >
      <VideoCard
        v-for="video in videoList"
        :key="video.uniqueId"
        :skeleton="!video.item"
        type="bangumi"
        :video="video.item ? {
          id: video.item.modules.module_author.mid,
          title: `${video.item.modules.module_dynamic.major.pgc?.title}`,
          cover: `${video.item.modules.module_dynamic.major.pgc?.cover}`,
          author: {
            name: video.item.modules.module_author.name,
            authorUrl: video.item.modules.module_author.jump_url,
            authorFace: video.item.modules.module_author.face,
            mid: video.item.modules.module_author.mid,
          },
          viewStr: video.item.modules.module_dynamic.major.pgc?.stat.play,
          danmakuStr: video.item.modules.module_dynamic.major.pgc?.stat.danmaku,
          capsuleText: video.item.modules.module_author.pub_time,
          epid: video.item.modules.module_dynamic.major.pgc?.epid,
        } : undefined"
        :show-watcher-later="false"
        :horizontal="gridLayout !== 'adaptive'"
      />
    </div>

    <!-- no more content -->
    <Empty v-if="noMoreContentWarning" class="pb-4" :description="$t('common.no_more_content')" />
  </div>
</template>

<style lang="scss" scoped>
</style>
