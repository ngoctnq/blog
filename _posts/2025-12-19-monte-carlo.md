---
layout: post
title: "[WIP] How [not] to run a Monte Carlo simulation."
description: "Wow I suck at probabilities."
thumb_image: 
tags: [math]
---

<!-- Original started date: 2025/09/26. -->

As usual, I was procrastinating my real research work by doing something irrelevant; this time it's equity calculator for a poker hand, assuming no betting actions are taken at any points after that. The plan has always been simple: sample hands, sample community cards, evaluate win/loss, and take the average. The execution though...

## The problem

Just to simplify, we are only concerning with the win/loss/tie probability. So, in a $n$-player game, given that you have seen your hole card and nothing else, what is the probability that you will win when the dust settles?

From this point on, assume that we have a function that evaluates the win/loss/tie probability given a set of hands, that bruteforce through all possible combinations of community cards. It takes 30 seconds for one such run on my dated MacBook Air with single core and no PyPy, so it's fast enough™️ in tandem with parallelization. Now, the Monte Carlo estimation is being averaged across sampled possible starting hands.

## The surefire way
Intuitively, just sample all other hands, run the simulation, and average outcomes. Simple, right?

Well, yeah. To put it rigorously, let $\mathcal{H}=\\\{H_0,\dots, H_{n-1}\\\}$ be the starting hands, and \\(H_i\\) are the hole cards of the \\(i\\)-th player. Let \\(H\\) (with no suffix) be any particular hand we're trying to evaluate, and \\(Y_0\\) is a Boolean variable showing whether the first player won — excuse my 0-indexing. Then we have:

$$
\begin{align*}
\mathbb{P}[Y_0\,\vert\, H_0=H]
&= \sum_\mathcal{H}\mathbb{P}[Y_0,\mathcal{H}\,\vert\, H_0=H] \\
&= \sum_\mathcal{H}\mathbb{P}[Y_0\,\vert\,\mathcal{H},H_0=H]\times\mathbb{P}[\mathcal{H}\,\vert\,H_0=H] \\
&= \mathbb{E}_{\mathcal{H}\,\vert\,H_0=H}\mathbb{P}[Y_0\,\vert\,\mathcal{H},H_0=H].
\end{align*}
$$

This is just mathspeak for "sample hole cards such that the first hand is what we want, then count many times the first hand won." Also functionally equivalent to "only sample the other players' hands," i.e. just \\({\mathcal{H}_{\backslash 0}} = \mathcal{H}\setminus\\\{H_0\\\}\\).

## The shortcut
With that approach, we have to do the whole song and dance above for each hand independently. Imagine we can accumulate counts for other hands in the above simulation! So here is what I did in pseudocode:

1. Sample the whole set of hole cards \\( \mathcal{H}=\\\{H_0,\dots, H_{n-1}\\\} \\)
2. Get all results from the game simulation \\( \mathbf{Y}=\\\{Y_0,\dots, Y_{n-1}\\\} \\)
3. For each hand, update the win/loss/tie statistics.

This would still work nicely, because everything is unique. Any hand in \\(\mathcal{H}\\) can be treated as index 0, and the full sampling process can be viewed as a 2-step process where one samples \\(H_0\\) first before the rest. If we condition the second step of sampling on the first, that is equivalent to doing one simulation run for a (randomly) instantiated \\(H_0\\).
I handwavied a little glossing over how the priors for all hands are identical, and how the hand positions are permutation-invariant, but those minor details don't affect the analysis — as per mathematician's tradition, the rest is left as an exercise to the reader.

## The canonical representation
But when things stop being unique is when all the troubles started. See, what we want is not the actual hand — that is just for fancy analyses and real-time TV annotations. What we really want is *canonical forms*, that is, instead of `AdKd` (Ace-King of Diamond), we work with `AKs` (Ace-King suited), which actually is a *set* of possible hands (`AcKc`, `AdKd`, `AhKh`, `AsKs`). This canonical form lets us work with only 169 possibilities, instead of the \\(52C2=1326\\) that only someone like Sheldon Cooper can remember.

Now, what I should have done is that I should have gone for the approach above storing all full hands, then aggregate them later for the canonical version. This is perfectly fine, as we'll see now: let \\( \mathcal{C}=\\\{C_0,\dots, C_{m-1}\\\} \\) be a canonical hand and \\( C_i \\) be the actual hands that satisfy that canonical representation. We have:

<!-- $$
\begin{align*}
\mathbb{P}[Y_0\,\vert\, H_0\in\mathcal{C}]
&= \mathbb{P}[H_0\in\mathcal{C}\,\vert\, Y_0]\times \mathbb{P}[Y_0] / \mathbb{P}[H_0\in\mathcal{C}] \\
&= \sum_{i=0}^{m-1}\mathbb{P}[H_0=C_i\,\vert\, Y_0]\times \mathbb{P}[Y_0] \;{\Large /}\; \sum_{i=0}^{m-1}\mathbb{P}[H_0=C_i] & \text{(}C_i\text{ are uniques }\textit{as a pair}\text{)} \\
&= \sum_{i=0}^{m-1}\mathbb{P}[H_0=C_i\,\vert\, Y_0] \;{\Large /}\; \sum_{i=0}^{m-1}\mathbb{P}[H_0=C_i] \times \mathbb{P}[Y_0]\\
&= \frac1m\sum_{i=0}^{m-1}\frac{\mathbb{P}[H_0=C_i\,\vert\, Y_0]}{\mathbb{P}[H_0=C_i]} \times \mathbb{P}[Y_0] & \text{(}C_i\text{ are equally probable)}\\
&=\frac1m\sum_{i=0}^{m-1}\mathbb{P}[Y_0\,\vert\, H_0=C_i].
\end{align*}
$$ -->
$$
\begin{align*}
\mathbb{P}[Y_0\,\vert\, H_0\in\mathcal{C}]
&= \sum_{i=0}^{m-1}\mathbb{P}[Y_0\,\vert\, H_0=C_i]\times\mathbb{P}[H_0=C_i\,\vert\, H_0\in\mathcal{C}] \\
&=\frac1m\sum_{i=0}^{m-1}\mathbb{P}[Y_0\,\vert\, H_0=C_i]\times \frac1m & \text{(}C_i\text{ are equally probable)}\\
&=\frac1m\sum_{i=0}^{m-1}\mathbb{P}[Y_0\,\vert\, H_0=C_i].
\end{align*}
$$

<!-- While the math seems almost too trivial, I had to write it down or else _I_ would have been confused. -->

## **Only** canonical representations
What if we don't store the full statistics for all card pairs, but only their canonical representations?

1. Sample the whole set of hole cards \\( \mathcal{H}=\\\{H_0,\dots, H_{n-1}\\\} \\)
2. Get all results from the game simulation \\( \mathbf{Y}=\\\{Y_0,\dots, Y_{n-1}\\\} \\)
3. Get the canonical representation of $H_0$ and its result, then update the statistics.

It made sense, right? This is (roughly) equivalent to just taking the average of the statistics corresponding to all hands of one canonical representation. But but, you may ask, (and _you_ here is me asking myself if it isn’t clear enough already that I may have schizophrenia),

> You can’t lump the counts of different hands altogether! The number of _seen_ `AsKh` is not equal to which of `AdKc`, so you have to keep the statistics separated, take their means, and _then_ average the means!

When the number of samples goes to infinity, the numbers of available samples for each individual hand will be more equal as they have identical priors. Also,

$$
\begin{align*}
\mathbb{P}[Y_0\,\vert\, H_0\in\mathcal{C}]
&=\mathbb{E}_{H\sim\mathcal{C}}\mathbb{P}[Y_0\,\vert\, H_0=H]\\
&=\mathbb{E}_{H\sim\mathcal{C}}\mathbb{E}_{\mathcal{H}_{\backslash 0}|H_0=H}\mathbb{P}[Y_0\,\vert\, H_0=H,{\mathcal{H}_{\backslash 0}}]
\end{align*}
$$

We can sample the outer expectation distribution by sampling $H_0$ from the global distribution, then filter only the samples that belong to $\mathcal{C}$ to compute statistics with, a textbook example of _rejection sampling_. Note that this estimator is _self-normalized_, which suffers from finite-sample bias, because the normalization term (number of globally-sampled $H_0$ that is in $\mathcal{C}$) itself is random. But the good news is, this bias term goes to zero as our sample size goes to infinity (i.e. a consistent estimator).

Effectively, our above algorithm is interleaving the sampling processes for different $\mathcal{C}$. While we still would get a correct estimate, these estimates' variances might not be identical, depending the size of $\mathcal{C}$. We can easily deal with that by actively upsampling less-seen canonical hands instead of just randomly generate a random pair, adding a little complexity in our code.

So all is still fine and dandy here, until I decided to try to take _another_ shortcut.
 <!-- and was confused why it didn’t give me the correct answer. -->

## Reusing results
Note that the position of the player in question does not need to be fixed:

$$
\mathbb{P}[Y_0\,\vert\, H_0\in\mathcal{C}] = \mathbb{P}[Y_i\,\vert\, H_i\in\mathcal{C}]\;\forall H_i\in\mathcal{H}.
$$

This means that we can pick any seat of the table to record these statistics; in fact, we can select any arbitrary (possibly non-random) seat every simulation and would still get the correct result! Just make sure that the seat selection is picked _before_ seeing the hands.

> But but, rejection sampling is done after seeing a hand!

Calm down, the process still select a hand index before the realization of the hand. The acceptance criteria $A$ just needs to be independent of the random variable being estimated, in this case, $A \, \mathrel{\perp\mspace{-10mu}\perp} \, Y_0 \mid H_0\in\mathcal{C}$. Note the conditional $H_0\in\mathcal{C}$: because we are estimating $Y_0$ only conditionally, the acceptance criteria can also rely on that condition without influencing the results of sampling $Y_0$.

So anyway, greed is good, I present to you my "optimized" implementation:

1. Sample the whole set of hole cards \\( \mathcal{H}=\\\{H_0,\dots, H_{n-1}\\\} \\)
2. Get all results from the game simulation \\( \mathbf{Y}=\\\{Y_0,\dots, Y_{n-1}\\\} \\)
3. For _each_ hand, get its canonical representation, then update the statistics.

Now if I accumulate statistics across all players of every simulation, eventually I’ll run into the case when more than one player in that game have the same canonical representation, e.g. both have `AKo`. In that case, the two accumulated "samples" are actually dependent of each other, and all intuitive senses go out the window. It's probably obvious that Monte Carlo estimator requires its samples to be independent to guarantee variance bounds, thanks to the Law of Large Numbers, but that's not _too_ important for correctness. The bigger question is, does this sample dependency affect the consistency of the estimator? Or more simply, is this estimator unbiased?
<!-- I guess it's obvious by now where my downfall starts. If I accumulate statistics across all players of every simulation, eventually I’ll run into the case when more than one player in that game have the same canonical representation, e.g. both have `AKo`. In that case, the two accumulated "samples" are actually dependent of each other, and all hell breaks loose. It's easy to see that Monte Carlo estimator requires its samples to be independent to guarantee variance bounds, thanks to the Law of Large Numbers, but that's not the dangerous part. The real deadly pitfall is, I now am not sampling from the right distribution anymore. This distribution will be biased towards canonical hands with larger size $|\mathcal{C}|$, because their hands appear _together_ more often; specifically, I'm double-counting those plays. And it gets worse as the number of players increases. -->

To answer this question, let us write out the complete math:

$$
\begin{align*}
p(\mathcal{C})
&= \frac{\mathbb{E}_\mathcal{H}[\sum_i\mathbb{1}[H_i\in\mathcal{C}]\mathbb{P}[Y_i|\mathcal{H}]]}{\mathbb{E}_\mathcal{H}[\sum_i\mathbb{1}[H_i\in\mathcal{C}]]} \\
&= \frac{\sum_i\mathbb{E}_\mathcal{H}[\mathbb{1}[H_i\in\mathcal{C}]\mathbb{P}[Y_i|\mathcal{H}]]}{\mathbb{E}_\mathcal{H}[|\mathcal{H}\cap\mathcal{C}|]} & \text{(Expectation is linear)} \\
&= \frac{\sum_i\mathbb{E}_{H_i}\mathbb{E}_{\mathcal{H}_{\backslash i}|H_i}[\mathbb{1}[H_i\in\mathcal{C}]\mathbb{P}[Y_i|H_i,\mathcal{H}_{\backslash i}]]}{\mathbb{E}_\mathcal{H}[|\mathcal{H}\cap\mathcal{C}|]} & \text{(sampling w/o replacement)} \\
&= \frac{\sum_i\mathbb{E}_{H_i}[\mathbb{1}[H_i\in\mathcal{C}]\mathbb{P}[Y_i|H_i]]}{\mathbb{E}_\mathcal{H}[|\mathcal{H}\cap\mathcal{C}|]} \\
&= \frac{n\times\mathbb{E}_{H_0}[\mathbb{1}[H_0\in\mathcal{C}]\mathbb{P}[Y_0|H_0]]}{\mathbb{E}_\mathcal{H}[|\mathcal{H}\cap\mathcal{C}|]} & \text{(positions are symmetric)} \\
&= \frac{n\times\mathbb{E}_{\mathcal{C}'\sim\mathfrak{C}}\mathbb{E}_{H_0\sim\mathcal{C}'}[\mathbb{1}[\mathcal{C}=\mathcal{C}']\mathbb{P}[Y_0|H_0\in\mathcal{C}']]}{\mathbb{E}_\mathcal{H}[|\mathcal{H}\cap\mathcal{C}|]} & \text{(sampling equivalence)} \\
&= \frac{n\times\mathbb{E}_{\mathcal{C}'\sim\mathfrak{C}}[\mathbb{1}[\mathcal{C}=\mathcal{C}']\mathbb{E}_{H_0\sim\mathcal{C}'}[\mathbb{P}[Y_0|H_0\in\mathcal{C}']]]}{\mathbb{E}_\mathcal{H}[|\mathcal{H}\cap\mathcal{C}|]} \\
&= \frac{n\times\mathbb{E}_{\mathcal{C}'\sim\mathfrak{C}}[\mathbb{1}[\mathcal{C}=\mathcal{C}']\mathbb{P}[Y_0|H_0\in\mathcal{C}']]}{\mathbb{E}_\mathcal{H}[|\mathcal{H}\cap\mathcal{C}|]} & \text{($C_i$ are equivalent)} \\
&= \frac{n\times\mathbb{P}[Y_0|H_0\in\mathcal{C}]\mathbb{P}[\mathcal{C}]}{\mathbb{E}_\mathcal{H}[|\mathcal{H}\cap\mathcal{C}|]} & \text{($\mathcal{C}\in\mathfrak{C}$ are disjoint)} \\
&= \frac{n\mathbb{P}[\mathcal{C}]}{\mathbb{E}_\mathcal{H}[|\mathcal{H}\cap\mathcal{C}|]}\mathbb{P}[Y_0|H_0\in\mathcal{C}] \\
&= \frac{n\times\mathbb{E}_H[\mathbb{1}[H\in\mathcal{C}]]}{\mathbb{E}_\mathcal{H}[\sum_i\mathbb{1}[H_i\in\mathcal{C}]]}\mathbb{P}[Y_0|H_0\in\mathcal{C}] \\
&= \frac{n\times\mathbb{E}_H[\mathbb{1}[H\in\mathcal{C}]]}{\sum_i\mathbb{E}_\mathcal{H}[\mathbb{1}[H_i\in\mathcal{C}]]}\mathbb{P}[Y_0|H_0\in\mathcal{C}] \\
&= \frac{n\times\mathbb{E}_H[\mathbb{1}[H\in\mathcal{C}]]}{n\times\mathbb{E}_\mathcal{H}[\mathbb{1}[H\in\mathcal{C}]]}\mathbb{P}[Y_0|H_0\in\mathcal{C}] & \text{(positions are symmetric)} \\
&= \mathbb{P}[Y_0|H_0\in\mathcal{C}].
\end{align*}
$$

Apparently it is.

## Variance reduction

Alright, so now I have a consistent estimator that still feels weird in my head, but the math guarantees that it really is unbiased, and the only issue left is with regards to variance. How do we fit this last piece of the puzzle?

Well, fun fact: the above algorithm is asymptotically equivalent to if we take the mean of the results of all the pairs that belong to the same canonical hand before aggregation.
<!-- As usual, the proof is left as an exercise to the reader, but -->
The main idea still revolves around the self-normalizing property of these estimators, and for the sake of rigor:

$$
\begin{align*}
\tilde{p}(\mathcal{C})
&= \mathbb{E}_\mathcal{H}\left[\frac{\sum_i\mathbb{1}[H_i\in\mathcal{C}]\mathbb{P}[Y_i|\mathcal{H}]}{\sum_i\mathbb{1}[H_i\in\mathcal{C}]}\right] \\
\end{align*}
$$

- Why does variance go down?
- Can we upsample/stratified?

## The final algorithm

TBD

## Appendix: How to calculate Equity
To my understanding, Equity is almost like EV, where it measures the portion of the pot that belongs to you given your hand, depending on how the game turn out (i.e. what cards are drawn). This is obviously proportional to the pot, so taking it outside as a common factor we have:

$$
\text{Equity} = \text{pot} \times \left(\mathbb{P}[\text{win}] + \sum_{k=2}^\text{player count}\mathbb{P}[k\text{-way tie}]/k\right).
$$

The summation may look scary, but in reality, there's only one probably term in that big parentheses that is nonzero for each simulation &mdash; you can't both win and be in a $k$-way tie in one game. The proper way to do this is to create a counter for each $k$-way tie count, then the linearity of expectation trivially guarantees the correctness of the algorithm. We can take it one step further and combine the total tie equity across all values of $k$ into one total tracker by weighting the binary outcome of each tie with $1/k$ before adding it to the sum, and linearity would still apply; but honestly just keep the individual counts for flexibility during post-simulation statistics aggregation.

Now let the pot value be 1 pot :), we can thus ignore it and just take the value within those big parentheses as the output, effectively expressing Equity as a fraction of the pot. The difference between Equity and EV is that EV measures in an actual monetary quantity, e.g. dollars or big blinds; and Equity assumes nothing else changes after you see your hand (e.g. no post-flop actions).

The one thing I did not know before turned out to be the simplest, and the things I thought I knew threw me in for a giant loop. Amazing.

_P.S.: I also handwaved a lot by not specifically require you to sample the community cards to get one realization of a game given a full set of hole cards. This in turn would create a third nested expectation that I'm not willing to handle. If the math is wrong somewhere because of this, please let me know &mdash; while I suffered immensely writing this post, I think it's a good mental exercise._