---
layout: post
title: "Problems and Solutions for installing Windows with OpenCore Legacy Patcher"
description: "Just my experience trying to install Boot Camp on a Legacy Mac."
thumb_image: 
tags: [misc]
---

After a day trying to install Windows on my computer, I decided to finally start my blog with a post containing two big issues I met along the way. The solutions are readily available on the internet, albeit very obscure -- hence I'm writing this post, hoping to increase visibility and SEO-friendliness of these solutions.

*You can skip the lengthy intro section if you just want to find out what the problems were and how to fix them.*

## TL;DR:
- *Problem 1:* **Windows cannot be installed to this disk. The selected disk has an MBR partition table. On EFI system, Windows can only be installed to GPT disks.**

    Follow this [instruction](#tldr-solution).

- *Problem 2:* **Boot Camp requires that your computer is running Windows 7.**

    Follow this [instruction](#tldr-solution-1).

## Intro

I'm currently still using my old Mac from 2012. It was top of the line back then, and future-proofing has been nice-ish to me, since I'm still using it to write this post. I'm going to get the new Air 15" soon:tm: though, how soon depending on how much I want to eat noodles 3 meals a day for the next few months.

The reason why I said "nice-*ish*" is that Apple has decided that Macs should only have a lifespan of 7 years, and thus will not provide support for newer versions of Mac. This wasn't a problem until Slack decided to stop working with my old unsupported MacOS, forcing me to find a solution to basically hack my way into a newer version. And that's when I found out and install [OpenCore Legacy Patcher](https://dortania.github.io/OpenCore-Legacy-Patcher/).

Then, on a whim, I decided to install Windows/Bootcamp on this Mac to see how the GT 650M is handling games now (not like I have anything else to play with). Fortunately, OCLP has a guide to [install Windows](https://dortania.github.io/OpenCore-Legacy-Patcher/WINDOWS.html), and even one to [install Boot Camp](https://dortania.github.io/OpenCore-Post-Install/multiboot/bootcamp.html) (note that Boot Camp specifically is the Apple support software for dual-booting Windows on a Mac). Seems simple enough, right?

## Problems and Solutions

It was smooth sailing for the most part, but I encountered two problems, one while installing Windows, and one while I'm installing Boot Camp and the accompanied drivers. The specific error message is provided in the [TL;DR](#tldr) section for ease of Googling.

### Windows not recognizing GPT partition style

Browsing through the guides and Google answers is like following a rabbit hole. I went into Disk Utility on Mac to see if it's partitioned as GPT, and it was. Googling to see if it's an APFS problem, no dice either. After a lot of even more hopeless Googling, I came into this lifesaving [gist](https://gist.github.com/oznu/8796d08d73315483c3b26e79a8e3d350). Essentially, the mistake I made was that I ran the Boot Camp Assistant (tried that before going to the official OCLP guide). Quoting the important bits:

> OSX tried to be helpful by converting our legal GPT disk partition into a hybrid MBR partition, which makes OSX see the disk as GPT and Windows it as MBR. Windows 10 requires a GPT disk when using EFI boot, so we need to revert this change using a tool called GPT fdisk (gdisk).

#### TL;DR solution
1. Download and install [GPT fdisk](https://sourceforge.net/projects/gptfdisk/).
2. Find the device number for the internal hard disk (it's probably `disk0`).
3. Run `sudo gdisk /dev/disk0` in the terminal, replace `disk0` accordingly if needed.
4. Type `p` to view the partition table to verify you're working on the correct disk. If not, type `q` to quit without saving your changes and double check the device number.
5. Type `x` to enter the experts' menu.
6. Type `n` to create a fresh protective MBR. Note that gdisk won't confirm a change; it'll just show you a new experts' prompt.
7. Type `w` to save your changes. You'll be asked to confirm this action. Do so.

### Boot Camp requires Windows 7
Everything works... until you cannot install Boot Camp. Tried Windows 7 compatibility, no dice. It's kind of obvious that it's a problem with the downloaded Boot Camp, but there's no newest version provided on the [official website](https://support.apple.com/downloads/bootcamp) (6.0.6136 for my model, 6.1.x for newer models, and I really doubt Apple would update either the software nor the download site).

Turns out Brigadier is rather outdated and just will not pull the newest (or compatible?) version. The solution is much simpler than the previous one, yet a lot more restrictive: [just go into the Boot Camp Assistant to download the updated software](https://discussions.apple.com/thread/7189652). I wasn't able to successfully run all of the packaged executables, but most is actually good enough. Might be relevant, but I actually ran some of the outdated software from Brigadier before I found the official one, so that might have helped.

Another solution on that link tells you to get the Apple Boot Camp update from the internet and install it (this [link](https://digiex.net/threads/apple-windows-10-bootcamp-6-drivers-download-applebcupdate-exe-april-1st-2016.14828/) seems to agree). I have not tried that (and I don't intend to), but if you don't have a Mac installation that might provide some hope. You can get internet working on your Windows installation by using the drivers included in the Brigadier version.

For completeness' sake, there's also a full list of mirrors for Boot Camp [here](https://www.applex.net/pages/bootcamp/). However, I **strongly** recommend **against** downloading and running potentially compromised software downloaded from untrusted sources, especially with escalated privilege.

#### TL;DR solution
1. Open Boot Camp Assistant.
2. Go to **Action** > **Download Windows Support Software**.
3. Copy the downloaded software to some USB and install them individually on Windows.

## Outro
Well, my Windows installation works now :shrug: Games run slow to a crawl with really low frame rate and jagged graphics and screaming fans, all that good stuff. I immediately went back to using Mac, and eventually wrote this guide.

Here's to another newly-created rarely-updated blog!