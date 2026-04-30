<script setup lang="ts">
import { computed, nextTick, ref, watch } from 'vue'
import type { ComponentPublicInstance } from 'vue'

interface Props {
  length?: number,
  columns?: number,
  error?: boolean,
  errorMessage?: string,
  disabled?: boolean,
}

const props = withDefaults(defineProps<Props>(), {
  length: 6,
  columns: 0,
  error: false,
  errorMessage: '',
  disabled: false
})

// error 為 true 或 errorMessage 有值時顯示錯誤樣式
const hasError = computed(() => props.error || !!props.errorMessage)

const model = defineModel<string>({ default: '' })

// 限制長度範圍 4-8，預設 6
const otpLength = computed(() => Math.min(8, Math.max(4, props.length)))

// 每列幾格，0 或未傳 → 同 length（單列）；上下限介於 1 ~ length
const columnCount = computed(() => {
  const target = props.columns > 0 ? props.columns : otpLength.value
  return Math.min(otpLength.value, Math.max(1, target))
})

const digits = ref<string[]>(Array.from({ length: otpLength.value }, (): string => ''))
const inputRefs = ref<HTMLInputElement[]>([])

// 將外部傳入的 model 同步到 digits
const syncFromModel = (value: string) => {
  const chars = (value || '').replace(/\D/g, '').slice(0, otpLength.value).split('')
  digits.value = Array.from({ length: otpLength.value }, (_: unknown, i: number): string => chars[i] || '')
}

watch(() => model.value, (val: string) => {
  if (val !== digits.value.join('')) {
    syncFromModel(val)
  }
}, { immediate: true })

// 當 length 改變時重置
watch(otpLength, () => {
  digits.value = Array.from({ length: otpLength.value }, (): string => '')
  model.value = ''
})

const focusInput = (index: number) => {
  const target = inputRefs.value[index]
  if (target) {
    target.focus()
    target.select()
  }
}

const syncToModel = () => {
  model.value = digits.value.join('')
}

const onInput = (event: Event, index: number) => {
  const target = event.target as HTMLInputElement
  const clean = target.value.replace(/\D/g, '')

  if (!clean) {
    digits.value[index] = ''
    target.value = ''
    syncToModel()
    return
  }

  if (clean.length === 1) {
    digits.value[index] = clean
    target.value = clean
    if (index < otpLength.value - 1) {
      nextTick(() => focusInput(index + 1))
    }
    syncToModel()
    return
  }

  // 若同時收到多個字元（瀏覽器自動補完等），分散填入後續欄位
  const chars = clean.split('').slice(0, otpLength.value - index)
  chars.forEach((c: string, i: number) => {
    digits.value[index + i] = c
  })
  target.value = chars[0]

  const nextIndex = Math.min(index + chars.length, otpLength.value - 1)
  nextTick(() => focusInput(nextIndex))
  syncToModel()
}

const onKeydown = (event: KeyboardEvent, index: number) => {
  if (event.key === 'Backspace') {
    if (digits.value[index]) {
      digits.value[index] = ''
      syncToModel()
      return
    }
    if (index > 0) {
      event.preventDefault()
      digits.value[index - 1] = ''
      focusInput(index - 1)
      syncToModel()
    }
    return
  }

  if (event.key === 'ArrowLeft' && index > 0) {
    event.preventDefault()
    focusInput(index - 1)
    return
  }

  if (event.key === 'ArrowRight' && index < otpLength.value - 1) {
    event.preventDefault()
    focusInput(index + 1)
  }
}

const onPaste = (event: ClipboardEvent, index: number) => {
  event.preventDefault()
  const text = event.clipboardData?.getData('text') ?? ''
  const clean = text.replace(/\D/g, '')
  if (!clean) {
    return
  }

  // 貼上完整碼時無視當前格，從第一格起覆蓋
  const startIndex = clean.length >= otpLength.value ? 0 : index
  const chars = clean.slice(0, otpLength.value - startIndex).split('')

  chars.forEach((c: string, i: number) => {
    digits.value[startIndex + i] = c
  })

  const nextIndex = Math.min(startIndex + chars.length, otpLength.value - 1)
  nextTick(() => focusInput(nextIndex))
  syncToModel()
}

const onFocus = (event: FocusEvent) => {
  ;(event.target as HTMLInputElement).select()
}

const setInputRef = (el: Element | ComponentPublicInstance | null, index: number): void => {
  if (el instanceof HTMLInputElement) {
    inputRefs.value[index] = el
  }
}
</script>

<template>
  <div class="inline-flex flex-col items-center">
    <div
      class="grid items-center gap-2"
      :style="{ gridTemplateColumns: `repeat(${columnCount}, minmax(0, max-content))` }"
    >
      <input
        v-for="(digit, index) in digits"
        :key="index"
        :ref="(el) => setInputRef(el, index)"
        :value="digit"
        type="text"
        inputmode="numeric"
        autocomplete="one-time-code"
        maxlength="1"
        :disabled="props.disabled"
        class="size-12 rounded-md border bg-white text-center text-xl font-bold text-black shadow-sm outline-0 transition-colors"
        :class="[
          hasError ? 'border-red-500' : 'border-gray-300',
          props.disabled && 'cursor-not-allowed bg-gray-100 opacity-60'
        ]"
        @input="onInput($event, index)"
        @keydown="onKeydown($event, index)"
        @paste="onPaste($event, index)"
        @focus="onFocus"
      >
    </div>
    <p
      v-if="props.errorMessage"
      class="mt-2 text-center text-sm text-red-500"
    >
      {{ props.errorMessage }}
    </p>
  </div>
</template>
