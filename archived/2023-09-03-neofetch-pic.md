---
layout: post
title: "neofetch on Mac and iTerm doesn't show picture?"
description: "More problems, more solutions, more me being just dumb."
thumb_image: 
tags: [misc]
---

This is going to be short, since it's rather simple, yet I couldn't find a fix on the internet.

## The Problem
Installed `neofetch` from Homebrew and iTerm. Per recommended, I ran

{% highlight bash %}
neofetch --iterm --source some_pic.png
{% endhighlight %}

And to my surprise, no images shown on the terminal! Note that I'm running on iTerm, which can show images on its own.

## The Debug

Googling actually doesn't help this time. I'll sum up some notable ones:
- [This link](https://github.com/dylanaraps/neofetch/issues/1053#issuecomment-406000785) tells you that some versions of `neofetch` fails to show image, and recommends you install the 4.0.1 release.
    
    Thing is, the newest version is 7.1.0 already! And I even tried installing that specific version from source just to encounter the same problem.

- [This link](https://github.com/dylanaraps/neofetch/issues/1644#issuecomment-848917411) tells you to tweak a few options in iTerm, specifically:
    - Go to **Preferences > Advanced**
    - Disable **Disable potentially insecure escape sequences** and **Show inline images at retina resolution**.
    
    I did that too. I actually never changed it back to the default values even after getting this resolved, since it made sense.

Then I went across some link on the Issues tab of the repo, which some guy asked OP to post his debug log. So I tried it:

{% highlight bash %}
neofetch --iterm --source some_pic.png -vv 2
{% endhighlight %}

Lo and behold:

{% highlight bash %}
[!] Image: Imagemagick not found, falling back to ascii mode.
{% endhighlight %}

## The Solution

{% highlight bash %}
brew install imagemagick
{% endhighlight %}

It's that simple. I don't know why it wasn't listed in the build script, but the author did specifically mention that `neofetch` uses ImageMagick for thumbnail creation.
Dunno how to contribute changes to the Ruby build script, so if anyone can do that or tell me how to, that'd be great. Please?

## P/S: Interaction with `powerlevel10k`

This is rather a minor issue, but escape sequences get messed up if you try to change (`clear`/`echo`) the terminal output after the Instant Prompt script.
The solution is also simple, just move the `neofetch` invocation to the top of `.zshrc` (or anywhere above the Instant Prompt script -- I haven't tested that, so YMMV).

Everything works now, hope this helped someone!

***P/P/S:** I got a new MacBook Air 15 with upgraded specs! Took like a week to arrive, but still sooner than expected. Apple please don't release a new one with M3 this Fall or my buyer remorse is going to eat me alive. Yes, I know that future-proofing is overrated, but my modest PhD stipend won't let me buy things at ease.*