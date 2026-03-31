<script setup>
import { computed, onMounted, ref } from 'vue'
import wx from 'weixin-js-sdk'

const status = ref('正在处理授权结果...')
const token = ref('')

const queryParams = computed(() => {
  const map = new URLSearchParams(window.location.search)
  const hash = window.location.hash
  if (hash.includes('?')) {
    const hashQuery = hash.slice(hash.indexOf('?') + 1)
    new URLSearchParams(hashQuery).forEach((value, key) => {
      if (!map.has(key)) {
        map.set(key, value)
      }
    })
  }
  return map
})

function isWeChatBrowser() {
  return /micromessenger/i.test(window.navigator.userAgent)
}

function isInMiniProgram() {
  return window.__wxjs_environment === 'miniprogram'
}

async function initWxSdk() {
  const appId = import.meta.env.VITE_WX_APPID
  const signatureApi = import.meta.env.VITE_WX_SIGNATURE_API
  if (!appId || !signatureApi || !isWeChatBrowser()) {
    return
  }

  const resp = await fetch(
    `${signatureApi}?url=${encodeURIComponent(window.location.href.split('#')[0])}`,
  )
  if (!resp.ok) {
    throw new Error('获取微信签名失败')
  }
  const data = await resp.json()

  wx.config({
    debug: false,
    appId,
    timestamp: data.timestamp,
    nonceStr: data.nonceStr,
    signature: data.signature,
    jsApiList: ['checkJsApi', 'miniProgram.navigateTo', 'miniProgram.redirectTo'],
  })
}

function redirectToMiniProgram(dataToken) {
  const redirectPath = queryParams.value.get('redirectPath') || '/pages/index/index'
  const connector = redirectPath.includes('?') ? '&' : '?'
  const url = `${redirectPath}${connector}dataToken=${encodeURIComponent(dataToken)}`

  if (wx?.miniProgram?.redirectTo) {
    wx.miniProgram.redirectTo({ url })
    status.value = '授权成功，正在返回小程序...'
    return true
  }

  if (wx?.miniProgram?.navigateTo) {
    wx.miniProgram.navigateTo({ url })
    status.value = '授权成功，正在返回小程序...'
    return true
  }

  return false
}

onMounted(async () => {
  try {
    await initWxSdk()
  } catch (error) {
    console.error(error)
  }

  token.value = queryParams.value.get('dataToken') || ''
  if (!token.value) {
    status.value = '未收到 dataToken，请检查回调地址参数。'
    return
  }

  if (isInMiniProgram() && redirectToMiniProgram(token.value)) {
    return
  }

  status.value = '已接收 dataToken。当前不在小程序环境，请手动返回。'
})
</script>

<template>
  <main class="container">
    <h1>电子社保卡登录中转页</h1>
    <p class="status">{{ status }}</p>
    <p v-if="token" class="token">dataToken: {{ token }}</p>
  </main>
</template>
