---
{"dg-publish":true,"permalink":"/Program/Codes/RabbitMQ Code/","noteIcon":"","created":"2026-01-12T11:34:29.834+08:00"}
---

## 根据Rabbit WEBUI 批量删除队列
```javascript
const deleteQueue = (queue) => {
  return fetch(`/api/queues/${YOUR_AUTH}/${queue}`, {
    headers: {
      accept: '*/*',
      'accept-language': 'en-US,en;q=0.9,zh-CN;q=0.8,zh;q=0.7',
      authorization: 'Basic YOUR_AUTH',
      'content-type': 'application/json',
    },#
    referrer: 'http://webui:15672/',
    body: `{"vhost":"YOUR_HOST","name":"${queue}","mode":"delete"}`,
    method: 'DELETE',
    mode: 'cors',
    credentials: 'include',
  })
}
Array.from(document.querySelectorAll('table.list a'))
  .filter((it) => it.attributes['href']?.value?.startsWith('#/queues'))
  .map((it) => it.textContent?.trim()).filter(it=>it).forEach(deleteQueue)

```