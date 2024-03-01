---
{"dg-publish":true,"permalink":"/program/vcs/git-assume-unchanged/","noteIcon":""}
---


```bash
# unchanged
git update-index --assume-unchanged package.json
# undo 
git update-index --no-assume-unchanged package.json
# list
git ls-files -v|grep '^h'
```