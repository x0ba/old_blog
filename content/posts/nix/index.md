---
title: "Nix"
date: 2023-05-11T17:26:12-07:00
draft: true
---

Hey there. Recently, I've been playing around with the Nix Package Manager, a tool that allows you to make reproducible environments declaratively. If those words are just a load of jargon to you, don't worry. They are to me to. I'll explain them in detail a bit later.

## Installing Nix

Installing Nix is relatively easy. You can either go with a system-wide installation that requires `sudo` privileges, systemd, and a [disabled SELinux](https://www.ibm.com/docs/ja/ahts/4.0?topic=t-disabling-selinux).
```bash
$ sh <(curl -L https://nixos.org/nix/install) --daemon
```

Or you could go with a single-user install that will install Nix for only the user you're logged in as. Either one is completely fine for the purposes of this guide.

```bash
$ sh <(curl -L https://nixos.org/nix/install) --no-daemon
```

Run through the installation process, making sure to give the installer `sudo` privileges so that it can create the `/nix` folder, where everything that is installed with Nix goes.

## Nix Basics

Now that Nix is installed, you should have access to the commands. Try running this:

```bash
nix-shell -p cowsay lolcat
```

This will open up a *shell environment* with `cowsay` and `lolcat` installed. You probably don't have these installed already.

```
$ cowsay hello
The program ‘cowsay’ is currently not installed.

$ echo please | lolcat
The program ‘lolcat’ is currently not installed.
```
Once you're inside the nix environment, you can now use cowsay and lolcat. Once you exit the environment by either typing `exit` or pressing `ctrl-d` and try to run either of those two commands again, it will not work. This can be useful if you have a command that you need to run once but don't have on the system, and you don't want to go through the trouble of installing it. I have a handy little script that allows you to type `run` followed by the command that will open a Nix shell with that command and run it, installing it if necessary.

```bash
#!/usr/bin/env sh

command="$@"
program=$(echo "$command" | awk '{print $1}')

if [ -n "$NIX_PKG" ]; then
  program="$NIX_PKG"
fi

nix-shell -p "$program" --run "$command"
```

This is all very useful, but what does this all have to do with declaratively generating reproducible environments? Well, with something like this, a `shell.nix` file.

```nix
{ pkgs ? import (fetchTarball "https://github.com/NixOS/nixpkgs/archive/06278c77b5d162e62df170fec307e83f1812d94b.tar.gz") {} }:

pkgs.mkShell {
  buildInputs = [
    pkgs.which
    pkgs.htop
    pkgs.zlib
  ];
}
```

Create this at the root of whatever project you want to use the environment on. This code looks a bit confusing, so let me break it down for you.

```nix
{ pkgs ? import (fetchTarball "https://github.com/NixOS/nixpkgs/archive/06278c77b5d162e62df170fec307e83f1812d94b.tar.gz") {} }:
```

This [pins](https://nix.dev/reference/pinning-nixpkgs#ref-pinning-nixpkgs) a `nixpkg`, basically a repository Nix downloads the binaries for your programs from. Specifying a specific version of the nixpkg helps with reproducibility, as the version stays the same no matter what you run this on.

```nix
pkgs.mkShell {
  buildInputs = [
    pkgs.which
    pkgs.htop
    pkgs.zlib
  ];
}
```

This makes the packages `which`, `htop` and the development library `zlib` available inside the nix shell. Pretty simple. Now you can run `nix-shell` inside the directory with the `shell.nix` file inside it, and after Nix installs the required packages it'll boot you into a nix shell with htop, which, and zlib installed. Pretty handy,right?

There is one with Nix Shell, though. There's no way to manage dependencies with Nix Shell, and so it's not actually completely reproducible. We been scammed. 

## Nix Flakes


### Why use Nix Flakes?

Nix was created for *reproducible* builds, and tries to ensure that two builds of the same project produce an identical result. Unfortunately, nix file evaluation is not nearly as reproducible, and there is also no standard way to compose Nix-based projects. Nix files assume that everything you need is going to be in `nixpkgs`, which is not always the case. If you need a package outside of `nixpkgs`, you'll have to use `fetchGit` or `fetchTarball`, which is a huge pain and hurts reproducibility. Nix flakes solve all of these problems.

### What is a Nix Flake?

Nix flakes are a good way to specify our code's dependencies in a reproducible and declarative way. But what *is* a nix flake? Well, a nix flake is a source tree (such as a fossil or git repository), that contains a file named `flake.nix`. This file provides a standardized interface to Nix packages and modules. The nice thing about flakes is that they can have dependencies on other flakes. To maximize reproducibility (feel like I'm saying that word a lot), a `flake.lock` file is automatically generated to pin down the dependency versions.

### Enabling Them

To get started, we need to enable flakes, since they're an experimental feature at the moment. 

Get the experimental branch of Nix by running:

```bash
nix-shell -I nixpkgs=channel:nixos-21.05 --packages nixUnstable
```

Then edit `~/.config/nix/nix.conf` and add:

```conf
experimental-features = nix-command flakes
```

You've now enabled nix flakes.

### Creating a Basic Flake






