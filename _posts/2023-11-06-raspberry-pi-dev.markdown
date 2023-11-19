---
layout: post
title:  "dailying a raspberry pi 4b"
date:   2023-11-06 00:00:00 -0500
tags: raspberry-pi
---

# tl;dr
don't

# why not?
- after the case, SD/SSD drive, charger, etc.. it's marginally cheaper than a budget mini-pc
- even with overclocking, it's slow
  - slow as in, "a computer from 15 years ago with the 15 years of fragmentation and limewire viruses" slow
- ymmv with any distro that isn't Raspberry Pi OS. Fedora 38 took a long time to boot and the GUI didn't work
- i'm assuming the second 4k60 HDMI port is in case the first one breaks because anything more than 1080p30 and you're noticably missing frames/stuttering.
- workflow of queueing multiple sites or videos in tabs will test your patience
- wifi speeds are inconsistent. Pixel 6a records 400Mb/s while Pi will top out at 70Mb/s (and takes a while to ramp up to that number)

# still want it?
- 4GB memory is probably enough. you'll be hard press to get above that without system freezing
- it's still usable. can run:
  - vim + some plugins
  - terminator
  - zsh + powerline
- not great but the new Bookworm os has been better in my experience:
  - it makes firefox a first class citizen
  - note, you should update `~/.config/wayfire.ini` now.
- stock up on tea/coffee if you're using rust
- python is fine
  - ~~not entirely sure why but i had to include PYTHONIOENCODING=utf-8 for pipenv to work~~ it seems i installed the non-utf8 version of en_CA. lesson learned, don't be patriotic. note, i had to reinstall postgresql (because it was easier than figuring out how to change it to utf-8)
  - i had to rename `/etc/_pip.conf` as the `https://www.piwheels.org/simple` would give conflicting packages.

## revisions
- 2023-11-08: update why i was getting encoding error and spell raspberry properly
