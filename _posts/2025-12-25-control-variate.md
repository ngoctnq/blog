---
layout: post
title: "[WIP] Control Variates and the Satisfying Telescoping Sum"
description: "This is more exciting than the last part...?"
thumb_image: 
tags: [math]
---

_This is part 2 of the pokerqt series, where I do poker-related things because I like a girl who do._
{: style="margin-bottom: 0.5em !important; font-size: .9em" }
1. [How [not] to run a Monte Carlo simulation]({% post_url 2025-12-19-monte-carlo %})
2. _Control Variates and the Satisfying Telescoping Sum_
3. [Stacking chips and CUDA cores]({% post_url 2026-01-01-stacking-chips %})
{: style="font-size: .95em; opacity: 0.9" }

---
{: style="margin-bottom: 2em !important" }

Control variate is a technique to reduce variance in Monte Carlo simulations, by exploiting information on the error of a different known estimate to effectively offset which of the unknown one.

But I'll spare you the boring text, we both had enough of that last time. The gist of this post is that Control Variate is actually _sooooo_ good for poker probability/equity estimation. So here comes the glaze fest in 3 parts.

## So what exactly is a Control Variate?
 
In short, control variate (CV) "corrects" a sample of a distribution being estimated $p(x)$ with another sample of a CV distribution $q(x)$, keeping the estimator unbiased while reducing variance. CV requires:

- The mean of $q(x)$ is known analytically.
- $p(x)$ and $q(x)$ must be highly correlated.

The algorithm goes like this:

1. Sample $x\sim p(x)$
2. Sample $x'\sim q(x)$
3. Augment the original sample by $\bar{x}=x + c(\mathbb{E}[x']-x')$, where $c$ is some hyperparameter.
4. Use $\tilde{x}$ as if it's $x$.

_<sup>Be careful, the signs of $c$ in this text is flipped from the Wikipedia version.</sup>_

By doing some secret magic derivation and just copy-pasting the results from Wikipedia, we get the optimal value of $c$ to be:

$$
c^*=\frac{\text{Cov}(\bar{p},\bar{q})}{\text{Var}(\bar{q})},
$$

where $\bar{p}$ is the empirical estimate of the desired statistics of $p$. With this optimal parameter, we get a new variance for the new distribution of $\tilde{x}\sim\tilde{p}(x)$ as:

$$
\text{Var}(\tilde{p}) = (1 - [\text{Corr}(p, q)]^2)\text{Var}(p).
$$

The more correlated $p$ and $q$ are, the variance of the augmented estimator comes closer to 0.
A special extreme case is when $p$ and $q$ are too similar to the point that they basically become one, then we would have $c^*=1$ and $\text{Var}(\tilde{p})=0$, as the estimator $\tilde{p}$ would just outputs exactly the known $\mathbb{E}[q]$ every time. But that's too trivial... or is it?

## The first great thing about CV is...
...that there is a very obvious choice for $q(x)$, that is, the probabilities for a 2-player (heads-up) game! The logic should be simple enough that you can write a symbolic solver for this problem without the need for MC sampling. And, even if you only get an estimate for the mean of the 2-player distribution, from here on noted as $p_2(x)$, CV still would work well enough, just less effective and without the unbiased guarantee. For the second requirement, it's pretty obvious that the same (canonical) hand in different $n$-player games would have very high correlation. I am not rigorously proving that though, you'd just have to take my word for it.

Then, let's say we're estimating $p_3(x)$. We know that $p_2(x)$ and $p_3(x)$ are highly correlated, so the estimator variance can go down to near zero! That's how magical it is.

> But, that is only for the optimal $c$!

Yes, but we can get a good estimate just by taking the statistics of the current samples. Or, [just set $c=1$ if $p$ and $q$ are close enough](https://www.value-at-risk.net/variance-reduction-with-control-variates-monte-carlo-simulation), in which they are!

So with $c=1$, the current algorithm would go something like this:

1. Sample $x\sim p_3(x)$
2. Sample $x'\sim p_2(x)$
3. Augment the original sample by $\bar{x}=x + (\mathbb{E}[x']-x')$, where $c$ is some hyperparameter.
4. Use $\tilde{x}$ as if it's $x$.

More concretely, (1) we sample a game for the 3-player setting and check to see which hand won, then (2) sample a game for the 2-player setting and also evaluate to see who won, and then finally apply the correction to (1) with (2) and the known win-probability of that hand from the analytic solution of the 2-player setting.

By doing this, we are sampling twice per game, which effectively double the amount of computation needed. That's why CV is usually used when the sampling overhead of $q$ is greatly less than which of $p$, which isn't in our case. But there's another saving grace:

## The second great thing about CV is...
... that we can completely reuse the sampled hands! So, if you're running simulation for all table sizes concurrently, the overhead is actually minimal. Moreover, if you're using the exact same sampled game state, the correlation actually would shoot through the roof. We are not correcting much though.

Since we're not updating much per run, we'd need a really large number of simulations to reach an accurate estimate... But that is only for anything other than poker.

## The third great thing about CV is...
... that you can _cleanly_ bootstrap it across increasing player-count settings to create what's called a multi-level MC. What happens if I apply CV recursively?

## The final algorithm

With all that I have written, I guess it's obvious now that it's better to just keep the statistics of each hand, and save yourself from all these headaches. Makes it easier to keep track, do control variate, upsampling, everything. So for a $n>2$-player game:

1. Sample the whole game, including hole cards and community cards. Stratified/upsample per canonical hand if desired.
2. Get all players' results from the game simulation.
3. Reuse the sampled game state to evaluate the result as if it's a $n-1$-player game.
4. Do control variate on the $n$-player game result from step 2.
5. Accumulate across all hands.

Then at the end (or any checkpoint), aggregate them by canonical hands. Simple as that.