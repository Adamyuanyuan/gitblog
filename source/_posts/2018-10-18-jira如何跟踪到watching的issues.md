layout: 
title: apache jira如何跟踪到watching的issues
date: 2018-10-18 10:57:01
tags:

---
当你遇到一个Apache（Hadoop、Spark、Hbase...）的bug的时候，会去 issue.apache.org 去寻找，可能这个问题已经被人提出，目前正在讨论但没有解决，然后你对感兴趣的issue，默默地点了 Start watching this issue，等着有缘人去解决，或等以后有空闲时间了提个PR；

过段时间后你会发现，找不到了，左边面板里没有一个是正在 watching的issue，而浏览过得可能因为时间过长导致太多，如果忘记了jira号，就会很麻烦。

{% asset_img jira.png %}

此时在Search栏中键入：
```
watcher = currentUser() ORDER BY priority DESC, updated DESC
```
即可完美解决这个问题，得到 watching列表。