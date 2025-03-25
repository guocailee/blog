---
{"dg-publish":true,"permalink":"/XLife/Tips/Mac技巧/","noteIcon":""}
---

#MacOS #Tips

## 长按生效

```bash
# 使长按生效
defaults write -g ApplePressAndHoldEnabled -bool true
```

## 显示占用端口进程
```bash
# 查看占用进程
lsof -i tcp:8080
# 查看对应进程名称
lsof -i -P|grep 8080
```
## RM .DS_Store

```bash
find . -name .DS_Store -print0 | xargs -0 git rm -f --ignore-unmatch
echo .DS_Store >> .gitignore
git add .gitignore
git commit -m '.DS_Store banished!'
```