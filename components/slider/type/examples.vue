<template>
  <div class="examples">
    <div class="max-h-full">
      <p v-if="data.description" v-html="data.description" />
      <template v-if="data.images && data.images.length">
        <img
          v-for="(image, imageIndex) in data.images"
          :key="imageIndex"
          :src="image.href"
          :alt="`Presentation Image ${imageIndex}`"
          :class="[data.imageClass]"
        />
      </template>
      <nuxt-link :to="data.link" target="_blank">Show example</nuxt-link>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Vue, Prop } from 'nuxt-property-decorator'
import { PresentationInterface } from '~/interfaces/PresentationInterface'

@Component({
  layout: 'presentation',
})
export default class SliderTypeExamples extends Vue {
  @Prop({ required: true })
  data!: PresentationInterface
}
</script>

<style lang="scss" scoped>
.examples {
  @apply flex items-center justify-center text-center h-full;
  p {
    @apply text-base lg:text-2xl;
    @apply max-w-[90%] md:max-w-[75%] m-auto mb-8;
    ::v-deep {
      code {
        @apply text-blue-600;
        &:before,
        &:after {
          content: '`';
        }
      }
    }
  }
  a {
    @apply inline-block transition duration-200 cursor-pointer;
    @apply py-2 lg:py-3 px-3 lg:px-4 text-sm lg:text-base bg-blue-500 font-medium rounded shadow-md hover:bg-blue-600 focus:outline-none;
    @apply text-primary-50;
    @apply dark:text-primary-900;
  }
  img {
    @apply block mb-8;
  }
}
</style>
