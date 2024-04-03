---
{"dg-publish":true,"permalink":"/XLife/Tips/cloudflare cancel page deployment/","noteIcon":""}
---

CloudFlare 批量取消 `Queued` 的`pages deployement`

```javascript
const accountId = location.href.split('/')[3]
Array.from(document.querySelectorAll('a'))
  .filter((a) => a.href.includes(`/${accountId}/pages/view`)&& a.href.includes("-"))
  .map((it) => it.href.split('/').pop())
  .filter(it=>it.includes('-'))
  .forEach((id) => {
    fetch(`https://dash.cloudflare.com/api/v4/accounts/${accountId}/pages/projects/blog/deployments/${id}/cancel`, {
      headers: {
        accept: '*/*',
        'accept-language': 'zh-CN,zh;q=0.9',
        'cache-control': 'no-cache',
        pragma: 'no-cache',
        'sec-ch-ua': '"Chromium";v="116", "Not)A;Brand";v="24", "Google Chrome";v="116"',
        'sec-ch-ua-mobile': '?0',
        'sec-ch-ua-platform': '"macOS"',
        'sec-fetch-dest': 'empty',
        'sec-fetch-mode': 'cors',
        'sec-fetch-site': 'same-origin',
        'x-cross-site-security': 'dash',
      },
      referrer: 'https://dash.cloudflare.com/',
      referrerPolicy: 'origin',
      body: null,
      method: 'POST',
      mode: 'cors',
      credentials: 'include',
    })
  })

```