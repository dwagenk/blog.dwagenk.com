---
layout: post
title: "Diving into NixOS: Scanning"
category: Nix
tags: nix, nixos, scanning, 15WeeksToBlog
excerpt_separator: <!--more-->
---

I announced posting more about my journey getting familiar with _Nix/NixOS_ and
getting my printer/scanner combo device to work well with it. Setting myself a
challenge to keep blogging weekly
([#15WeeksToBlog](https://chaos.social/web/timelines/tag/15WeeksToBlog)) and
seeing a lot of other peoples blog posts appearing has motivated me enough to
actually get this post written down albeit a lot later than planned.

<!--more-->

Like mentioned in the [post about printing](/nix/2020/04/nix-printing) I've got
a _Canon PIXMA TR4551_ printer/scanner combo device with USB and Wi-Fi
connectivity. I've used it for scanning with linux before, using _Canons_
proprietary scanning application _scangearmp2_. Using it is no fun, it doesn't
remember any preferences like the preferred paper format. The worst part: the
save dialog window is unusable with the keyboard. It doesn't preselect the file
name input field and the save button is absolutely inaccessible by keyboard.

>In most applications there's underlined letters in every button, menu item
>etc.  when pressing the ALT key, that allow selecting them via the keyboard.
>Alternatively they accept ENTER for the default action (OK/SAVE/CONFIRM) and
>ESC to abort. This application does neither.

The scanner has an automatic document feeder (ADF) that is supported by the
application, so scanning a multi-page document is OK, as long as it's
single-sided and fits into the feeder.  

Well, I guess the impression, why I'd rather use some "better" software got
across, so let's boil this down into a list of requirements for a driver or
scanner-application:

* works via network / Wi-Fi
* allows selection of default values (paper format, source, color-mode) or
  remembers the last used options
* doesn't require manual input for the file name. A quick scan with a default
  filename should not involve any other button presses the OK, OK, NEXT, SAVE,
  OK, ... (and not that many)
* is usable without reaching for the mouse
* supports the ADF
* (optional) Scanning the next page of a multi-page document can be triggered
  by pressing a button on the scanner
* (optional) Text recognition (OCR)

# SANE

The usual setup for scanning on linux involves the _SANE Project_, with the
acronym standing for _ScannerAccessNowEasy_. _SANE_ has a huge selection of
backends — the device-specific drivers — and provides a common API for their
usage, so that the actual applications (the _SANE_ frontends) are device
independent. Regarding the frontends I've used
[gscan2pdf](http://gscan2pdf.sourceforge.net/) for documents and
[SimpleScan](https://gitlab.gnome.org/GNOME/simple-scan) for quick scans of
single pages or photos in the past and the combination of those tick all the
frontend-related boxes of my requirements list from above. So all that's
missing is a network-enabled _SANE_ backend for my printer which also supports
the ADF and optionally can trigger the next scan via an on-device button.

## Backends

Online-research led to quite a few options with a varying amount of information
available:

Backend  | included with _SANE_ <br/> (version) | packaged for _Nix_ <br/> (channel) | Wi-Fi  | USB  |  ADF | on-device button 
---|---|---|---|---|--- 
eSCL 		| :heavy_check_mark: (1.0.29) | :x: | :heavy_check_mark: | :x: | :heavy_check_mark: | :x:
airscan 	| :x: | :heavy_check_mark: (unstable) | :heavy_check_mark: | :x: | :heavy_check_mark: | :x: 
pixma 		| :heavy_check_mark:(1.0.28) | :heavy_check_mark: (20.03) | :grey_question: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: 
canon-pixma 	| :x: | :x: | :heavy_check_mark: | :heavy_check_mark: | :grey_question: | :grey_question:

### pixma 

Support for my scanner via the _pixma_ backend might work. The list of features
that are supported in general, but might or might not be
supported for my device, is long, including niceties like button controlled
scanning, adjusting image settings (gamma, brightness) or setting a wait time
for when the automated document feeder runs empty, to allow reloading it and
continue scanning the same document. The documentation mentioned networked
scanning support via the proprietary _Canon BJNP_ protocol, but I didn't find
any information, whether this was supported by my device. 

In recent versions of the [_SANE_ list of supported
hardware](http://www.sane-project.org/sane-mfgs.html#Z-CANON) the device is
listed as `untested` meaning its information has been added to the backend, to
enable testing of the status of device-support without further modification to
the source code. My switch to the `nixos-20.03` channel containing the required
_SANE_ package was due anyhow, so I switched channels, waited for the system to
update and installed _SANE_. 

The scanner wasn't found via the network even with temporarily disabled
firewall, so the _pixma_ backend won't work for me. I did a quick check of all
the other functions with the scanner connected via USB. Scanning from the
flatbed worked, but I got an error when trying to use the ADF. This is now
reported back to the _SANE_ project.
> If you encounter bugs or other annoyances in openly
> developed software, please do file a bug, provide fixes or suggest improvements
> to the documentation (some projects pay a lot of attention to keeping their
> barrier of entry low and their communication tone very welcoming, to make it
> easy for people with and without coding skills to contribute). 

### eSCL and airscan

The eSCL and airscan backends both use the HTTP-based eSCL[^escl] protocol and
don't need device-specific drivers. The protocol seems to be well known through
_Apples_ use of it and is supported by a quite broad palette of network
connected scanners, but it seems to not be publicly documented. Work on
reverse-engineering and reimplementing (parts of) it in free software drivers
is has just started recently, with most documentation notes and work on the SANE backends
dating to 2019.

Details on how to get those backends working in _Nix_ will be part of a separate
blog post, but I sure was surprised about the amount of work that can be
associated with bumping up a packages version number.

Using the airscan backend scanner discovery, scanning from flatbed and ADF all
work[^sane-airscan-error], so the remaining parts of my requirements list are
covered and I guess I could stop at this point. But I'm trying to get an
overview of the available options here and experiment with _Nix_ packaging, so
I can just as well go on and evaluate the remaining backends. 

The eSCL backend has been a moving target while preparing this blog post. The
version contained in release 1.0.29 of the _SANE_ backends was unpolished,
selecting the correct paper-size was buggy and it didn't support the ADF yet.
This changed as development went on and the most recent commit fixing some
remaining ADF errors happened just a few days ago. Scanning from flatbed and ADF
work when using the _SimpleScan_ GUI application, but I get messed up image
files when scanning via the commandline with the _scanimage_ tool. I did open
another bug report for this.

### canon-pixma

This backend is written as a wrapper around the proprietary driver libraries,
that are part of _scangearmp2_. The main contributor to this wrapper is also
heavily involved in the _SANE_ project, but the backend is not integrated into
the _SANE_ backends repository, I guess that's due to licencing issues.  As it
isn't packaged for _NixOS_ yet, I had a little work to do here and write my
first package definition in _Nix_. I'm still working on polishing it for a pull
request_ to the _nixpkgs_ repository, but for local testing it works already. I
also packaged the _scangearmp2_ application, to have a first
check, if the libraries and the printer work together.  _scangearmp2_ didn't
find the scanner via Wi-Fi at first, but disabling the firewall made it work and
there's no difference between the scanner connected via USB and via Wi-Fi then.
For further testing I used the USB connection and reenabled the firewall, but
finding the correct ports to open in the firewall is still on my ToDo list. All
functions provided by _scangearmp2_ seem to work, including scanning from the
ADF.

_SANE_ with the _canon-pixma_ backend finds the scanner via USB and — with
disabled firewall — via Wi-Fi as well. Scanning works fine from the flatbed, but
the ADF doesn't do anything. I haven't yet found the cause for that and given that
most of the functionality is hidden in proprietary, binary-only libraries, the
debuggability is not so good. I'll see, if I find the time and motivation to
dig deeper into that phenomenen, but since my needs are covered by other
backends and there's no additional features to be expected to be usable via
this backend, I probably won't waste a lot more time on this.

## Verdict

To sum up, here's the list of supported features again, based on what I
actually encountered:

Backend  | included with _SANE_ <br/> (version) | packaged for _Nix_ <br/> (channel) | Wi-Fi  | USB  |  ADF | on-device button 
---|---|---|---|---|--- 
eSCL 		| :beetle: (1.0.29) <br/> :heavy_check_mark: (master) | :x: | :heavy_check_mark: | :x: | :heavy_check_mark: | :x:
airscan 	| :x: | :heavy_check_mark: (unstable) | :heavy_check_mark: | :x: | :heavy_check_mark: | :x: 
pixma 		| :heavy_check_mark:(1.0.28) | :heavy_check_mark: (20.03) | :x: or :beetle: | :heavy_check_mark: | :beetle: | :beetle: 
canon-pixma 	| :x: | :x: | :heavy_check_mark: | :heavy_check_mark: | :beetle: | :x:

I'll be using the airscan backend for now, since it is packed for Nix and can
easily be integrated into my system without me having to keep local package
definitions around. 

[^escl]: eSCL actually contains both, Apple AirScan and Apple AirPrint
[^sane-airscan-error]: It reports an error when the ADF runs empty. The scanned documents are successfully retrieved though. 

----

This post is part of the
[#15WeeksToBlog](https://chaos.social/web/timelines/tag/15WeeksToBlog)
challenge.<br/> Week 2/15. Technically I failed the challenge already, skipping
more than a. I've been steadily working on the post though, ensuring my notes
from the experiments a little while ago where still correct. The backends
getting updated in the meantime led to me reevaluating some aspects. I'll
interpret this experience as a push toward smaller, more focused posts and
publishing them faster.

Feel free to contact me via [mail](mailto:dwagenk@mailbox.org) or
[mastodon](https://chaos.social/@dwagenk) if you've got any notes on this post
or start a public discussion via this blogs [github issue
tracker](https://github.com/dwagenk/blog.dwagenk.com/issues). You can also
subscribe to the [RSS feed](/feed.xml) to stay tuned! 
