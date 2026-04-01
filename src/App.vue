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

const MINI_LOGIN_PATH = '/pages-sub/login/index'

function miniProgramGo(url) {
  if (wx?.miniProgram?.redirectTo) {
    wx.miniProgram.redirectTo({ url })
    return true
  }
  if (wx?.miniProgram?.navigateTo) {
    wx.miniProgram.navigateTo({ url })
    return true
  }
  return false
}

/** 统一回登录页；有凭证则带上 dataToken，供登录页 onLoad 处理 */
function redirectToMiniProgramLogin(dataToken) {
  const url = dataToken
    ? `${MINI_LOGIN_PATH}?dataToken=${encodeURIComponent(dataToken)}`
    : MINI_LOGIN_PATH
  return miniProgramGo(url)
}

onMounted(async () => {
  try {
    await initWxSdk()
  } catch (error) {
    console.error(error)
  }

  token.value =
    queryParams.value.get('dataToken') || queryParams.value.get('token') || ''

  if (isInMiniProgram()) {
    if (redirectToMiniProgramLogin(token.value)) {
      status.value = '正在返回小程序登录页...'
      return
    }
    status.value = '跳转失败，请关闭页面后重试。'
    return
  }

  if (!token.value) {
    status.value = '未收到 dataToken。'
  } else {
    status.value = '已接收凭证。当前不在小程序环境，请手动返回小程序。'
  }
})
</script>

<template>
  <main class="container">
    <h1>电子社保卡登录中转页</h1>
    <p class="status">{{ status }}</p>
    <p v-if="token" class="token">dataToken: {{ token }}</p>
  </main>
</template>
