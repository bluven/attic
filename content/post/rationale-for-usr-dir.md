---
title: "What is the rationale for the `/usr` directory?"
date: 2018-01-20T21:23:57+08:00
draft: true
---

问：
======
What is the rationale for the "unix system resources", or `/usr` directory, as described here, which duplicates many of the directory names under the root directory /?

My purpose: I'm installing Oracle JDK for the umpteenth time and decided this time to just put it under `/home/user` and I'm just reading around a bit to see if it is a bad idea on a single user machine.

答：
=====
There is the short version and the long version of your answer...

Short version:

As your link already said, `/usr` is a place for system-wide, read-only files. So all your installed software goes there. It does not duplicate any names of `/` except `/bin` and `/lib`, but, originally, with a different purpose: `/bin`, `/lib` is only for binaries and libraries required for booting, while `/usr/bin`, `/usr/lib` is for all the other executables and libraries. (now be a good boy and don't ask about `/sbin`, this is the Short Version after all).

Nowadays, the distinction between "required for booting" has diminished, since most modern distros, including Ubuntu, can not properly boot without several files from `/usr`. And that's why there is a strong movement towards merging /usr/bin and /bin, so probably in the near future (Ubuntu 12.10 perhaps?) `/bin` will be a symlink to `/usr/bin`.

But maybe you are confusing `/usr` and `/usr/local`? Because yes, there is (and should be) a lot of duplicated directory names. More on that later...

Long version:

Back in the 70's, in Unix (yes, Unix, way before Linux), floppies had little space (no HD, remember?), and at a given point the system binaries grew too much in number and size to a point they would not fit a single disk, and developers had to split them across several media and thus create new mount points for them. `/bin` filesystem was full, so they installed the new binaries at... `/usr/bin`. And `/usr` was, at that time, their... user directory!

After the (almost embarrassing and often told as a joke/lore) split happened, they started to create "artificial" justifications (and criteria) to decide what would go to `/bin` and what would go to `/usr/bin`. The informal rule was: "essential" stuff go to  `/bin`, "the rest" goes to `/usr/bin`. Same with `/lib`. It was not long before `/usr` became crowded with system-related dirs, mixed with user dirs. Thus `/home` was born, to keep all user-related dirs and keep `/usr` clean for system "stuff" only.

This was long before FHS exist. When it was created, it embraced (and formalized) the current tradition and kept the name `/usr`, although at that time it already had nothing to do with "user" anymore. So yes, the fancy names "U NIX s ource r epository" or "U NIX s ystem r esources" are all made-up names, and it's too late to rename it anyway. (but not too late to merge `/bin` to it)

"Ok, what about `/usr/sbin`?", you ask. Damn, I was hoping you had forgotten. Ok... `/usr/sbin` is for commands that can only be (or is only meaningful when) executed by the root user, like mount and fdisk.

"But isn't that almost the same as `/bin`?". Yes, sure, but...

"Wait, then why there is a `/sbin` too? Doesn't make any sense!". Well, that's because... err.. humm..

Look, a 3-headed monkey behind you!

Ok, hopefully you got distracted enough. Moving on...

(if you think I'm cheating, yes, you're correct. But so is the "official" answer "essential commands that can only be executed by root and must be available before you even mount `/`"). Truth is: the line is indeed blurry, and there is a lot of legacy names that just "stuck" and now it's quite hard to get rid of.

Nowadays

Currently, regarding install directories, your the best way to understand is to think this way:

 - /usr - all system-wide, read-only files installed by (or provided by) the OS

 - /usr/local - system-wide, read-only files installed by the local administrator (usually, you). And that's why most directory names from /usr are duplicated here.

 - /opt - an atrocity meant for system-wide, read-only and self-contained software. That is, software that does not split their files over bin, lib, share, include like well-behaved software should.

 - ~/.local - the per-user counterpart of /usr/local, that is: software installed by (and for) each user

 - ~/.local/opt - the per-user counterpart of /opt

So where to install software?

The above list is already half the answer of you Oracle JDK question, at least it gives several clues. The checklist to "Where should I install software X?" goes by:

 - Is it a completely self-contained, single directory software, like Eclipse IDE and other downloaded java apps, and you want it to be available to all users? Then install in `/opt`

 - Same as above, but you don't care about other users and I want to install for your user alone? Then install in `~/.local/opt`

 - Its files split over multiple dirs, like `bin` and `share`, like traditional software compiled and installed with `./configure && make && sudo make install`, and should be available to all users? Then install in `/usr/local`

 - Same as above, but for your user only? Then install in `~/.local`

Software installed by the OS, or via package managers (like Software Center), and, most importantly, which any local modification might be overwritten when update manager upgrades it to a new version? It goes to `/usr`

Notes:

This explains why default install prefix for compiled software is `/usr/local`, and why you should change it to `./configure --prefix=$HOME/.local` when installing software for your own user only

You may have noticed that all above directories are read-only (except, of course, when you install/remove software). The writable files (like config files) usually go to `/etc` (for system-wide software) and `~/.config` (for per-user settings). Although many legacy software (and, unfortunately, some modern too) use `~/.<software-name>`, cluttering your home folder with billions of dirs and files.

`~/.local` and `~/.config` are not part of the FHS specification. FHS does not deal with user's home folder. They are an attempt of XDG, another standards organization geared towards Desktop Environments (like Gnome, KDE and Unity), to try to set some conventions regarding structure of the user's home. Not all software adheres to it (for example, `~/.local/bin` is not in the user's default `$PATH`, while by logic it should), and no user is forced to follow it, but both gain many interoperability benefits if they do.

I hope this helps clarifying things a bit. Feel free to ask anything so I can improve the answer!

(and I also hope purists don't kill me for such an extremely informal language and explanation. It was intentional, and it surely have many inaccuracies, but I do believe it's a good way to make a newcomer have a brief overview understanding about install directories rationales)
