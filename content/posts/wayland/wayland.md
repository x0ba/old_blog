---
title: "The Benefits of Wayland over X11"
date: 2023-03-14T22:29:22-07:00
draft: false
---

## About Wayland

Wayland is a display server protocol that aims to be a successor to the dominant X11 display server. It is designed to improve the graphical performance and security of graphical user interfaces by providing direct access to the hardware and allowing applications to render graphics themselves rather than relying on a separate server. Additionally, unlike in X11, where the window manager and the compositor are two seperate things, Wayland compositors and window managers are tighly integrated together. Though it has been out for almost a decade, Wayland is still overshadowed by the ever prominent X11, as many users still believe that it is not ready for daily use yet. In my opinion, however, Wayland is more than stable enough for *most* people to use it successfully (not all, though; I'll touch on that later).

### Wayland Compositors

Like I stated above, Wayland's window managers and compositors are tightly woven together. This means that what are referred to as "window managers" on X11 are instead known as "compositors" on Wayland, and they perform the tasks both of managing windows, which is what an X11 window manager would do, and of drawing them on the screen and adding eyecandy effects such as blur and animations, which is what an X11 compositor would do. As Wayland is a relatively new protocol, there are not nearly as many options.

> Wayland is a relatively new thing to me. In this article, I will inevitably interchange window managers and Wayland compositors. Just know that in most cases, I will be referring to Wayland compositors.

The one I will be talking about most is Sway, since that's what I daily drive nowadays. However, there are more options, most of which are still alpha software and are thus relatively unstable. I know Hyprland is a very popular one these days because of the eyecandy animations and blurring. Another one is River, which many people recommended to me when I was on the hunt for a Wayland compositor. I settled on Sway because of its relative stability compared to the other Wayland compositors, as well as it's extreme similarity to i3, allowing me to just use my i3 config to get started.

## Cons

I believe it's important to start with Wayland's shortcomings first. Though none of these really affect nor bother me, some of Wayland's shortcomings *will* be a deal breaker for some.

### Lack of Nativity 

There are a *lot* of applications that do not run natively on Wayland. Several notable examples are the Suckless Terminal, Emacs, and most electron apps. 

> Though most electron apps do not run natively on Wayland out of the box, it is possible to force them to by running them with the `--ozone-platform-hint=auto` flag.

Though these applications do not run *natively* on Wayland, most of them still can be run through an application called XWayland.

#### What is XWayland?

XWayland is a mini X-server for running X11 apps on Wayland. It provides applications access to Xorg libraries, making it possible to run programs that normally would only be able to run on Xorg. Though this works perfectly for the most part, there is one issue: the lack of compatability with fractional scaling. 

#### Issues with XWayland

If you're on a laptop like me, the default 1.0x scale in Wayland is way too small. Setting the scale to 1.5x presents a problem with XWayland, namely bluriness. If you set the scaling to be fractional (non whole numbers), XWayland *will* run into bluriness issues. As I have, up until recently, been using a 10 year old laptop with a terrible display, this does not bother me at all. I do understand if some people are bothered by this, however. If you're one of them, Wayland may not be for you.

### You need to relearn *everything*

On Wayland, take everything you know about X11 and throw it out the window. It's useless to you here. All your .xinitrc's, your .Xresources, your Xmodmap's... They all do not work on Wayland. You'll have to find your own way to replace what these X11 specific things used to do. This might be obvious, but your previous window managers and compositors and panels will not work on Wayland. There are some drop-in replacements, such as Sway, which is basically a direct clone of i3, and Waybar,which requires some configuration but is the closest thing Wayland has to Polybar.

> I lied. Some `.Xresources` settings will still work in Wayland, such as the colorscheme settings, but you need to manually `xrdb -merge` them.

### Customization (or the lack thereof)

Customization on Wayland is more limited than it is on X11. Most stable Wayland compositors do not include blur, animations, drop shadows or rounded corners. The obvious exception to this is Hyprland, but if I ran Hyprland, let's just say... ***laptop go boom***. In addition to this, a lot of well known customization apps suchas eww and Conky do not work without hacks on Wayland, which makes it difficult to achieve the same level of customization as you would otherwise be able to get on X11.

## Pros 

Let's finally get to the meat of this article; the benefits of using Wayland over X11. 

### No Screen Tearing

One thing that has plagued me about X11 is the screen tearing and cursor flickering. It's really weird to be scrolling through an article and suddenly half of the page scrolls at a different speed than the other half. It's also really weird to see the mouse cursor spazzing out when I'm not moving it. Wayland solves both of these issues because of its nature of tight integration between the window manager and the compositor. This causes V-sync to be enabled all the time, which completely fixes the screen tearing and makes Wayland buttery smooth.

### Better Performance

Wayland provides smoother graphics rendering, faster window resizing, and reduced input latency compared to X11. This is because Wayland handles graphics rendering and compositing on the GPU, which can improve performance and reduce CPU usage.

### Improved security

Wayland has a more secure design than X11, as it restricts applications from accessing other windows or the system's input devices. This prevents malicious applications from spying on user activities or injecting harmful input events.

## Conclusion

In conclusion, the decision to switch to Wayland ultimately depends on your specific needs and preferences. While Wayland offers many benefits such as improved security and performance, it may not be the best option for everyone. However, if you are open to trying new things and are looking for a modern display server, then Wayland is definitely worth a try. It's always a good idea to experiment with different technologies to find the one that suits you best.
