---
{"dg-publish":true,"permalink":"/Program/Mixed/git assume unchanged/","noteIcon":"","created":"2024-05-22T16:17:54.161+08:00"}
---

#Git 

```bash
# unchanged
git update-index --assume-unchanged package.json
# undo 
git update-index --no-assume-unchanged package.json
# list
git ls-files -v|grep '^h'
```