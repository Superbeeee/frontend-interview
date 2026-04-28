<script setup lang="ts">
import { ref, watch } from 'vue'

const otp = ref('')
const errorMessage = ref('')
const isVerifying = ref(false)
const isVerified = ref(false)

const verify = async (code: string) => {
  isVerifying.value = true
  errorMessage.value = ''

  try {
    const response = await useApi().example.verifyOtp(code)
    if (response.success && response.data.verified) {
      isVerified.value = true
      return
    }
    errorMessage.value = 'Invalid code'
  } catch (error) {
    errorMessage.value = 'Verification failed. Please try again.'
    console.log(error)
  } finally {
    isVerifying.value = false
  }
}

// 輸入完成即時驗證
const onComplete = (code: string) => {
  verify(code)
}

// 按下 Submit 時：未填滿提示錯誤，已填滿手動觸發驗證
const onSubmit = () => {
  if (otp.value.length < 6) {
    errorMessage.value = 'Please enter the complete code'
    return
  }
  verify(otp.value)
}

// 使用者重新編輯時清掉錯誤訊息與成功狀態
watch(otp, (val: string) => {
  if (val.length < 6) {
    isVerified.value = false
  }
  errorMessage.value = ''
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
        :disabled="isVerifying || isVerified"
        @click="onSubmit"
      >
        {{ isVerifying ? 'Verifying...' : isVerified ? 'Verified' : 'Submit' }}
      </button>
    </div>
  </div>
</template>
