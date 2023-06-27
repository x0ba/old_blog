---
title: "Chezmoi; The Best Dotfile Manager"
date: 2023-03-29T18:06:54-07:00
draft: false

author: "Aspect"
authorLink: "https://aspect.is-a.dev"
subtitle: "A feature-rich dotfile manager."
summary: "In this article, I'll go over Chezmoi, an extremely feature-rich dotfile manager that thinks of everything."
description: "Dotfile management with Chezmoi."

tags: ["dotfiles", "chezmoi"]
categories: ["linux"]
keywords: ["dotfile"]

featuredImage: "images/cover-image.png"
featuredImagePreview: "images/cover-image.png"

fraction: true
fontawesome: true
linkToMarkdown: true
rssFullText: false

toc:
  auto: true
---

# Introduction

## A History of Dotfile Managers

Ever since I began using arch, I've been searching for a good way to manage my dotfiles. I started out by just `git init`ing my entire home directory and putting any files that I wanted to exclude in `.gitignore`. This was horrible. I found myself accidentally pushing secrets, sensitive documents, and copyrighted content just because I forgot to put those files in my `.gitignore`. Furthermore, it cluttered up my home directory with version control files, and managing the git repository just became a tedious, unrewarding task. So, I tried something different. Instead of maintaining my entire home directory as a git repo, I put all my dotfiles into `~/.dotfiles`, and then individually symlinked them to where they needed to be. This was also extremely tedious, and the mental overhead of adding new dotfiles was excrutiating.

That concludes the saga of *agonizing* dotfile management. The way I had been managing my dotfiles for the longest time was with a git bare repository, in which I `git init --bare`'ed a git repository with `~` as the repo's worktree, and added files and folders into the repo with `git --git-dir=$HOME/dotfiles --work-tree=$HOME add "file name"`, which I had bound to a zsh alias. This kind of worked? However, it presented its own problems. If I accidentally added a file I had not meant to, say my `~/.ssh` folder, there *was no graceful way to remove it from the git bare repo*. This caused its own frustrations, and there was also the conundrum of the git repo inside a git repo. If I had something like tpm downloading my plugins for tmux, git would complain about having a git repo inside a git repo and the files just would not show up in the git repo. Additionally, there was no good way to manage secrets. For example, I use spotify-tui and spotifyd to listen to Spotify. Those folders contain my Spotify password and API key, which I do *not* want to get leaked to the public. There is no good way to manage these secrets, so I just excluded them from my dotfile management system.

## A Breath of Fresh Air

And along comes Chezmoi. I first heard of Chezmoi from the Catppuccin Discord server, being described as "the best dotfile manager." At that point, I was starting to get fed up with traditional methods of dotfile management, so I thought, what the heck. **Let's give it a shot**. 

Trying Chezmoi was like taking a breath of fresh air. And no, *I'm not exaggerating*. It seemed as if the devs had thought of everything, from secret management to file encryption to external files. It's been amazing, and you can find my Chezmoi dotfiles over on my [Gitea](https://git.onefractal.tech/aspect/dotfiles), if you want to use them for reference. Let me walk you through some features that genuinely make me believe that Chezmoi is the best dotfile manager.

# Features

## Basics

To start using Chezmoi, you first need to install it. It should be in the Arch Linux community repository, so you can install it with a simple

```shell
sudo pacman -S chezmoi
```

Once Chezmoi is installed, you can initialize a directory in `~/.local/share/chezmoi` by running

```shell
chezmoi init
```

This directory will serve as a versioned folder with your dotfiles in it, and will be what you check into a version control system such as **git**. 

Add any file in your home directory with `chezmoi add`. In this example, I'll be adding my `.config/sway`.

```shell
chezmoi add ~/.config/sway
```

This will add your sway config to your chezmoi directory. Let's cd over there and see what we find.

```
$ chezmoi cd
$ tree
.
└── dot_config
    └── sway
        └── executable_config
```

You can probably guess from this the basic naming conventions that Chezmoi uses. Instead of `.config`, `dot_config` is used. Based on whether a file is executable or not, `executable_` will be added in front of the file name. These file naming conventions are not that important to know in my experience, but it does add a bit of clarity as to the type of file you're looking at when you're jumping around in your Chezmoi directory. Once you've added a couple of your files, you can edit the files in your Chezmoi directory by running 

```shell
chezmoi edit $PATH_TO_FILE
```

Note that you pass in the path to the actual file, *not* the path to the Chezmoi version. This does take some getting used to, but it becomes second nature after a very short period of time. Once you're done editing your Chezmoi files, check the changes with `chezmoi diff` and apply with `chezmoi apply`.

<script async id="asciicast-c2zRUj6l5ZeKoRDfVq5YrR5er" src="https://asciinema.org/a/c2zRUj6l5ZeKoRDfVq5YrR5er.js"></script>

## Encrypting Files

One of the killer features of Chezmoi is, in my opinion, the ability to quickly and easily encrypt and decrypt dotfiles. I'd like to make my dotfiles public, and also include my `.ssh` folder for easy migration between computers, but I obviously do not want my secret key to get leaked. That's where Chezmoi file encryption comes in. Please do consult the [docs](https://www.chezmoi.io/user-guide/encryption/) for a much more in-depth and detailed explanation on dotfile encryption, but I'll do my best to give you a quick rundown. Chezmoi supports encryption with both age and pgp. I personally use age because it reminds me of my mortality, but you can use either. Both are (basically) impossible to crack. 

You need to first generate an age key with 

```shell
age-keygen -o ~/.config/key.txt
```

You can move the `key.txt` file wherever you want. Catting it out results in something like this

```txt
# created: 2023-03-29T23:46:27-07:00
# public key: age1q4ucvhhl4a3r6ccfvhp5uk0r5m00u5shpm29j6ve0wu9qz47x30qwmec4l
AGE-SECRET-KEY-10XW36FKPAFGPJCT8Z0VMKQV7AS3U2NX69NT68CZYDP0VTFW83DTQ2JUPE3
```

Take note of the public key. That's important. *Make sure you never share your secret key*. Now you can create a file at `~/.config/chezmoi/chezmoi.toml` and add this to it

```toml
encryption = "age"
[age]
    identity = "/home/user/.config/key.txt"
    recipient = "PUBLIC_KEY"
```

Replace PUBLIC_KEY with the public key from your `key.txt` file. For every new computer, you'll need to generate a new age key and add that to your `chezmoi.toml`.

To add an encrypted file, run 

```shell
chezmoi add --encrypt /path/to/file
```

After doing this for my .ssh directory, my `~/.local/share/chezmoi` now looks something like this

```shell
.
├── dot_config
│   └── sway
│       └── executable_config
└── private_dot_ssh
    ├── encrypted_config.age
    ├── encrypted_deploy.pub.age
    ├── encrypted_id_rsa.pub.age
    ├── encrypted_id_vpn_aspect.age
    ├── encrypted_private_deploy.age
    ├── encrypted_private_id_rsa.age
    ├── encrypted_private_known_hosts.age
    └── encrypted_private_known_hosts.old.age
```

Notice how these files are all encrypted with age and have `encrypted_` in front of their filenames. If I cat out, say `encrypted_config.age`, I'll get a load of gibberish that cannot be decrypted without my personal age key.

```age
-----BEGIN AGE ENCRYPTED FILE-----
YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUxOSBLcnpwQktSS2k4aXMwUFlv
SUZJNjVOZjRLd3FvaXpiQUNTOFlwMzg4MWhjCkVMZ3dvZEVlbXNEK3ZXWGl6dDFB
b09vUllYamhlTDZVbHFkVzlSdmQ3b1EKLS0tIG9WcGR2dStNRnlIMW5zdzc1R2VJ
bFpzUlphOUVJeldKRDJMckNIZ3JkeFUK42D1vjOFF3lwD9FlJOIyBKRVsB9ie1Bk
lNVRbYgEnFS1Z4QPTmCezfLy58DQbE6O2IjczOzt4fNppY6KNVAS+19BdpL28vQV
tSU5OxGXHseuNWyrlt6LhrgXPySn5Fu3QkoZPth7YHqlExOzsZ9OsA7h5SF6hwAh
dfmeKxWKapZTzwBbu8/bzq1hZBb59wY1G/G7ILFmkIKebYc0h9iifL/LXVi3eJHs
sej4lcPopOp4HmyZILaF4FvMGncasPp9ViA/9mv90dKjHGjny/zS8xe/BiBhoahl
BOO4/eUEUIimTDhgzQiZkgivxGsJIMiPfzhLJFKzvWLvas9J
-----END AGE ENCRYPTED FILE-----
```

Perfect encryption.

## Bitwarden Integration/Templates

Let us now dive into the realm of templates. Let's say I want to share my `.irssi/config` file, but I don't want to publicize my irc password, which is stored in plain text in the config file. I still want the public to have access to the rest of the file. What do I do? Thankfully, Chezmoi has a useful little feature called templates. Chezmoi templates are a huge rabbit hole, and covering them all would take forever, so I'll just cover the only one that I use; Bitwarden. Once again, check the official [docs](https://www.chezmoi.io/user-guide/templating/) on this topic, as they will be able to explain this much better than I would be able to.

Start by adding a dotfile as a template. Run

```shell
chezmoi add --template /path/to/file
```

I'll add my `.irssi/config` file. When I edit the template file with `chezmoi edit .irssi/config`, my irc password is just sitting there in plain text. 

```
servers = (
  {
    address = "tilde.chat";
    chatnet = "tildechat";
    password = "supersecretpassword";
    port = "6697";
    use_tls = "yes";
    tls_verify = "no";
    autoconnect = "yes";
  },
  {
    address = "irc.libera.chat";
    chatnet = "liberachat";
    password = "supersecretpassword";
    port = "6697";
    use_tls = "yes";
    tls_verify = "no";
    autoconnect = "yes";
  }
);
```

We can't have that. Thankfully, I have my IRC password stored on my self-hosted Bitwarden server, so I can make a template that'll use Bitwarden to retrive my password every time I apply my dotfiles. First set up bitwarden-cli by following [this](https://bitwarden.com/help/cli/) guide. Once you have that set up, a nice premade template should suffice. 

```tmpl
{{ (bitwarden "item" "example.com").login.password }}
```

Replace "example.com" with the name of the item in your Bitwarden vault. In my case, it'll look something like this

```tmpl

servers = (
  {
    address = "tilde.chat";
    chatnet = "tildechat";
    password = "{{ (bitwarden "item" "irc").login.password }}";
    port = "6697";
    use_tls = "yes";
    tls_verify = "no";
    autoconnect = "yes";
  },
  {
    address = "irc.libera.chat";
    chatnet = "liberachat";
    password = "{{ (bitwarden "item" "irc").login.password }}";
    port = "6697";
    use_tls = "yes";
    tls_verify = "no";
    autoconnect = "yes";
  }
);
```

Now every time I run `chezmoi apply`, Chezmoi will automatically pull my irc password from Bitwarden and replace the template with it.

## External Git Repos

One issue that has plagued me for the longest time with other dotfile management systems is how they handle git repos. I use tmux, and manage tmux's plugins with [tpm](https://github.com/tmux-plugins/tpm). The problem is that this requires a git repo to be cloned into your `.config/tmux` folder, which causes issues with putting that folder inside a git repo. Chezmoi makes this very easy. Simply `chezmoi cd` into your Chezmoi directory and create a new file called `.chezmoiexternal.toml`. In that folder, put something like this

```toml
[".config/tmux/plugins/tpm"]
    type = "git-repo"
    url = "https://github.com/tmux-plugins/tpm.git"
    refreshPeriod = "168h"
```

Inside the square brackets put the path you'd like the git repo to be cloned to. Replace the url field with whatever repo you'd like to be cloned. Now when you run `chezmoi apply`, it'll clone the Github repo into the directory specified. Note the refresh period, which in this case is 168 hours. `chezmoi apply` will run a `git pull` on the repository once every 168h, which makes it easy to keep your git repo updated.

<script async id="asciicast-DDre1EDPcbGknHUe83gPgmKXm" src="https://asciinema.org/a/DDre1EDPcbGknHUe83gPgmKXm.js"></script>

# Conclusion

Other people hop distros, I hop dotfile managers. Chezmoi is the first dotfile manager I've been truly satisfied with, and I'll continue using it until I inevitably get sucked into the Nix rabbit hole. God help me.
