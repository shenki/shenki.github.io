---
layout: post
title: Git and mailing list patch changelog workflow
---

When writing commit messages for patch sets sent via email on the Linux Kernel
Mailing List it is convention to include a changelog between the current and
previous versions of the patch.

When you run `git format-patch` it spits out a file like this. Convention is to
put your changelog below the snip line, which is used by patch applying tool
`git am` to indicate the end of the commit message:

```
rom 35ec8b09bd045f45ead22979dc5596247ac85881 Mon Sep 17 00:00:00 2001
From: Joel Stanley <joel@jms.id.au>
Date: Wed, 27 Mar 2019 11:32:15 +1030
Subject: [PATCH] rtc: Add ASPEED RTC driver

Read and writes the time to the non-battery backed RTC in the ASPEED BMC
system on chip families.

Signed-off-by: Joel Stanley <joel@jms.id.au>
Signed-off-by: Alexandre Belloni <alexandre.belloni@bootlin.com>
---
v2:
 - Fix memory leak
 - Invert Christmas tree

 drivers/rtc/Kconfig      |  11 ++++
 drivers/rtc/Makefile     |   1 +
 drivers/rtc/rtc-aspeed.c | 136 +++++++++++++++++++++++++++++++++++++++
 3 files changed, 148 insertions(+)
 create mode 100644 drivers/rtc/rtc-aspeed.c
```

However, this gets tricky when you have many version of the changelog. Or when
you realise you have a typo in a comment, and re-generate the patch, only to
overwrite your changelog.

It turns out `git am` stops ath the first instance of `---` in your patch commit
message. We can take advantage of this by putting the changelog in the commit
message below our own snip line:


```
rtc: Add ASPEED RTC driver

Read and writes the time to the non-battery backed RTC in the ASPEED BMC
system on chip families.

Signed-off-by: Joel Stanley <joel@jms.id.au>
---
v2:
 - Fix memory leak
 - Invert Christmas tree
 - Correct mispelling
```

Now when we do a `git format-patch`:

```
rom 35ec8b09bd045f45ead22979dc5596247ac85881 Mon Sep 17 00:00:00 2001
From: Joel Stanley <joel@jms.id.au>
Date: Wed, 27 Mar 2019 11:32:15 +1030
Subject: [PATCH] rtc: Add ASPEED RTC driver

Read and writes the time to the non-battery backed RTC in the ASPEED BMC
system on chip families.

Signed-off-by: Joel Stanley <joel@jms.id.au>
Signed-off-by: Alexandre Belloni <alexandre.belloni@bootlin.com>
(cherry picked from commit 184a182ed52a9b224cfa081a01e920c6ab1ce0bd)
Signed-off-by: Joel Stanley <joel@jms.id.au>
---
v2:
 - Fix memory leak
 - Invert Christmas tree
 - Correct mispelling
---

 drivers/rtc/Kconfig      |  11 ++++
 drivers/rtc/Makefile     |   1 +
 drivers/rtc/rtc-aspeed.c | 136 +++++++++++++++++++++++++++++++++++++++
 3 files changed, 148 insertions(+)
 create mode 100644 drivers/rtc/rtc-aspeed.c
```

There are two snip lines, but that's fine, as git am stops at the first one,
and the metadata (such as the diffstat) below is ignored.

You can use this method to "automatically" create changelogs when making fixes
following a review. Make a single change, such as removing the pesky memory
leak, and then create a commit with the one line:

```
 - Fix memory leak
```

Do the same for the other reivew comments. Then, when you're done, use `git
rebase -i` to squash the changelog messages, and the changes, into your commit
message:
```
pick ba238bef86ac - Fix memory leak
pick 04c6c6b58cdc - Invert Christmas tree
pick 9328612a3c78 - Correct mispelling
pcik ae3b371109e0 rtc: Add ASPEED RTC driver
```
Change the `pick` to `s` for squash, and move the commits to be before the
patch you're editing:

```
pcik ae3b371109e0 rtc: Add ASPEED RTC driver
s ba238bef86ac - Fix memory leak
s 04c6c6b58cdc - Invert Christmas tree
s 9328612a3c78 - Correct mispelling
```

git will gather up the commits you have and present them below the patch's
commit message. You can massage the format in your editor and then quit to
complete the rebase.
