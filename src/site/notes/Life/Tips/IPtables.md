---
{"dg-publish":true,"permalink":"/life/tips/i-ptables/","noteIcon":""}
---

```bash
# show all rull
iptables -n -L
iptables -I FORWARD -p tcp  -m multiport --dport 0:79,81:442,444:1000,10001:16822,16824:28584,28586:40000 -j DROP

# save
iptables-save


```