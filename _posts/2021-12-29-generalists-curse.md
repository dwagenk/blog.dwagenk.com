---
layout: post
title: "The Generalist's Curse"
category: Debugging
tags: nix, nixos, engineering, debugging
excerpt_separator: <!--more-->
---

It's time for another blog post. I've been working on a side project over the
past couple of weeks. It goes by the preliminary name `kodinix` and is yet
another incarnation of the kodi media center software on a small single board
computer (SBC). It is similar to [LibreELEC](https://libreelec.tv/), but adding
a crucial bit of functionality: the ability to open a browser as fallback for
accessing web content that is not covered by a kodi plugin. More details will
follow shortly (the usual promise :wink:).

In this context I had to debug some issues with the graphics stack that drove
me nuts for a few days. Working with embedded linux systems both professionally
and for fun I often touch a wide variety of software components. Each and every
one of them is a complex beast by itself. There's people and teams out there
for whom the development and maintenance for any single of these projects is a
full-time job spanning years to decades, they are without doubt experts in
their domain. Stitching together those components into a full system,
selecting, packaging and confgiuring them to work well together is a task that
leaves me with a glimpse into a wide variety of components and technologies,
but for very few of these I have deeper knowledge or insight. I'd thus consider
myself a generalist. And that sometimes makes it hard to track down issues.

<!--more-->

The generalist's curse is that the toes are far out in unknown waters when
tracking down an issue. Each of the many ponds around us could hold the key for
solving it. But getting acquainted to one pond, aquiring the needed
throughsight just to find out that the problem lays somewhere else takes a lot
of energy and time. And this can be frustrating as hell.

For this project I'm using a Pine64 H64 (model B) board. It's a nicely specced
board and should be able to suffice for video streaming. But I couldn't even
get it to load th kodi GUI. Well no, that's not true. It did. At first. But
than it didn't anymore. So I reverted to the previous NixOS configuration...

Aaaaaand

... it still didn't work. Down the rabbit hole I went.

To let you parttake in this I'll elaborate on the relevant bits of the tech
stack a little. We've got:

- the Pine64 H64 board
  - ARM SoC
  - ARM Mali T720 GPU
  - Display connected via HDMI
- mainline 64-bit ARM (aarch64) Linux kernel (v 5.10)
- a 32-bit ARM (armv7l-hf) userland
- Mesa3D graphics libs with panfrost GPU driver
- cage wayland compositor
- kodi media center software using wayland backend
- kodi browser launcher plugin
- firefox web browser

"Why mix architectures?" you might ask. Well, I'm stubborn and don't want to
run a 32bit kernel on hardware that can do better. Besides, I don't really know
if building a 32bit Linux kernel for this SoC and board would even properly
work since some relevant device-tree bits live in arm64 subdirs of the kernel
source tree. The 32bit userland is needed because I want widevine to work, a
proprietary library needed to play DRM[^DRM] ~~protected~~ restricted media.
Google doesn't really release this library for ARM architectures, but their
chromebooks ship a 32-bit ARM variant. There's archane knowledge and utilities
in the kodi community to extract the library from chromebook recovery images
and use it inside kodi.

The problems manifested after small changes and a reboot. The graphics crashed
whenever the cage compositor was started. The TTY1 console was displayed fine
though. Reverting to the previous configuration didn't bring the system back
into a working state. This got me off track, since NixOS manages the system
configuration in a declarative manner and thus should prevent exactly these
runtime inconsistencies. But I hadn't done any deeper inspection of the
graphics or the general system before this error occured, so I was also lacking
the reference to compare against.

I first blamed mesa and the panfrost GPU driver for this. The panfrost driver
is still quite young. Mesa as a system library linking to hardware specific
backends is complex and not 100% pure even in NixOS. Enough clues to start
digging here. I spent some time recompiling mesa, ensuring it provided the
panfrost driver and left out unrelated drivers. In the meantime I read up on
mesa's docs and the development state of the panfrost driver.  By the time
everything was ready for testing I had almost forgotten what the next planned
step was. The long compile times of system components make it hard to stay
focused on the assumption that I'm currently evaluating. I find hand-written
notes quite helpful to keep some structure with this, whilest digital
notekeeping add even more context switches and thus make things even worse.

I figured out it wasn't panfrosts fault. Starting the system with a software
rendering driver yielded the same result. Mesa's documentation listed some
environment variables that increased verbosity. The wall of text hinted at a
problem with buffer allocation. After some research I tried increasing the
memory region reserved for the kernels ContiniousMemoryAllocator (CMA) by
adding a [`cma=`
argument](https://www.kernel.org/doc/html/v5.10/admin-guide/kernel-parameters.html?highlight=cma)
to the kernel cmdline. It didn't work and I moved on. In hindsight this was the
correct approach probably just carried out wrong.

I trimmed down my test setup a bit, experienced the same issue with test
programs like kmscube and started inspection of why I couldn't get any core
dumps (probably due to systemd-coredumpd and the kernel being incompatible due
to the architecture mismatch). Many time consuming dead ends. At some
intermediary point I had it mostly working again by hard coding the
ExtendedDisplayIdentificationData (EDID) to use a lower display resolution. The
graphics didn't crash immediately anymore, the crash was delayed to some random
point later in time. This and the fact that the crashlogs - where present -
still mentioned buffer allocation errors made me revisit the CMA a few days
later. Despite having tried it before. This time it worked.

I cursed. And I was happy to finally have a solution to the problem. I've
learned some more about the linux graphics stack and especially mesa in the
meantime, but I still feel like the next similar poroblem will probably be way
above my head again. This is the curse of the generalist...

[^DRM]: Speaking of DigitalRightsManagement aka DigitalRestrictionsManagement here. The linux kernel graphics stack also includes a component named DRM that can lead to a little confusion.

----

Feel free to contact me via [mail](mailto:dwagenk@mailbox.org) or
[mastodon](https://chaos.social/@dwagenk) if you've got any notes on this post
or start a public discussion via this blogs [github issue
tracker](https://github.com/dwagenk/blog.dwagenk.com/issues). You can also
subscribe to the [RSS feed](/feed.xml) to stay tuned! 
