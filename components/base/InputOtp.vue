<script setup lang="ts">
import { computed, nextTick, ref, watch } from 'vue'
import type { ComponentPublicInstance } from 'vue'

interface Props {
  length?: number,
  errorMessage?: string,
  disabled?: boolean,
}

const props = withDefaults(defineProps<Props>(), {
  length: 6,
  errorMessage: '',
  disabled: false
})

const emit = defineEmits<{
  complete: [value: string]
}>()

const model = defineModel<string>({ default: '' })

// 限制長度範圍 4-8，預設 6
const otpLength = computed(() => Math.min(8, Math.max(4, props.length)))

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
  const value = digits.value.join('')
  model.value = value
  if (value.length === otpLength.value && digits.value.every((d: string) => /^\d$/.test(d))) {
    emit('complete', value)
  }
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
  <div>
    <div class="flex items-center gap-2">
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
        class="size-12 rounded-md border-2 bg-white text-center text-lg font-medium text-black outline-0 transition-colors"
        :class="[
          props.errorMessage ? 'border-red-500' : 'border-black',
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
      class="mt-2 text-sm text-red-500"
    >
      {{ props.errorMessage }}
    </p>
  </div>
</template>
