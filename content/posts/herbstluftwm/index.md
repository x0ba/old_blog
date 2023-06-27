---
title: "Herbstluftwm"
subtitle: "My new favorite window manager."
date: 2023-06-15T10:21:26-07:00
lastmod: 2023-06-15T10:21:26-07:00
draft: false
authors: ["aspect"]
description: "Herbstluftwm is a very neat window manager."

tags: ["linux"]
categories: ["linux"]

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: "images/states.png"
featuredImagePreview: ""

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---

# Introduction

Recently, I've switched from dwm and bspwm to a window manager that's not as well known; Herbstluftwm. That's a really awful name. Despite it's mouthful of a name, it's a very neat window manager that's both easy to configure and very extensible. In this article we will be looking at how herbstluftwm functions and sharing some configuration tips.

{{< admonition >}}
Herbstluftwm is both too hard to pronounce and too hard to type, so I'll be referring to it as hlwm for the rest of the article.
{{< /admonition >}}

## About Herbstluftwm

Herbsftluftwm handles windows in a unique way. The tiling algorithm is based on splitting frames into subframes which can then be split again or filled with windows. Think of tiling a new window like putting a picture frame on a wall, then putting a picture in it. Tags, or workspaces, can be added or removed at runtime, and each tag contains it's own frame/subframe. Similar to bspwm, herbstluftwm is configured at runtime through ipc calls from a utility called `herbstclient`. These combine together to form a very flexible and customizable window manager.

## Monitors

Monitors in hlwm are very easy to set up, unlike some other window managers. The screen space can be freely divided into "monitors" which may or may not match the actual (multi-)monitor hardware configuration. If you have a monitor with a large resolution you can just divide it into two or more virtual monitors such that you can view two virtual desktops at a time. Monitors can be automatically setup with a simple `herbstclient detect_monitors`.

## Monitor-workspace relation

All monitors share the same pool of tags, or workspaces. Exactly one tag can be viewed on each monitor, and tags are monitor-independent, similar to Xmonad.

> Workspaces are called tags in hlwm. They are used interchangeably. You will see both of them appear in this article.

## Configuration

Configuration is done through the `~/.config/herbstluftwm/autostart` file. This file is just a shell script that calls `herbstclient` to configure the window manager, as well as anything else you might want to start at runtime. The most important things you should do here before you launch the window manager for the first time include creating tags and setting keybinds. In order to make configuration easier, I recommend creating a function like this:

```bash
hc() {
	herbstclient "$@"
}
```

This way, you can just type `hc` instead of `herbstclient` every time.

### Tags

To create tags, you can either call `herbstclient add NAME_OF_TAG`, or do something like this:

```bash
# tags
tag_names=({1..9} 0)
tag_keys=({1..9} 0)
# Need this before tag creation
hc set default_frame_layout 'max'
hc substitute ALGO settings.default_frame_layout \
	foreach T tags.by-name. \
	sprintf ATTR '%c.tiling.root.algorithm' T \
	set_attr ATTR ALGO

hc rename default "${tag_names[0]}" || true
for i in "${!tag_names[@]}"; do
	hc add "${tag_names[$i]}"
	key="${tag_keys[$i]}"
	if [ -n "$key" ]; then
		#hc keybind "$Mod-$key" use_index "$i" #Focus tag at number
		hc keybind "Mod4-$key" substitute PRE tags.focus.index chain + use_index "$i" + or , compare tags.focus.index != PRE , use_previous #Focus tag at number
		hc keybind "Mod4-Shift-$key" move_index "$i"                                                                                        #Move focus window to tag at number
	fi
done
```

This is my personal preference, as it creates 10 tags, and sets the default layout to max, which creates tabs whenever one opens a new window instead of tiling. I have something set up that tiles terminal windows when I open them later in the guide, but general I prefer to open windows in tabs.

{{< image
  src="./images/tabs.png"
  title="Tabs"
  caption="Showing window titles a.k.a tabs"
  align="center"
>}}

### Bindings

There are two types of bindings that can be set in hlwm; keychains and keymaps. 

#### Keymaps

Keymaps are just your run of the mill, `Super+KEY` bindings. 

```bash
hc keybind Mod4-Shift-Return spawn kitty -1 #Spawn terminal
```

#### Keychains

Keychains are keychords as seen in Emacs; something like `super+s f`. That is, hit `super+s`, then `f`. They are a bit more complicated to set.

```bash
# Toggle window states
keys=(p f a)

unbind=()
for k in "${keys[@]}" Escape; do
	unbind+=(, keyunbind "$k")
done

hc keybind Mod4-s chain \
	'->' spawn notify-send "hlwm" "Toggle window states (pfa) or press Escape" \
	'->' keybind "${keys[0]}" chain "${unbind[@]}" , pseudotile toggle \
	'->' keybind "${keys[1]}" chain "${unbind[@]}" , attr clients.focus.floating toggle \
	'->' keybind "${keys[2]}" chain "${unbind[@]}" , fullscreen toggle \
	'->' keybind Escape chain "${unbind[@]}" #Keychain for window states
```

First, specify the keys to be used after your initial keymap, which Mod-s in this case, in an array `keys=(YOUR KEYS HERE)`. Then call a function called `unbind` that unbinds the keys after you use it once, thus ending the keychain. Then, you can bind your specified keymap as a chain.

I use the following keychain to spawn my terminal in a bsp-like layout.

```bash
hc keybind Mod4-Return or ',' and '_' compare tags.focus.curframe_wcount = 0 '_' spawn kitty -1 ',' chain '-' split auto '-' cycle_frame '-' spawn kitty -1 #bsp-like spawning of terminal
```

{{< image
  src="./images/bsp.png"
  title="BSP-like window spawning"
  caption="Making use of keychains to spawn terminal windows tiled"
  align="center"
>}}

### Scratchpad

The one thing that I require in a window manager is a scratchpad feature. Hlwm does not have a *built in* scratchpad, but I wrote a handy dandy little script that spawns a terminal scratchpad that can be hidden and opened on any tag.

```hlscthpd.sh
#!/usr/bin/env bash
#set -euo pipefail

geometry() {
	mrect=($(herbstclient monitor_rect ""))

	width=${mrect[2]}
	height=${mrect[3]}
	x=${mrect[0]}
	y=${mrect[1]}
	rect=(
		$(bc -l <<<"$width * 0.7" | cut -d'.' -f1)
		$(bc -l <<<"$height * 0.6" | cut -d'.' -f1)
		#$(($x+$(bc -l <<< "$width * 0.15" | cut -d'.' -f1)))
		$(bc -l <<<"$width * 0.15" | cut -d'.' -f1)
		$((y + 16))
	)
	herbstclient rule once instance=dropdown floating_geometry=${rect[0]}x${rect[1]}+${rect[2]}+${rect[3]}
}

dropdown=/tmp/herbstluftwm:dropdown
if xdo id -n dropdown; then
	if [[ $(herbstclient list_monitors | grep '[FOCUS]' | cut -d'"' -f2) == $(herbstclient attr clients.$(cat $dropdown) | grep 's - - tag' | awk '{ print $6 }' | tr -d \") ]]; then
		xdo hide -n 'dropdown'
		exit
	fi
fi
if [[ -f $dropdown ]]; then
	if ! herbstclient bring $(cat $dropdown); then
		geometry
		xdo show -n 'dropdown' && exit
	fi
fi
if ! xdo show -n 'dropdown'; then
	geometry
	setsid -f kitty -1 --class 'dropdown' -e tmux new -As Dropdown
	xdo id -m -n 'dropdown' | tr "[:upper:]" "[:lower:]" | sed -r 's/0x([0]+)/0x/' >$dropdown
fi
```

{{< image
  src="./images/spad.png"
  title="Hlwm scratchpad"
  caption="Scratchpad implementation"
  align="center"
>}}

### Window States

A window in hlwm can be in one of three states: tiled, pseudotiled, and floating. These are divided into two layers, one with tiled and pseudotiled windows, and the floating layer above the tiled windows. Floating windows can be freely resized/moved around.

{{< image
  src="./images/states.png"
  title="Hlwm window states"
  caption="Floating, Tiled, and Pseudotiled windows"
  align="center"
>}}

Notice how the tiling windows (left and lower right) take up all available screen space while the pseudotiled window (top right) still tiles but does not take up the entire space. Note the frame around the pseudotiled window; that's the hlwm "frame" at work. Even though the window does not take up the volume of the entire frame, that is still where the window is set to spawn.

Here is a keychain that I use to set the window state:

```bash
# Toggle window states
keys=(p f a)

unbind=()
for k in "${keys[@]}" Escape; do
	unbind+=(, keyunbind "$k")
done

hc keybind Mod4-s chain \
	'->' spawn notify-send "hlwm" "Toggle window states (pfa) or press Escape" \
	'->' keybind "${keys[0]}" chain "${unbind[@]}" , pseudotile toggle \
	'->' keybind "${keys[1]}" chain "${unbind[@]}" , attr clients.focus.floating toggle \
	'->' keybind "${keys[2]}" chain "${unbind[@]}" , fullscreen toggle \
	'->' keybind Escape chain "${unbind[@]}" #Keychain for window states
```

## Window Rules

Similar to BSPWM, window rules in hlwm are set using calls from herbstclient. You can set properties such as window state, tag placement, and placement on the screen with rules. Rules can be set according to either window title or class. Window classes are user-specified; launching Alacritty with the `--class` flag is what sets this. Window titles are just the name of the application as specified in its `.desktop` file. Both of these can be obtained using [xprop](https://www.x.org/releases/X11R7.5/doc/man/man1/xprop.1.html).

{{< image
  src="./images/prop.png"
  title="Xprop output"
  caption="The title is given by the WM_NAME field while the class is given by the WM_CLASS field."
  align="center"
>}}

Thus, window rules can be set as such:

```bash
hc rule instance=dropdown floating=on focus=on
hc rule class=Nsxiv floating=on floatplacement=center focus=on
hc rule class=firefox tag=1
```

In hlwm, the class is set with `instance`, while the app name is set using `class`. That's kinda weird.

## References

- https://herbstluftwm.org/
- https://gitlab.com/dwt1/dotfiles/-/tree/master/.config/herbstluftwm
