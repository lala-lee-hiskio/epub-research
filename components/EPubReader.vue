<template>
  <div class="relative flex">
    <ul class="w-1/4">
      <div
        v-if="!navigation"
        class="h-full flex align-center justify-center bg-gray-300"
      >
        <div class="self-center">目錄載入中...</div>
      </div>
      <template v-else>
        <li
          class="border-b cursor-pointer"
          :class="{ 'bg-yellow-300': isCurrentToc(toc.href) }"
          v-for="(toc, index) in navigation.toc"
          :key="index"
          @click="jumpToChapter(toc.href)"
        >
          {{ index }} {{ toc.label }}
        </li>
      </template>
    </ul>
    <div class="book-wrapper w-3/4">
      <div class="w-full h-16 p-4 bg-gray-600 flex align-center justify-center">
        <div class="self-center" v-if="!bookAvailable">進度條載入中...</div>
        <div v-else class="w-full">
          <input
            class="w-full"
            type="range"
            step="1"
            min="0"
            max="100"
            @change="onProgressChange($event.target.value)"
            @input="onProgressChange($event.target.value)"
            :value="progress"
            :disabled="!bookAvailable"
            ref="progress"
          />
          <div class="text-center text-white">目前閱讀進度：{{ progress }}</div>
        </div>
      </div>
      <div class="flex">
        <div
          :class="{ invisible: !bookAvailable }"
          class="w-1/12 bg-gray-600 text-white flex align-center justify-center cursor-pointer"
          @click="prevPage"
        >
          <div class="self-center">《</div>
        </div>
        <div id="read">
          <!-- <div class="absolute top-0 left-0 z-50 w-full h-full flex">
          <div class="flex-none w-1/12" @click="prevPage"></div>
          <div class="flex-1"></div>
          <div class="flex-none w-1/12" @click="nextPage"></div>
        </div> -->
        </div>
        <div
          :class="{ invisible: !bookAvailable }"
          class="w-1/12 bg-gray-600 text-white flex align-center justify-center cursor-pointer"
          @click="nextPage"
        >
          <div class="self-center">》</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import EPub from 'epubjs'
const DOWNLOAD_URL = '/How_to_Transform_Your_Life.epub'

export default {
  data() {
    return {
      book: null,
      rendition: null,
      locations: null,
      navigation: null,
      // 圖書是否處於可用狀態
      bookAvailable: false,
      progress: 0,
      progressFromPrevNext: 0,
      currentTocHref: '',
    }
  },
  mounted() {
    this.$nextTick(async () => {
      await this.showEpub()
    })
  },
  methods: {
    isCurrentToc(href) {
      // console.log('chapter href', href)
      // console.log('currentTocHref', this.currentTocHref)
      return this.currentTocHref && href.includes(this.currentTocHref)
    },
    // 根據鏈結跳轉到指定位置
    jumpToChapter(href) {
      this.rendition.display(href)
    },
    prevPage() {
      // Rendition.prev
      if (this.rendition) {
        this.rendition.prev()
      }
    },
    nextPage() {
      // Rendition.next
      if (this.rendition) {
        this.rendition.next()
      }
    },
    // 電子書的解析和渲染
    async showEpub() {
      // 生成BOOK
      this.book = new EPub(DOWNLOAD_URL)
      // 生成Rendition
      // const bookWrapper = document.querySelector('.book-wrapper')
      this.rendition = this.book.renderTo('read', {
        // width: bookWrapper.innerWidth,
        // height: bookWrapper.innerHeight,
        width: '1000px',
        height: '800px',
      })
      this.$nuxt.$loading.start()
      // 通過Rendition.display渲染電子書
      await this.rendition.display()
      const currentLocation = await this.rendition.currentLocation()
      this.currentTocHref = currentLocation.start.href

      // 獲取 Locations 物件，要透過 epubjs的Hook
      await this.book.ready
      await this.book.locations.generate()
      this.locations = this.book.locations
      this.rendition.on('locationChanged', (newLocation) => {
        console.log('newLocation', newLocation)
        this.currentTocHref = newLocation.href
        this.progress = newLocation.percentage * 100
      })
      this.navigation = this.book.navigation
      console.log('navigation', this.navigation)
      this.bookAvailable = true
      this.$nuxt.$loading.finish()
    },
    onProgressChange(progress) {
      this.progress = progress
      const percentage = progress / 100
      const location =
        percentage > 0 ? this.locations.cfiFromPercentage(percentage) : 0
      this.currentTocHref = location.href
      this.rendition.display(location)
    },
  },
}
</script>

<style></style>
