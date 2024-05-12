---
layout: post
title: Kernel panic during system update
date: 2024-05-07
categories: [Linux]
tags: [troubleshooting,arch]
toc: true
published: true
---

## The crash
Who hasn't managed to run a system update without making sure you're charging and running out of battery? It'll happen. It'll suck.
This time though I ran a pacman -Syu and had to leave my laptop for half an hour and came back to a black screen kernel panic.

Great! Just great. Ok, hard reboot and...kernel panic on boot.
Lovely. Just...lovely.


Time to dig out the live ISO usb stick.

## Troubleshooting
fsck the partitions (volgroups) and managed to repair the root fs. So far, so good.
Let's mount up and chroot.

Nope! chroot failed due to input/output error.
Let's check pacman then. DB is locked, so it did indeed panic during the upgrade. pacman.log has a block of ^@...interesting.
Even more interesting is that find /lib64 -size 0 gives a lot of output... glibc is definitely broken.

Fortunately we can set the root directory in pacman, so we don't have to chroot to start repairing. But as the ISO is old I had to init and populate pacman-key first.

But then

> pacman --root=/mnt -S glibc --overwrite "*"

returns a bunch of ldconfig: File <file> is empty, not checked.
Need to update the file database with
> pacman --root=/mnt -Fy

Seems to have done the trick.
Now we can chroot! Let's run a new update. Nope. libpsl.so is also broken, so no pacman for you.

Exit out and run
> pacman --root=/mnt -S libspl.so --overwrite "*"

Chroot in again, and we have pacman!
Now there's bound to be a lot more broken packages, so I did a reinstall of all packages using
>pacman -Syu $(pacman -Qnq) --overwrite "*"

(which ofc could've just been `pacman -Qqn | pacman -S -`)

Which did the trick. Rebooted, and got back in!

## Failed attempts
Easy, right?
Not really... took a few hours of googling, reading forums and arch-wiki on my phone (which is a pain in the rear - if I had been home with my desktop computer it would've been easier).
Tried using pacstrap without success. Took a while to get around the expired PGP-keys too, and had to reboot as the fs was read-only.
Tried reinstalling the packages before chroot, using a shell script with --config and --gpgdir, but without much success.

Tbh, I should've documented every step as I did it, and the above steps may not be 100% accurate as I'm writing from memory lol
That's a tip actually. Just take photos with the phone of every command and important output to be able to retrace your steps.

But in the end, I did manage to avoid a reinstall this time too. It would've been quicker to just save a list of the installed packages and reinstall the root fs, but repairing is so much more satisfying (and I'm glad that I could use the cache and not the terrible connection I have on my phone)

There's nothing like learning by doing, and while it sucks when you're in that situation, the joy of bringing it back to life is wonderful. Also, backups are nice. I didn't need it this time, but regular backing up of at least your home dir really is a good idea.

## Final thoughts
Ever since I started using Arch linux I've seen a lot of people who say they've bricked their system, and that it's a part of being an Arch user, but I don't think that's true. I have screwed up a few times, rendering my system inoperative due to my rookie mistakes, but I've always managed to fix it. There will of course be cases where a system will pretty much be without hope of saving (a recursive chmod of the whole system is not going to be pretty), but I think that most of the time when someone says their system is bricked, they haven't put in an effort to repair it.

I have never had to post a question to do a repair; pretty much all you need is already out there if you search for it. Many times you can find parts of your solution scattered in different places, and you may have to adapt what you find to your situation. The key is to not give up and keep looking. If there's a will, there's a way.
