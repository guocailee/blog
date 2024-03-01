---
{"dg-publish":true,"permalink":"/life/tips/mac/","noteIcon":""}
---

[[Life/Tips/Tips\|Tips]]

## 长按生效

```bash
# 使长按生效
defaults write -g ApplePressAndHoldEnabled -bool true
```

## 显示占用端口进程

## RM .DS_Store

```bash
find . -name .DS_Store -print0 | xargs -0 git rm -f --ignore-unmatch
echo .DS_Store >> .gitignore
git add .gitignore
git commit -m '.DS_Store banished!'
```
```