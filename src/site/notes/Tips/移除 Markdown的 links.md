---
{"dg-publish":true,"permalink":"/Tips/移除 Markdown的 links/","noteIcon":"","created":"2026-05-16T12:34:51.581+08:00","dg-note-properties":{}}
---

1. using the Regex Find and Replace Add on.
2. run the “find and replace” command.
3. in the find section insert: ```(?:__|[#])|[(.?)] (. *?) ```
4. in the replace section insert: ``` $1 ```.
5. if you select a section of the text and hit the command, there is the option to only replace links there.