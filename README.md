# H5 Bridge

电子社保卡授权中转页，独立项目目录：`D:\DC\ecard\h5-bridge`

## 你只需要这两步

```bash
cd D:\DC\ecard\h5-bridge
npm run dev
```

## 功能

- 接收 URL 参数 `dataToken`
- 在微信小程序 WebView 中回跳小程序，并把 `dataToken` 带回
- 不在小程序环境时，直接把 `dataToken` 展示出来

## 可选环境变量

在项目根目录创建 `.env` 或 `.env.production`：

```bash
VITE_WX_APPID=你的微信appid
VITE_WX_SIGNATURE_API=https://你的签名接口地址
```

## 回调地址示例

```text
https://你的域名/?dataToken=xxxx&redirectPath=/pages/index/index
```
