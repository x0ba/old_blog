---
title: "Distrobox"
date: 2023-04-22T17:42:15-07:00
draft: false

author: "Aspect"
authorLink: "https://aspectsides.site"
subtitle: "Containerizing different distros with Distrobox."
summary: "This is an article explaining Distrobox, and how it can be integrated into your daily workflow."
description: "Understanding Distrobox and why people use it."

tags: ["distrobox", "containers", "linux"]
categories: ["linux", "command line"]
keywords: ["distrobox", "containers"]

featuredImage: "images/boxes.png"
featuredImagePreview: "images/boxes.png"

fraction: true
fontawesome: true
linkToMarkdown: true
rssFullText: false

toc:
  auto: true
---

# Introduction

Distrobox is a neat little tool that allows you to use any Linux distribution inside your terminal as a container. In this article, we will be looking at Distrobox, trying out some of its features, and trying to explain why someone would want to use it.


## What is Distrobox?

At its core, Distrobox is just a wrapper script around Docker or Podman that allows you to create a container with the Linux distribution of your choice. These containers are tightly integrated with the host system, allowing the sharing of home directories, external storage, audio, even graphical apps with X11 or Wayland. This obviously provides many usecases right out of the gate, including an easy way to test out a new distro's terminal environment and its without having to create a whole VM or install it on hardware. This is just touching the surface of Distrobox's potential, though.

## Why would you use Distrobox?

As mentioned before, Distrobox can be used to try out a new Linux distro, provided you only want to use use the terminal. You can also use it to get tooling that isn't available on your host distro. Say I'm on Fedora, but there's an obscure tool on the AUR that I'm too lazy to build from source. I can just spin up an Arch Distrobox and install it. Another use-case is for those of you that use immutable operating systems such as Fedora Silverblue, OpenSUSE CoreOS, and VanillaOS. Since these operating systems' root filesystem cannot be modified, it is not encouraged to install programs on the host system. Instead, GUI apps should be installed using Flatpak and everything else should be containerized. Fedora Silverblue even comes with Toolbox, the program that Distrobox aims to improve upon, preinstalled. Distrobox can be used to provide a mutable environment on these immutable operating systems, which negates many of the downsides surrounding them. Though I do not use an immutable operating system (I use Fedora Workstation), I do agree with their way of doing things, and thus try to containerize everything I can. 

# Usage

## Creating a Container

A container can theoretically be created with any image with Docker/Podman support out there. However, there is a [list of tested images](https://github.com/89luca89/distrobox/blob/main/docs/compatibility.md#containers-distros) that are pretty much guaranteed to work with Distrobox. I would suggest pulling your image from there, as you'll have much less trouble getting everything up and running. For the purposes of this guide, I'll be using Podman and creating a Debian distrobox. 

```bash
distrobox-create -i quay.io/toolbx-images/debian-toolbox:unstable -n debian -H $HOME/.var/debian
```

Let's walk through this command one flag at a time. Obviously, `distrobox-create` is the command for, well, creating a new Distrobox. From there, we have a variety of options. The ones shown above are likely the ones you'll be using the most, though. The `-i` flag specifies the image that'll be used during container creation. You can either include the full url, or just the image name. Distrobox will prompt you to choose which registry you want to pull the image from if you choose the latter option. By default, Distrobox uses a Fedora Toolbox image if you don't specify your desired image.  The `-n` flag names your Distrobox, allowing easy operations on it later. The `-H` flag specifies the *home directory* for your Distrobox. Whether to use this depends on your use case. If you intend to use your Distrobox as your user environment, you might consider omitting this flag and just sharing a home directory with your current user. However, if you want to spin up a Distrobox for testing purposes, you might want to isolate the home directory in its own folder to avoid any accidental changes to your files.

Now you can press enter, and after pulling the image, you can now run 

```bash
distrobox-enter debian
```

This will start the container and install some needed programs on the container. This initial setup will take a while, but after it finishes you will have a blazingly fast container in terms of startup time. Now you can do everything you would be able to to in a terminal environment on this container. Install packages using `apt`, run them, whatever you want. You're now using Debian. 

## Distrobox Stacks

This is already cool enough, but there's much more you can do with Distrobox. For example, if you have a stack of Distroboxes and installed packages you'd like to be able to create with one command, you can create something similar to a `docker-compose` file and initialize it with `distrobox-assemble`

```ini 
[ubuntu]
additional_packages=git vim tmux nodejs
image=ubuntu:latest
init=false
nvidia=false
pull=true
root=false
replace=true

[arch]
additional_packages=git vim tmux nodejs
home=/tmp/home
image=archlinux:latest
init=false
init_hooks="touch /init-normal"
nvidia=true
pre_init_hooks="touch /pre-init"
pull=true
root=false
replace=false
volume=/tmp/test:/run/a /tmp/test:/run/b
```

This allows for Distroboxes to be much more portable, and you can even share Distroboxes with other people.

## Commands Used Inside Distroboxes

There are three commands that can be used inside a Distrobox container. These are `distrobox-host-exec`, `distrobox-export`, and `distrobox-init`.

### distrobox-host-exec

This command is pretty self-explanatory. It allows you to execute a single command on the host. 

```bash
distrobox-host-exec flatpak install flathub org.kde.kdenlive
```

This will allow you to install a Flatpak on your host machine. Pretty handy if you don't want to leave your Distrobox to run a quick command on your host. You could also use symlinks to make `distrobox-host-exec` execute that command. 

```bash
ln -s /usr/bin/distrobox-host-exec /usr/bin/flatpak
```

Now whenever you run `flatpak` in your Distrobox container, it should automatically run it with `distrobox-host-exec`.

### distrobox-export

This command is, in my opinion, what really makes Distrobox shine. It allows you to export an app or binary from the Distrobox container to the host. It can either create a desktop entry that runs a GUI app within the container automatically or create a binary that runs a CLI command within the container. With this, the level of integration with the host system skyrockets.

Say I installed the Thunar file manager on my Debian Distrobox and I don't want to constantly enter the container and run Thunar that way. I can export the app and its desktop entry using 

```bash
distrobox-export --app thunar
```

This will create a .desktop file so that you can launch it with dmenu or rofi or whatever launcher you're using, and launch Thunar within your Distrobox container. This is very useful for GUI apps, but I don't find myself using it much with CLI apps, as I normally just use my `run` script that I'll link a bit later in the article. I'll show you how to export a CLI binary anyway.

```bash
distrobox-export -b /usr/sbin/ranger --export-path $HOME/.local/bin/
```

This will export the binary `ranger` from the container's `/usr/sbin` into the host's `~/.local/bin`.

# How I use Distrobox

Even though I personally don't use an immutable operating system, I still prefer containerizing as much as I can. I love Fedora, but it does have some shortcomings. Firstly, package availability is not the best. COPR does exist, but not many packages on there support the Apple M1 Mac's aarch64 architecture that I'm using, and I'm too lazy to build things from source. Secondly, DNF is slower than molasses during the winter. I use it when I have to, but nowadays I much prefer a setup using Distrobox.

I currently use three distroboxes on a daily basis: one running Void Linux, one running Debian Sid, and one running OpenSUSE Tumbleweed. Between these three, I have no problems with package management, since I now have the power of three package repos on my side.

When installing a package (which isn't all that often nowadays), I consider the following:
- Is it a GUI app? 
    - Look on Flathub.
    - If it's not on Flathub, install it with one of my Distroboxes and export it to the host.
    - If some weird bug arises with this, then install it with DNF.
- Is it a CLI app?
    - If so, install it using one of my distroboxes.

I mentioned a script earlier that I use to run commands inside my Distrobox. Well, [here](https://github.com/Aspectsides/dotfiles/blob/main/home/dot_local/bin/executable_run) it is. Now you can just type `run` followed by whatever command you want to run, and it'll run it on my Void distrobox! Make sure your container is named Void, or edit the script to fit your container name. 

Here's the contents of the script if you're too lazy to click the link :D 

```bash
#!/bin/sh

command="$@"
container_name="void"

program=$(echo "$command" | awk '{print $1}')
distrobox-enter "$container_name" -- "$command"
```

Additionally, if I want to test out something inside a container, I just create an *ephemeral* container (usually running Void because it's amazing), and place the home directory of the container in `/tmp/`. This way, the rest of the system is not going to be affected, unless something goes *really* wrong.

```bash
distrobox-ephemeral --image ghcr.io/void-linux/void-linux:latest-full-x86_64 --home /tmp/void
```

Now I can tinker away without accidentally deleting systemd (believe me; I'm tempted sometimes).

# Conclusion

So there you have it. Now you can run Linux.... In Linux. It's like a Linux Subsystem for Linux. Enjoy!

# References

- [Distrobox Docs](https://github.com/89luca89/distrobox/tree/main/docs)
- [Void Linux Handbook](https://voidlinux.org/)
- [Jorge Castro's YouTube Video](https://www.youtube.com/watch?v=Q2PrISAOtbY)






