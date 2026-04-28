<script setup lang="ts">
import { ref, watch } from 'vue'

const otp = ref('')
const errorMessage = ref('')
const isVerifying = ref(false)
const isVerified = ref(false)

const onComplete = async (code: string) => {
  isVerifying.value = true
  errorMessage.value = ''
  isVerified.value = false

  try {
    const response = await useApi().example.verifyOtp(code)
    if (response.success && response.data.verified) {
      isVerified.value = true
      return
    }
    errorMessage.value = '驗證碼錯誤，請重新輸入'
  } catch (error) {
    errorMessage.value = '驗證失敗，請稍後再試'
    console.log(error)
  } finally {
    isVerifying.value = false
  }
}

// 使用者重新輸入時清掉錯誤訊息與成功狀態
watch(otp, (val: string) => {
  if (val.length < 6) {
    errorMessage.value = ''
    isVerified.value = false
  }
})
</script>

<template>
  <div class="flex min-h-screen items-center justify-center p-12">
    <div class="w-full max-w-md">
      <h1 class="mb-2 text-center text-2xl font-bold">
        輸入驗證碼
      </h1>
      <p class="mb-6 text-center text-sm text-gray-500">
        請輸入 6 位數驗證碼（測試用：123456）
      </p>
      <div class="flex justify-center">
        <BaseInputOtp
          v-model="otp"
          :length="6"
          :error-message="errorMessage"
          :disabled="isVerifying || isVerified"
          @complete="onComplete"
        />
      </div>
      <div class="mt-4 text-center text-sm">
        <p
          v-if="isVerifying"
          class="text-gray-500"
        >
          驗證中...
        </p>
        <p
          v-else-if="isVerified"
          class="text-green-600"
        >
          驗證成功
        </p>
      </div>
    </div>
  </div>
</template>
