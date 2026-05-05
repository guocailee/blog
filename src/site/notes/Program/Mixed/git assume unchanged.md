---
{"dg-publish":true,"permalink":"/Program/Mixed/git assume unchanged/","noteIcon":"","created":"2026-01-24T01:53:54.568+08:00","dg-note-properties":{}}
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