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
    jsApiList: [
      'checkJsApi',
      'miniProgram.navigateTo',
      'miniProgram.redirectTo',
      'miniProgram.reLaunch',
      'miniProgram.switchTab',
    ],
  })
}

/** 小程序首页（与 xa_card_mini pages.json tabBar 一致） */
const MINI_HOME_PATH = '/pages/index/index'
const MINI_LOGIN_PATH = '/pages-sub/login/index'

/** tabBar 页面不可使用 navigateTo/redirectTo，需 reLaunch（可带 query）或 switchTab（不可带参） */
const TAB_BAR_PATHS = [
  '/pages/index/index',
  '/pages/serve/index',
  '/pages/shop/index',
  '/pages/mine/index',
]

function normalizeMiniPath(path) {
  if (!path) {
    return ''
  }
  const pathOnly = path.split('?')[0]
  return pathOnly.startsWith('/') ? pathOnly : `/${pathOnly}`
}

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

function redirectToMiniProgramHome(dataToken) {
  const raw = queryParams.value.get('redirectPath') || MINI_HOME_PATH
  const base = normalizeMiniPath(raw) || MINI_HOME_PATH
  const queryInPath = raw.includes('?') ? raw.slice(raw.indexOf('?')) : ''
  const pathWithQuery = queryInPath ? `${base}${queryInPath}` : base
  const connector = pathWithQuery.includes('?') ? '&' : '?'
  const url = `${pathWithQuery}${connector}dataToken=${encodeURIComponent(dataToken)}`

  if (TAB_BAR_PATHS.includes(base)) {
    if (wx?.miniProgram?.reLaunch) {
      wx.miniProgram.reLaunch({ url })
      status.value = '授权成功，正在返回小程序...'
      return true
    }
    if (wx?.miniProgram?.switchTab) {
      wx.miniProgram.switchTab({ url: base })
      status.value = '授权成功，正在返回小程序（当前环境无法携带 dataToken，请从首页重新发起授权）。'
      return true
    }
  }

  if (miniProgramGo(url)) {
    status.value = '授权成功，正在返回小程序...'
    return true
  }
  return false
}

function redirectToMiniProgramLogin() {
  return miniProgramGo(MINI_LOGIN_PATH)
}

onMounted(async () => {
  try {
    await initWxSdk()
  } catch (error) {
    console.error(error)
  }

  // 回调参数以 dataToken 为准；兼容部分渠道使用 token
  token.value =
    queryParams.value.get('dataToken') || queryParams.value.get('token') || ''

  if (!token.value) {
    status.value = '未收到 dataToken。'
    if (isInMiniProgram()) {
      redirectToMiniProgramLogin()
    }
    return
  }

  if (isInMiniProgram() && redirectToMiniProgramHome(token.value)) {
    return
  }

  status.value = '已接收凭证。当前不在小程序环境，请手动返回小程序。'
})
</script>

<template>
  <main class="container">
    <h1>电子社保卡登录中转页</h1>
    <p class="status">{{ status }}</p>
    <p v-if="token" class="token">dataToken: {{ token }}</p>
  </main>
</template>
