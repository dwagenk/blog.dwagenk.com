---
layout: post
title: "Diving into NixOS: Printing"
category: Nix
tags: nix, nixos, printing
excerpt_separator: <!--more-->
---

I like to dive into problems to learn new skills. Especially when it comes to
technology and crafts. This led to a self-made cargo-bike, basic welding
skills, but also the basis for my profession: knowledge of embedded systems,
programming in C and configuring various software systems.

In this and probably two future blog posts I want to document my last deep-dive
into a new-to-me technology:

_Nix_/_NixOS_ or more specifically getting my printer/scanner combo device to work
well with _NixOS_.

<!--more-->

_Nix_ is a functional approach to packaging software, and approaching system
configuration. The OS is configured declaratively by writing config files,
instead of changing the system in an iterative way. I've used both _Debian_ and
_Arch_ _Linux_ based distributions before and experienced problems regarding
software dependencies and malfunctioning packages now and then with them.
Probably _NixOS_ will have some challenges for me as well, but I'm more willing
to tackle them.  Whatever I do to mitigate those is documented and version
controlled in my system configuration and won't come back to haunt me (_ooh, I
remember I spent hours debugging and solving this problem, but what the heck
did I do?_). I've had _NixOS_ installed on my personal laptop for 3 months,
with barely enough configuration for it to serve my everyday needs, but without
wrapping my head around it yet. So it's about time to develop better skills
around the system I'm using.

I'll use this post to document the chunk that worked quite flawlessly:
getting the printer to work. I'll have a similar post for the scanner as well,
which was more challenging to get running well.

I've bought a scanner/printer combo device (_Canon PIXMA TR4551_) with USB and
Wi-Fi connectivity about one and a half years ago.  It worked OK with my
previous _Arch_ _Linux_ setup but needed proprietary software and drivers provided
by the vendor.

## IPP Everywhere

Before installing those drivers again on _NixOS_, I tried out using the printer
with available free software drivers. The printer supports _IPP_
(InternetPrintingProtocol) for access via the network. _CUPS_
(CommonUnixPrintingService, the standard way to access printers on _Linux_) has a
built-in `Generic IPP Everywhere Printer` driver. To get
this working on _NixOS_ I added the line
```nix
services.printing.enable = true;
```
to my `configuration.nix` file.

Turns out this driver works OK with my printer, but there are some pitfalls:
* The driver allows selecting the print quality in DPI from a drop-down with
  many many options. Selecting a value that's not well-supported by my printer
  causes no error message, but weirdly scaled pages
* A similar issue arises with the color-profile selection
* The duplex printing function is inversed

Those pitfalls are more or less cosmetic, but would probably be a constant
source of failed printouts in the future. I'm human and tend to forget
which options I need to choose. I don't want to be able to select unsupported
modes.

It's nice to have the printer supported kinda-out-of-the-box. I decided to try
using _Canon_'s proprietary drivers as I remember them to work a little better.

## Proprietary Driver

The drivers provided by _Canon_ are already available as a package in _NixOS_!
The right keyword to use in the _Nix_ package search is _cnijfilter2_ which is
also part of the filename of the driver archive offered for download at
_Canon_'s website.

I went on extending my `configuration.nix` file like suggested in the [_NixOS_
wiki](https://nixos.wiki/wiki/Printing)
```nix
services.printing.enable = true;
services.printing.drivers = [ pkgs.cnijfilter2 ];
```
This led to an error while rebuilding the _NixOS_ system:
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
to `configuration.nix` solved the problem. The printer works with the new
driver and I'm glad it offers fewer options and therefore fewer possibilities
for me to screw up, but [...]

## A Note on Free Software

[...] I don't like the `allowUnfree = true` setting. I'm a big fan of free
software and use it whenever there's a viable solution available. Being
pragmatic I use proprietary software on my devices as well. The decision should
remain on a per-package base though, not to give carte blanche for all unfree
packages. I found the relevant
[section](https://nixos.org/nixpkgs/manual/#sec-allow-unfree) in the nixpkgs
manual, but the solution presented there didn't work. Turns out there's also a
[github
issue](https://github.com/NixOS/nixpkgs/issues/67830#issuecomment-542933255)
mentioning the documentation being incorrect and offering a working alternative
for allowing individual unfree packages.
```nix
nixpkgs.config.allowUnfreePredicate = (pkg: builtins.elem
  pkg.pname [
    "cnijfilter2"
  ]
);
```

## Network Printing

The printer offers USB and Wi-Fi as means of communication. Using the
_cnijfilter2_ driver _CUPS_ finds the printer via it's USB connection, but not
via Wi-Fi. Printing via Wi-Fi works after configuring the printer manually
(`ipp://123.ip.of.printer`). It would be nice to have _CUPS_ auto-detect
network printers. Without going into detail (I really don't know a lot about
auto-discovery of networked devices and just followed the clues from the
_NixOS_ wiki) here's what I had to add to my `configuration.nix` for _CUPS_ to
automatically find the printer via Wi-Fi:
```nix
services.avahi.enable = true;
services.avahi.nssmdns = true;
```
> Note: the printer needs to be in the same IP-subnet as the computer for
auto-discovery to work.

## Declarative Printer Configuration

I've skipped over the part of actually configuring the printer, since I've not
done it _the Nix way_ yet. Configuring printers declaratively in _NixOS_ seems
to be possible and I'll take a look into it later.

## Config File

To summarize, this is the section in my `configuration.nix` related to printing
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
