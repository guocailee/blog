---
{"dg-publish":true,"permalink":"/Program/Mixed/Github抽风解决办法/","noteIcon":"","created":"2025-09-09T02:08:30.820+08:00"}
---

## 设置Proxy
```bash
git config --global https.proxy http://127.0.0.1:8001

git config --global https.proxy https://127.0.0.1:8001

git config --global --unset http.proxy

git config --global --unset https.proxy

npm config delete proxy
```

### 设置443端口
> To set this in your SSH configuration file, edit the file at ~/.ssh/config, and add this section:
```bash
Host github.com
    Hostname ssh.github.com
    Port 443
    User git
```

> thanks: https://gist.github.com/laispace/666dd7b27e9116faece6