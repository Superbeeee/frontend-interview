<script setup lang="ts">
import { computed, ref, watch } from 'vue'

const otp = ref('')
const errorMessage = ref('')
const isVerifying = ref(false)
const isVerified = ref(false)

const isComplete = ref(false)
const canSubmit = computed(() => isComplete.value && !isVerifying.value && !isVerified.value)

const onComplete = () => {
  isComplete.value = true
}

const verify = async () => {
  if (!canSubmit.value) {
    return
  }
  isVerifying.value = true
  errorMessage.value = ''

  try {
    const response = await useApi().example.verifyOtp(otp.value)
    if (response.success && response.data.verified) {
      isVerified.value = true
      return
    }
    errorMessage.value = 'Invalid code'
  } catch (error) {
    errorMessage.value = '驗證失敗，請稍後再試'
    console.log(error)
  } finally {
    isVerifying.value = false
  }
}

// 使用者重新編輯時清掉錯誤訊息與成功狀態
watch(otp, (val: string) => {
  if (val.length < 6) {
    isComplete.value = false
    isVerified.value = false
    errorMessage.value = ''
  }
})
</script>

<template>
  <div class="flex min-h-screen items-center justify-center bg-gray-100 p-12">
    <div class="flex flex-col items-center">
      <BaseInputOtp
        v-model="otp"
        :length="6"
        :error-message="errorMessage"
        :disabled="isVerifying || isVerified"
        @complete="onComplete"
      />
      <button
        type="button"
        class="mt-4 rounded-md border border-gray-300 bg-white px-6 py-2 text-sm font-medium text-black shadow-sm transition-colors hover:bg-gray-50 disabled:cursor-not-allowed disabled:opacity-60"
        :disabled="!canSubmit"
        @click="verify"
      >
        {{ isVerifying ? '驗證中...' : isVerified ? '驗證成功' : 'Submit' }}
      </button>
    </div>
  </div>
</template>
