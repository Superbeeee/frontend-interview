<script setup lang="ts">
import { ref, watch } from 'vue'

const otp = ref('')
const errorMessage = ref('')
const isVerifying = ref(false)
const isVerified = ref(false)

const OTP_LENGTH = 6
const columns = ref<number>(OTP_LENGTH)

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

// 按下 Submit 時：未填滿提示錯誤，已填滿手動觸發驗證
const onSubmit = () => {
  if (otp.value.length < OTP_LENGTH) {
    errorMessage.value = 'Please enter the complete code'
    return
  }
  verify(otp.value)
}

// 透過 v-model 監聽 otp 變化：填滿即時驗證，未填滿時清掉錯誤與成功狀態
watch(otp, (val: string) => {
  errorMessage.value = ''
  if (val.length < OTP_LENGTH) {
    isVerified.value = false
    return
  }
  if (!isVerifying.value && !isVerified.value) {
    verify(val)
  }
})
</script>

<template>
  <div class="flex min-h-screen items-center justify-center bg-gray-100 p-12">
    <div class="flex flex-col items-center">
      <label class="mb-3 flex items-center gap-2 text-sm text-gray-700">
        Columns per row
        <input
          v-model.number="columns"
          type="number"
          min="1"
          :max="OTP_LENGTH"
          class="w-16 rounded-md border border-gray-300 bg-white px-2 py-1 text-center outline-0 focus:border-black"
        >
      </label>
      <BaseInputOtp
        v-model="otp"
        :length="OTP_LENGTH"
        :columns="columns"
        :error-message="errorMessage"
        :disabled="isVerifying || isVerified"
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
