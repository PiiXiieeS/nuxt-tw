<template>
  <div class="presentation">
    <div class="presentation__wrapper" :class="{ fullscreen: fullscreenMode }">
      <div class="presentation__navigation">
        <button
          :class="{
            hidden: !showButtonPrev,
          }"
          class="presentation__navigation-prev"
          type="button"
          @click="onScroll('prev')"
        >
          <i class="material-icons">navigate_before</i>
        </button>
        <ul ref="presentationScrollNavigation">
          <li
            v-for="slide in slides"
            :id="isActive(slide.slug) ? 'tabElement' : ''"
            :key="slide.slug"
            :tabindex="slide.slug"
            :class="{
              active: isActive(slide.slug),
            }"
            @click="goToSlide(slide.slug)"
            @keypress.enter="goToSlide(slide.slug)"
          >
            {{ slide.navigationTitle }}
          </li>
        </ul>
        <button
          class="presentation__navigation-next"
          type="button"
          :class="{
            hidden: !showButtonNext,
          }"
          @click="onScroll('next')"
        >
          <i class="material-icons">navigate_next</i>
        </button>
      </div>
      <div class="presentation__container">
        <div v-if="formatTime" class="countdown">
          <i class="material-icons-outlined" @click="resetCountdown(true)"
            >restart_alt</i
          >
          <span v-text="formatTime" />
        </div>
        <nuxt-child />
        <div class="watermark">
          <img src="/images/logo-blue.svg" alt="Logo" />
        </div>
        <div
          class="presentation__pagination"
          :class="{ fullscreen: fullscreenMode }"
        >
          <button class="fullscreen" type="button" @click="toggleFullScreen">
            <span
              class="material-icons-outlined"
              v-text="fullscreenMode ? 'fullscreen_exit' : 'fullscreen'"
            />
          </button>
          <button
            :disabled="currentSlide <= 1"
            type="button"
            @click="slidePrev"
          >
            <span class="material-icons-outlined">skip_previous</span>
          </button>
          <span
            class="text-sm opacity-50 pointer-events-none"
            v-text="returnSliderRange"
          />
          <button
            type="button"
            :disabled="currentSlide >= totalSlides"
            @click="slideNext"
          >
            <span class="material-icons-outlined">skip_next</span>
          </button>
        </div>
        <div class="progress">
          <div
            class="progress__bar"
            :style="{ width: `${returnSliderProgress}%` }"
          />
        </div>
        <div class="back">
          <button type="button" @click="$router.push('/')">
            <i class="material-icons">home</i>
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Vue, Watch } from 'nuxt-property-decorator'
import { IContentDocument } from '@nuxt/content/types/content'
import { PresentationInterface } from '~/interfaces/PresentationInterface'

@Component({
  layout: 'presentation',
})
export default class PresentationBase extends Vue {
  @Watch('initData')
  onInitData(val: boolean) {
    if (val) {
      this.init()
    }
  }

  @Watch('countdown')
  onWatchCountdown(val: number) {
    if (!val) {
      this.$router.push('/thank-you')
    }
  }

  // Data
  slides: PresentationInterface[] | IContentDocument[] = []
  scrollOffsetValue: number = 0
  showButtonPrev: boolean = false
  showButtonNext: boolean = true
  fullscreenMode: boolean = false
  countdown: number | null = null
  countdownInterval: any = null
  initData: boolean = false

  // Hooks
  validate({ params, redirect }: { params: { id: string }; redirect: any }) {
    // If param doesnt exist in route redirect to first slide
    if (!params.id) {
      return redirect('/presentation/slide/1')
    }
    return true
  }

  async fetch() {
    this.slides = await this.$content('slides')
      .sortBy('order', 'asc')
      .fetch<any>()
    this.initData = true
  }

  mounted() {
    document.addEventListener('fullscreenchange', this.onToggleFullscreenMode)
    window.addEventListener('keydown', this.onKeyboardNavigation)
  }

  beforeDestroy() {
    document.removeEventListener(
      'fullscreenchange',
      this.onToggleFullscreenMode
    )
    window.removeEventListener('keydown', this.onKeyboardNavigation)
  }

  // Getters
  get returnSliderRange(): string {
    return `${this.currentSlide}/${this.totalSlides}`
  }

  get currentSlide(): number {
    return +this.$route.params.id
  }

  get totalSlides(): number {
    return this.slides.length
  }

  get returnSliderProgress(): number {
    return (this.currentSlide / this.totalSlides) * 100
  }

  get formatTime(): boolean | string {
    if (this.countdown === null) {
      return false
    }
    const duration = this.countdown
    // Hours, minutes and seconds
    const hrs = ~~(duration / 3600)
    const mins = ~~((duration % 3600) / 60)
    const secs = ~~duration % 60
    // Output like "1:01" or "4:03:59" or "123:03:59"
    let ret = ''
    if (hrs > 0) {
      ret += '' + hrs + ':' + (mins < 10 ? '0' : '')
    }
    ret += '' + mins + ':' + (secs < 10 ? '0' : '')
    ret += '' + secs
    return ret
  }

  // Methods
  init() {
    const { start } = this.$route.query
    if (start) {
      this.resetCountdown(true)
      return
    }
    const savedCountdown = window.localStorage.getItem('countdown')
    if (savedCountdown) {
      this.countdown = +savedCountdown
    } else if (this.countdownInterval === null) {
      const timeInSeconds = prompt(
        'How much time you need for presentation? (seconds) // By default smart slide timer will be user'
      )
      if (timeInSeconds) {
        this.countdown = +timeInSeconds
      } else {
        const smartSliderTimer = (this.slides as PresentationInterface[])
          .map((obj) => obj.time)
          .reduce((a, b) => a + b, 0)
        this.countdown = smartSliderTimer || 60
      }
    }
    this.onCountdown()
    this.$nextTick(() => {
      setTimeout(() => {
        this.onAutoScrollDetect('next') // Autoscroll on page refresh if we are on different slide
      }, 250)
    })
  }

  onToggleFullscreenMode() {
    this.fullscreenMode = !!document.fullscreenElement
  }

  onKeyboardNavigation($e: KeyboardEvent) {
    const { key } = $e
    switch (key) {
      case 'ArrowRight':
        this.slideNext()
        break
      case 'ArrowLeft':
        this.slidePrev()
        break
    }
  }

  onCountdown() {
    this.countdownInterval = setInterval(() => {
      if (typeof this.countdown === 'number') {
        this.countdown--
      }
      if (!this.countdown) {
        this.resetCountdown(false)
      } else {
        window.localStorage.setItem('countdown', this.countdown!.toString())
      }
    }, 1000)
  }

  resetCountdown(reload: boolean = false) {
    if (!reload) {
      this.countdown = null
    }
    window.localStorage.removeItem('countdown')
    window.localStorage.removeItem('slide')
    clearInterval(this.countdownInterval)
    if (reload) {
      window.location.href = `${window.location.origin}${window.location.pathname}`
    }
  }

  onScroll(scrollType: 'prev' | 'next', nextValue = 0) {
    const element = this.$refs.presentationScrollNavigation as HTMLElement
    const elWidth = element.offsetWidth
    let leftScrollOffset = elWidth - 150
    if (scrollType === 'prev') {
      leftScrollOffset = -leftScrollOffset
    }
    if (nextValue) {
      leftScrollOffset += nextValue
    }
    this.scrollOffsetValue += leftScrollOffset
    if (this.scrollOffsetValue > elWidth) {
      this.scrollOffsetValue = elWidth
    } else if (this.scrollOffsetValue < 0) {
      this.scrollOffsetValue = 0
    }
    element.scrollBy({
      left: leftScrollOffset,
      behavior: 'smooth',
    })
    this.showButtonPrev = this.scrollOffsetValue >= 1
    this.showButtonNext = this.scrollOffsetValue !== elWidth
  }

  goToSlide(slide: string) {
    this.$router.push({ path: `/presentation/slide/${slide}` })
  }

  slidePrev() {
    let currentSlide = this.currentSlide
    if (currentSlide > 1) {
      currentSlide--
      this.goToSlide(currentSlide.toString())
    }
    this.onAutoScrollDetect('prev')
  }

  slideNext() {
    let currentSlide = this.currentSlide
    if (currentSlide < this.totalSlides) {
      currentSlide++
      this.goToSlide(currentSlide.toString())
    }
    this.onAutoScrollDetect('next')
  }

  onAutoScrollDetect(type: 'prev' | 'next') {
    const element = this.$refs.presentationScrollNavigation as HTMLElement
    const elWidth = element.offsetWidth
    let elementOffset = document.getElementById('tabElement')?.offsetLeft || 0
    const elementOffsetWidth =
      document.getElementById('tabElement')?.offsetWidth || 0
    elementOffset += elementOffsetWidth
    if (type === 'prev') {
      if (elementOffset < elWidth) {
        this.onScroll(type)
      }
    } else if (type === 'next') {
      if (elementOffset > elWidth) {
        this.onScroll(type, elementOffsetWidth)
      }
    }
  }

  toggleFullScreen() {
    if (!document.fullscreenElement) {
      document.documentElement.requestFullscreen()
    } else if (document.exitFullscreen) {
      document.exitFullscreen()
    }
  }

  isActive(id: string): boolean {
    return id === this.$route.params.id
  }
}
</script>

<style lang="scss" scoped>
.presentation {
  @apply h-full w-full;
  @apply flex items-center justify-center;
  @apply bg-gray-200;
  @apply dark:bg-gray-900;
  &__wrapper {
    @apply overflow-hidden;
    @apply w-full h-full max-w-[1280px] max-h-full;
    @apply lg:w-[85%] lg:h-[80%] lg:max-h-[768px];
    @apply bg-gray-100;
    @apply dark:bg-gray-800;
    &.fullscreen {
      @apply md:w-full md:h-full md:max-w-full md:max-h-full;
    }
  }
  &__navigation {
    @apply relative;
    @apply bg-gray-100;
    @apply dark:bg-gray-800;
    @apply flex items-center;
    &:after {
      content: '';
      @apply absolute top-0 right-0 bottom-0 left-0;
      @apply opacity-90;
      @apply bg-gradient-to-r from-transparent to-gray-200;
      @apply dark:to-gray-900;
      @apply pointer-events-none;
    }
    ul {
      scroll-behavior: smooth;
      width: calc(100% - 5rem);
      @apply h-10;
      @apply flex items-center;
      @apply overflow-auto;
      &::-webkit-scrollbar {
        -ms-overflow-style: none; /* IE and Edge */
        scrollbar-width: none; /* Firefox */
        @apply hidden;
      }
      li {
        white-space: nowrap;
        @apply ml-2 cursor-pointer text-sm;
        @apply flex items-center;
        @apply transition duration-200;
        @apply focus:outline-none focus:text-blue-500;
        @apply hover:text-green-600;
        &:not(:last-child):after {
          font-family: 'Material Icons';
          content: 'arrow_forward';
          @apply ml-1.5;
          @apply text-primary-900;
          @apply dark:text-primary-50;
        }
        &.active {
          @apply text-green-600;
        }
      }
    }
    &-prev,
    &-next {
      @apply relative z-10;
      @apply flex items-center justify-center;
      @apply transition duration-200;
      @apply focus:outline-none focus:text-blue-400;
      @apply w-10 h-10;
      @apply bg-gray-100;
      @apply dark:bg-gray-800;
      @apply hover:text-green-600;
      &.hidden {
        @apply opacity-5 cursor-not-allowed;
      }
    }
    &-next {
      @apply ml-2;
    }
  }
  &__container {
    height: calc(100% - 2.5rem);
    @apply relative pb-16;
  }
  &__pagination {
    @apply absolute bottom-4 right-14 z-10;
    @apply lg:right-4;
    @apply flex items-center;
    &.fullscreen {
      @apply right-16;
    }
    button {
      @apply flex items-center justify-center mx-1;
      @apply focus:outline-none focus:text-blue-400;
      @apply disabled:opacity-50 disabled:cursor-not-allowed;
      &:not(:disabled) {
        @apply hover:text-green-600;
      }
      &.fullscreen {
        @apply hidden;
        @apply md:flex;
      }
    }
  }
}
.countdown {
  @apply absolute top-2 left-4 z-10;
  @apply text-xs font-medium;
  @apply flex items-center;
  span {
    @apply pointer-events-none;
  }
  i {
    @apply text-base mr-1 cursor-pointer hover:text-green-600;
    line-height: 0;
  }
}
.watermark {
  @apply absolute bottom-4 left-4 z-10;
  @apply flex items-center;
  img {
    @apply block w-8;
  }
  span {
    @apply ml-2 font-medium;
  }
}
.progress {
  @apply absolute bottom-0 left-0 right-0;
  @apply h-0.5;
  &__bar {
    transition: width 0.2s linear;
    @apply bg-blue-500;
    @apply h-full w-0;
  }
}
.back {
  @apply absolute top-0 right-0 z-50;
  button {
    @apply flex items-center justify-center focus:outline-none cursor-pointer transition duration-200;
    @apply text-blue-500 hover:bg-blue-100 focus:bg-blue-200;
    @apply w-8 h-8;
    i {
      @apply text-2xl;
    }
  }
}
</style>
