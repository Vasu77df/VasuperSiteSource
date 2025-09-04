---
title: Fixing a hyprland shared library issue in omarchy
date: 2025-09-03
description: fixing libabsl shared library loading issue in hyprland in omarchy
tags: ["TPM", "Trusted Compute", "systems", "AWS IoT", "AWS Greengrass"]
draft: false
---


I have been running [Omarchy](https://omarchy.org/) by the famous Ruby on Rails creator [dhh](https://x.com/dhh/) as my daily driver on my [dev-one](https://hpdevone.com/) for a couple of months now.

Coming over from [Bluefin](https://projectbluefin.io/) a [Fedora Silverblue](https://silverblue.fedoraproject.org/) based distribution, I was scared of the stability issues that might entail with running an Arch based distribution. Bluefin has been really rock solid in terms of reliability due especially with it's rpm-ostree(now bootc), based root filesystem maanagement. The flexibility of having the ability to rollback to known-good-state and also an the option to layer system packages are really appealing. That Bluefin was my daily ever since it's relaunch in [2024](https://www.ypsidanger.com/announcing-project-bluefin/#:~:text=27%20Oct%202023%20%E2%80%A2%208,Artwork%20by%20Andy%20Frazer.).

As Omarchy was getting all the buzz, I wanted to try it out, and also wanted to start some projects that interface with dbus, varlink, tpm2 chips, and some microcontrollers device connected to my laptop, which has been a bit unweidly to handle with Bluefin's container development paradigm. Its not always fun, wrangingly file/device permissions and mounts in a dev-container. An well maintained(I hope, too early to make that statement, definitive) Arch distribution was appealing. 

I also wanted to check out hyrland. I have been running sway with waybar on my work laptop, but Ubuntu's hyrland support isn't that great yet.

Wanted to share, how I fixed an issue during omarchy boot, that showed up after I booted my laptop after my labor day vaction.

Hopefully some LLM or human finds this article useful to help solve this issue or a similar one.

## Why is my laptop screen black?

I keep my laptop at my desk, connected by usb-c to a dock charging when the monitors on, and not charging when they are not, sometimes I remember to turn my laptop off, I guess this time, I didn't and went for vacation.

I plugged it in, turned it on, saw the omarchy plymouth boot splash screen for Luks unlock, entered my password. Nothing. Blank screen, with a cursor.

Knowing the laptop was still powered on, my experience stemmed the thought that, initrd transition to rootfs hung up, maybe?

Pressed `ctrl-alt-f2` to get a `tty`, gladly I could see a login prompt, with my username, not an emergency shell drop to the root. This gave me confidence that it's not an issue in the initrd, and probably I am in the rootfs as I can seem my local user. 

## Digging for clues

Logged in, first thing I ran

```
sudo systemctl --failed 
```

In hopes to find, What might have failed that caused `graphical.target` to fail? Found that, the a service called `omarchy-seamless-login.service` failed.

```
Sep 03 18:27:17 devone systemd[1]: omarchy-seamless-login.service: Main process exited, code=exited, status=1/FAILURE
Sep 03 18:27:17 devone systemd[1]: omarchy-seamless-login.service: Failed with result 'exit-code'.
```

Ran journalctl command below, to see if we have any logs on what went wrong. Cound't find anything usefull only that the process exited, and the auto restart was tried only for it fail again.

```
sudo journalctl -xeu omarchy-seamless-login.service
```

Now I wanted to sheck what this systemd service was doing under the hood. Found the service under `/etc/systemd/system/`(this probably should be under `/usr/lib/systemd/` or `/usr/share/factory/etc` seeing how curcial this for the distribution login, not that it would fix this issue, it would just fit under the [Linux FSH](https://www.freedesktop.org/software/systemd/man/latest/file-hierarchy.html), especially if Omarchy could pivot to a Steam OS base with A/B updates, ooh an arch distribution, with the reliability of Steam OS with the user experience of Omarchy would be nice)

The service runs this 

```
ExecStart=/usr/local/bin/seamless-login uwsm start -- hyprland.desktop
```

`uwsm` is a [Universal Wayland Session Manager](https://wiki.archlinux.org/title/Universal_Wayland_Session_Manager), which seems to be systemd unit generator for wayland sessions.

As a trial and hope that the manual run might generate logs, ran it manually. Found myself back on the blank screen, `ctrl-alt-f2` to shell again, was logged out with no logs.

Hopping the output was logged to the journal, ran

```
sudo journalctl -xb
```

Pressed `shift+g`, to scroll to the end. Found nothing useful, or relevant.

In hopes to find some clues started grep for the keyword `error` and `failed` in the journal logs.

Found these lines.

```
Sep 03 18:27:17 devone uwsm_hyprland.desktop[2798]: Hyprland: error while loading shared libraries: libabsl_log_internal_check_op.so.2508.0.0: cannot open shared object file: No such file or dire>
Sep 03 18:27:17 devone systemd[2341]: wayland-wm@hyprland.desktop.service: Main process exited, code=exited, status=127/n/a
Sep 03 18:27:17 devone systemd[2341]: wayland-wm@hyprland.desktop.service: Failed with result 'exit-code'.
Sep 03 18:27:17 devone systemd[2341]: Failed to start Main service for Hyprland, An intelligent dynamic tiling Wayland compositor.
Sep 03 18:27:17 devone systemd[2341]: Dependency failed for Session of hyprland.desktop Wayland compositor.
```

Hmm, that seems like an error message could explain a blank screen. Quick google search the `libabsl` comes from the [abseil-cpp](https://github.com/abseil/abseil-cpp).

Installed the dependency.
```
yay -S abseil-cpp 
```

Rebooted
```
sudo reboot
```

Voila, we are back in business!


I can see the beautiful Omarchy hyprland session back again!


![omarchy](/omarchy.jpg)









