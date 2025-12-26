---
layout: post
title: "[WIP] Stacking chips and CUDA cores"
description: "I desperately needed that warmth from my GPU."
thumb_image: 
tags: [ml]
---

_This is part 3 of the pokeqt series, where I do poker-related things because I liked a girl who do._
{: style="margin-bottom: 0.5em !important; font-size: .9em" }
1. [How [not] to run a Monte Carlo simulation]({{ page.previous.previous.url }})
2. [Control Variates and the Satisfying Telescoping Sum]({{ page.previous.url }})
3. _Stacking chips and CUDA cores_
{: style="font-size: .95em; opacity: 0.9" }

---
{: style="margin-bottom: 2em !important" }

We finally came to the implementation section, woo. The source code for all of these experiments can be found [here](https://github.com/ngoctnq/pokeqt).

## Heads up, analytics!

The first thing we need to do to write an analytical solver for the heads-up setting. Now, I could use a bazillion of existing solutions on the internet, but that's no fun. And, I'm writing it all in branchless logic so that I can later run it on CUDA, because branching ruins warp's parallelism. So here are a couple of interesting functions that I gathered throughout the internet &mdash; note that all of these are done on unsigned integers.

### Get the maximum of two numbers

We will use this to compare the computed evaluations of hands.

{% highlight cpp %}
uint32_t get_max(uint32_t a, uint32_t b) {
  return a ^ ((a ^ b) & -(a < b));
}
{% endhighlight %}

If `a < b`, `-(a < b)` evaluates to -1, which after underflow is all 1s. We then get `a ^ (a ^ b) = b`. Otherwise, `-(a < b)` evaluates to 0, which leaves us with only `a`.

### Keep only the most-significant non-zero bit

This is used to keep only the largest hand (e.g. top flush).

{% highlight cpp %}
uint16_t keep_top_bit(uint16_t num) {
  num |= num >> 1;
  num |= num >> 2;
  num |= num >> 4;
  num |= num >> 8;
  return num - (num >> 1);
}
{% endhighlight %}

Basically, the chain of `or`s will make every bit after the most-significant non-zero bit to be 1, hence the name. Then, we shift it right by 1, meaning now it has one less 1 bit, and remove that smeared suffix from the full smear. Voila, only the MSB is left. You can add another `num |= num >> 16;` if you want this to work with 32-bit.

_I might be the only one who finds the name bit-smearing kinda funny._

### _Bonus:_ Check if number is 0

This can be used if you don't trust the compiler to not branch, or to produce a 0/1 when doing a boolean comparison. But I'm learning to have more faith...

{% highlight cpp %}
uint32_t is_all_zeros(uint16_t x) {
  uint32_t y = -x;  // Zero: MSB=0, non-zeros: MSB=1
  y = (~y) >> 31;   // Zero: MSB=1, non-zeros: MSB=0
  return y;         // Zero: 1, non-zeros: 0
}
{% endhighlight %}

### Put it all together
The rest of the code is just boring branchless arithmetics, where I abused the boolean-to-numeric pipeline to death. You can read through the source code in the link above. I even included my debugging code that prints things out as a Christmas gift too.

## Roll the dice, baby
With all the above helpers, we can now start moving towards writing a Monte Carlo code to estimate these probabilities.

## _Bonus:_ Dress the part
Well, can't be a complete project without the GUI showing live-update! &mdash; said my perfectionism. So I wrote a GUI that fetches the server and display live estimate update. After terminating the simulation, the result is stored in a JSON file that is served statically. The source code is also in the repository above, and you can check the site out at [this link](https://ngoc.io/pokeqt).

Thank you for reading this far! Have fun, and gamble responsibly.