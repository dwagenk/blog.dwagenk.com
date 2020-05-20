---
layout: post
title: "Diving into NixOS: Printing"
category: Nix
tags: nix, nixos, printing
excerpt_separator: <!--more-->
---

I like to dive into problems to learn new skills, especially when
it comes to technology and crafts. This led to a self-made cargo-bike (back on
the road "soon", once the overdue maintenance is done) accompanied with some
basic welding skills, but also knowledge of embedded systems, programming in C
and configuring various software systems. 

In this and probably two future blog posts I want to document my last deep-dive
into a new-to-me technology: 

_Nix_/_NixOS_ or more specifically getting my printer/scanner combo device to work
well with _NixOS_.

To really grasp something I've always had to pick a few tasks that go way
further than most textbook examples and narrowed down problems in hands-on
training sessions. Since time for focused work at the desk is sparse, I often
start out by reading a lot of resources on my phone, whenever and wherever it
fits, listening to talk recordings and podcasts on the topic and keeping a record
in my head, which parts seem to need more attention for me to grasp them. When
time permits I then sit down and give myself a task targeting those parts.

<!--more-->

_Nix_ is a functional approach to packaging software, and approaching system
configuration. The OS is configured declaratively by writing config files,
instead of changing the system in an iterative way. I've used both _apt_ and
_pacman_ _Linux_ distros before and experienced problems regarding software
dependencies and malfunctioning packages every now and then with them. I guess
_NixOS_ will have some hurdles for me as well, but I'm more willing to dive
into those, since whatever I do to mitigate those will be documented and
version controlled in my system configuration and won't come back to haunt me
(_ooh, I remember I spent hours debugging and solving this problem, but what
the heck did I do?_). I've had _NixOS_ installed on my personal laptop for 3
months already, with barely enough configuration for it to serve my everyday
needs, but without wrapping my head around it yet.  I still had a secondary
drive with my previous setup at hand and booted it up more often than I'd like
to admit just to get some small task done without fiddling with the _NixOS_
system configuration. So it's about time to learn more about the system I'm
using.

## The Agenda

So I'll use this post to document the chunk that worked quite flawlessly:
getting the printer to work. I'll have a similar post for the scanner as well,
which was more complicated to get running well, with several different drivers
available, each having their own set of drawbacks. Getting the scanner to work involved updating and
adding _Nix_ packages, which is the main reason I approached this challenge at
all and deserves a separate blog post.

## The Device

I've bought a scanner/printer combo device (_Canon PIXMA TR4551_) with USB and
WIFI connectivity about one and a half years ago.  It worked OK with my
previous _Arch_ _Linux_ setup but needed proprietary software and drivers provided
by the vendor.

## IPP Everywhere

Before installing those drivers again on _NixOS_, I tried out using the printer
with available free software drivers. The printer supports _IPP_
(InternetPrintingProtocol) for access via the network and _CUPS_
(CommonUnixPrintingService, the standard way to access printers on _Linux_) has a
built-in `Generic IPP Everywhere Printer` driver. All that was necessary to get
this working on _NixOS_ was adding a line 
```nix
services.printing.enable = true;
```
to my `configuration.nix` file.

Turns out this driver works OK with my printer, but there's some pitfalls:
* The driver allows selecting the print quality in DPI from a drop-down with
  many many options. Selecting a value that's not well supported by my printer
  causes no error message, but weirdly scaled pages
* A similar issue arises with the color-profile selection
* The duplex printing function is inversed

Those pitfalls are more or less cosmetic, but would probably be a constant
source of failed printouts in the future, because I'm human and tend to forget
which options I need to choose. I don't want to be able to choose unsupported
modes.

It's nice to have the printer supported kinda-out-of-the-box and with
free software drivers, but I decided to see if I can get _Canon_'s proprietary
drivers to work, which I remembered to be working a little better.

## Proprietary Driver

The drivers provided bt _Canon_ are already available as a package in _NixOS_!
I didn't expect this, but entering the right keywords into the [_Nix_ package
search](https://nixos.org/nixos/packages.html) helps. In this case the right
keyword was _cnijfilter2_ which is also part of the filename of the driver
archive offered for download at _Canon_'s website. 

I went on extending my `configuration.nix` file like suggested in the [_NixOS_
wiki](https://nixos.wiki/wiki/Printing)
```nix
services.printing.enable = true;
services.printing.drivers = [ pkgs.cnijfilter2 ];
```
This yielded an error while rebuilding the _NixOS_ system:
```
$ sudo nixos-rebuild switch 
building Nix...
building the system configuration...
error: Package ‘cnijfilter2-5.30’ in /nix/store/ywjr6djpwc1cp4hp52yqwwgyb9vphkql-nixos-19.09.2426.9237a09d8ed/nixos/pkgs/misc/cups/drivers/cnijfilter2/default.nix:117 has an unfree license (‘unfree’), refusing to evaluate.

a) For `nixos-rebuild` you can set
  { nixpkgs.config.allowUnfree = true; }
in configuration.nix to override this.

b) For `nix-env`, `nix-build`, `nix-shell` or any other _Nix_ command you can add
  { allowUnfree = true; }
to ~/.config/nixpkgs/config.nix.

(use '--show-trace' to show detailed location information)
```
following the hint in the error message and adding
```nix
nixpkgs.config.allowUnfree = true;
```
to `configuration.nix` indeed solved the problem and the system rebuilt
successfully. The printer works with the new driver and I'm glad it offers less
options and therefore less possibilities for me to screw up, but [...]

## A Note on Free Software

[...] I don't like the `allowUnfree = true` setting. I'm a big fan of free
software and use it wherever there's a viable solution available. Being
pragmatic I use proprietary software on my devices as well, like in this case,
but the decision should remain on a per-package base and not give a carte
blanche for all unfree packages.  Trying to find information on how to achieve
this I found the relevant
[section](https://nixos.org/nixpkgs/manual/#sec-allow-unfree) in the nixpkgs
manual, but the solution presented there gave me errors when rebuilding the
system. Turns out there's also a [github
issue](https://github.com/NixOS/nixpkgs/issues/67830#issuecomment-542933255)
mentioning the documentation being incorrect and offering a working alternative
to whitelist individual unfree packages.
```nix
nixpkgs.config.allowUnfreePredicate = (pkg: builtins.elem
  pkg.pname [
    "cnijfilter2"
  ]
);
```

## USB and Network Printing

The printer offers both USB and WIFI as means of communication. Using the
_cnijfilter2_ driver _CUPS_ finds the printer via it's USB connection
automatically, but not via WIFI. Printing via WIFI works after configuring the
printer with _CUPS_ manually (`ipp://123.ip.of.printer`), but it would be nice
to have _CUPS_ auto-detect network printers. Without going into much (I really
don't know much about auto-discovery of networked devices and just followed the
clues from the _NixOS_ wiki) here's what I had to add to my `configuration.nix` for _CUPS_
to automatically find the printer via WIFI:
```nix
services.avahi.enable = true;
services.avahi.nssmdns = true;
```

> Note: the printer needs to be in the same IP-subnet as the computer for
auto-discovery to work.

## Declarative Printer Configuration

I've skipped over the part of actually configuring the printer for this post,
since I've not done it *the _Nix_ way* yet. Configuring printers declaratively
in _NixOS_ seems to be possible and I'll take a look into that later this week,
but for now I need a break.

## Config File

So to summarize, this is the section in my
`configuration.nix` related to printing
```nix
{
  #...

  services.printing.enable = true;
  services.printing.drivers = [ pkgs.cnijfilter2 ];

  nixpkgs.config.allowUnfreePredicate = (pkg: builtins.elem
    pkg.pname [
      "cnijfilter2"
    ]
  );

  services.avahi.enable = true;
  services.avahi.nssmdns = true;

  #...
}
```

Feel free to contact me via [mail](mailto:dwagenk@mailbox.org) or
[mastodon](https://chaos.social/@dwagenk) if you've got any notes on this post
or start a public discussion via this blogs [github issue
tracker](https://github.com/dwagenk/blog.dwagenk.com/issues). Like I mentioned
above, I've got more posts planned on _Nix_ and my printer/scanner, so if this
interests you subscribe to the [RSS feed](/feed.xml) to stay tuned! 

[CC-BY-SA-4.0](http://creativecommons.org/licenses/by-sa/4.0/)
