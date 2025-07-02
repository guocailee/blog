---
{"dg-publish":true,"permalink":"/XLife/Tips/移除 Markdown的 links/","noteIcon":"","created":"2024-05-22T16:17:54.169+08:00"}
---

1. using the Regex Find and Replace Add on.
2. run the “find and replace” command.
3. in the find section insert: ```(?:__|[#])|[(.?)] (. *?) ```
4. in the replace section insert: ``` $1 ```.
5. if you select a section of the text and hit the command, there is the option to only replace links there.