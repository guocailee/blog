---
{"dg-publish":true,"permalink":"/Program/VCS/git assume unchanged/","noteIcon":""}
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