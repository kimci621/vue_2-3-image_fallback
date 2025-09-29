<template>
  <NuxtImg
    :key="imageKey"
    :src="currentSrc"
    :alt="alt"
    :width="w"
    :height="h"
    :loading="loading"
    @error="handleError"
  />
</template>

<script setup>
import { defineProps, computed, ref, watch } from '@nuxtjs/composition-api'
import { makeResizedImageUrl } from '~/helpers/image'

const props = defineProps({
  alt: {
    type: String,
    required: true,
    default: '',
  },
  src: {
    type: String,
    required: true,
    default: '',
  },
  w: {
    type: Number,
    required: true,
    default: 0,
  },
  h: {
    type: Number,
    required: true,
    default: 0,
  },
  ext: {
    type: String,
    required: false,
    default: 'webp',
  },
  sizes: {
    type: Array,
    default: () => [],
    required: false,
  },
  loading: {
    type: String,
    default: 'lazy',
    required: false,
  },
})

// Кэш для проверенных URL'ов
const urlCache = new Map()

const srcWithExt = computed(() => {
  const indexOfLastDot = props.src.lastIndexOf('.')
  const baseUrl = props.src.slice(0, indexOfLastDot)
  const extension = props.ext
    ? `.${props.ext}`
    : props.src.slice(indexOfLastDot)
  return `${baseUrl}${extension}`
})

// Генерируем все возможные URL'ы для попыток
const fallbackUrls = computed(() => {
  const urls = []

  urls.push(
    makeResizedImageUrl({
      src: srcWithExt.value,
      w: props.w,
      h: props.h,
    })
  )

  if (props.sizes.length > 0) {
    props.sizes.forEach((size) => {
      const extToUse = size.ext || props.ext || 'webp'
      const baseUrl = props.src.substring(0, props.src.lastIndexOf('.'))
      const urlWithExt = `${baseUrl}.${extToUse}`

      urls.push(
        makeResizedImageUrl({
          src: urlWithExt,
          w: size.w || props.w,
          h: size.h || props.h,
        })
      )
    })
  }

  urls.push(srcWithExt.value)

  return [...new Set(urls)]
})

const currentIndex = ref(0)
const imageKey = ref(0)
const currentSrc = ref('')

const initializeImage = () => {
  currentIndex.value = 0
  currentSrc.value = fallbackUrls.value[0] || ''
  imageKey.value++
}

// Обработчик ошибки загрузки
const handleError = (e) => {
  const failedUrl = currentSrc.value

  // Кэшируем неработающий URL
  urlCache.set(failedUrl, false)

  console.warn(`ImgFallbackResizer: ${failedUrl}`)

  // Переходим к следующему URL
  currentIndex.value++

  if (currentIndex.value >= fallbackUrls.value.length) {
    console.error('ImgFallbackResizer: All urls failed:', fallbackUrls.value)
    return
  }

  const nextUrl = fallbackUrls.value[currentIndex.value]

  if (urlCache.has(nextUrl) && !urlCache.get(nextUrl)) {
    // Этот URL уже проверен и не работает, пропускаем
    handleError(e)
    return
  }

  currentSrc.value = nextUrl
  imageKey.value++ // Принудительно перерендерим img

  console.info(`ImgFallbackResizer: Trying to load: ${nextUrl}`)
}

// Опциональная предварительная проверка URL'ов
// const preloadUrls = async () => {
//   const promises = fallbackUrls.value.map((url) => {
//     return new Promise((resolve) => {
//       const img = new Image()
//       img.onload = () => {
//         urlCache.set(url, true)
//         resolve({ url, success: true })
//       }
//       img.onerror = () => {
//         urlCache.set(url, false)
//         resolve({ url, success: false })
//       }
//       img.src = url
//     })
//   })

//   const results = await Promise.all(promises)
//   const firstWorking = results.find((r) => r.success)

//   if (firstWorking && firstWorking.url !== currentSrc.value) {
//     currentSrc.value = firstWorking.url
//     currentIndex.value = fallbackUrls.value.indexOf(firstWorking.url)
//     imageKey.value++
//   }
// }

// Инициализация при изменении props
watch(
  [() => props.src, () => props.ext, () => props.w, () => props.h],
  initializeImage,
  { immediate: true }
)

// Раскомментируй, если нужна предварительная проверка
// watch(fallbackUrls, preloadUrls, { immediate: true })
</script>


// helpers/image
/**
 * Возвращает URL изображения с добавленными размерами (ширина x высота) перед расширением файла.
 * @param {string} src - Исходный путь к изображению
 * @param {number} s - Требуемая ширина изображения
 * @param {number} h - Требуемая высота изображения
 * @param {number} dens - Плотность изображения
 * @returns {string} Новый путь к изображению с размерами
 */
export const makeResizedImageUrl = ({ src, w, h, dens, ext }) => {
  const indexOfLastDot = src.lastIndexOf('.')
  const baseUrl = src.slice(0, indexOfLastDot)
  const sizes = w && h ? `__${w}x${h}` : ''
  const extension = ext ? `.${ext}` : src.slice(indexOfLastDot, src.length)
  const densityString = dens ? ` ${dens}x` : ''

  return `${baseUrl}${sizes}${extension}${densityString}`
}

/**
 * Возвращает массив URL изображений с размерами (ширина x высота) перед расширением файла.
 * @param {string} src - Исходный путь к изображению
 * @param {{w: number, h: number, dens?: number, ext?: string}[]} sizes - Массив объектов с размерами изображений
 * @returns {Array} Массив URL изображений с размерами
 */
export const makeResizedSrcSet = (src, sizes) => {
  return sizes
    .map((size) => {
      return makeResizedImageUrl({
        src,
        ...size,
      })
    })
    .join(', ')
}

