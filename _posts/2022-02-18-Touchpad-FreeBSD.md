---
layout: post
title: Hard Target
intro: The true story about one touchpad fix on FreeBSD.    
introImg: assets/images/freebsd-psm/feb17logo.jpg
titleImg: assets/images/freebsd-psm/feb17.jpg
comments: true
---

This rainy day I finally solved one of my longest-running issues with FreeBSD:

> [psm] Mouse cursor freezes for some seconds in X

[https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=108659](https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=108659)

So simple description actually hides a sea of pain: a bug was never 100% reproducible, the behavior was different between FreeBSD releases and  ACPI suspend/resume could freeze the touchpad completely. 

All as we love ;) 

It took almost 5 years to track roots and solve this problem.

I've got this problem with Synaptics touchpads, primary on my Fujitsu-Siemens Lifebook U775.
But there should be more vendors affected because touchpad behavior had been changed somewhere after FreeBSD 10. x release.

Basically, if you need **i8042.notimeout** kernel parameter in Linux (see below) to make the touchpad work - you would face this bug in FreeBSD too.

Below are some links on similar issues with touchpad requiring i8042.notimeout setting:

[https://askubuntu.com/questions/322906/touchpad-doesnt-work-on-fujitsu-lifebook-uh552](https://askubuntu.com/questions/322906/touchpad-doesnt-work-on-fujitsu-lifebook-uh552)

[https://www.reddit.com/r/linux/comments/2a8cyf/anyway_to_fix_touchpad_without_using/](https://www.reddit.com/r/linux/comments/2a8cyf/anyway_to_fix_touchpad_without_using/)


### The root of evil

This code block in FreeBSD PSM driver is responsible for timeouts check:

 ![Source Sample](/assets/images/freebsd-psm/freebsd-psm-screen1.png)

[https://github.com/freebsd/freebsd-src/blob/releng/13.0/sys/dev/atkbdc/psm.c#3014](https://github.com/freebsd/freebsd-src/blob/releng/13.0/sys/dev/atkbdc/psm.c#3014)

The main idea is to discard received bytes if the delivery has been delayed on more than the expected limit.

Let's open Linux driver sources and look inside, especially on this comment:

 
>  When MUXERR condition is signalled the data register can only contain
>  0xfd, 0xfe or 0xff if implementation follows the spec. Unfortunately
>  it is not always the case. Some KBCs also report 0xfc when there is
>  nothing connected to the port while others sometimes get confused which
>  port the data came from and signal error leaving the data intact. They
>  _do not_ revert to legacy mode (actually I've never seen KBC reverting
>  to legacy mode yet, when we see one we'll add proper handling).
>  Anyway, *we process 0xfc, 0xfd, 0xfe and 0xff as timeouts*, and for the
>  rest assume that the data came from the same serio last byte
>  was transmitted (if transmission happened not too long ago).
 

[https://github.com/torvalds/linux/blob/master/drivers/input/serio/i8042.c#555](https://github.com/torvalds/linux/blob/master/drivers/input/serio/i8042.c#555)


So what happens: Synaptics touchpad sends vendor-specific bytes and FreeBSD driver discards them with correct ones. 


### The 'patch'

Pach is actually a hack, that just disables described timeout checking, that's all.
I've got no kernel panics yet, also touchpad now survives multiple ACPI suspend/resume actions.
So probably my small patch would not add additional instability





 ```diff
--- psm.c	2022-02-17 21:22:45.481834000 +0300
+++ psm.c.1	2022-02-17 21:25:30.616384000 +0300
@@ -3011,14 +3011,14 @@
 			continue;
 
 		getmicrouptime(&now);
-		if ((pb->inputbytes > 0) &&
+		/*if ((pb->inputbytes > 0) &&
 		    timevalcmp(&now, &sc->inputtimeout, >)) {
 			VLOG(3, (LOG_DEBUG, "psmintr: delay too long; "
 			    "resetting byte count\n"));
 			pb->inputbytes = 0;
 			sc->syncerrors = 0;
 			sc->pkterrors = 0;
-		}
+		}*/
 		sc->inputtimeout.tv_sec = PSM_INPUT_TIMEOUT / 1000000;
 		sc->inputtimeout.tv_usec = PSM_INPUT_TIMEOUT % 1000000;
 		timevaladd(&sc->inputtimeout, &now);
```
Save this snippet to psm.patch file then run:

```
 cd /usr/src/sys/dev/atkbdc
 patch < /path/to/psm.patch
```
