---
layout: post
title: "[WIP] How [not] to run a Monte Carlo simulation."
description: "Wow I suck at probabilities."
thumb_image: 
tags: [math]
---

As usual, I was procrastinating my real research work by doing something irrelevant; this time it's equity calculator for a poker hand, assuming no betting actions are taken at any points after that. The plan has always been simple: sample hands, sample community cards, evaluate win/loss, and take the average. The execution though...

## The problem

Just to simplify, we are only concerning with the win/loss/tie probability. So, in a \\(n\\)-player game, given that you have seen your hole card and nothing else, what is the probability that you will win when the dust settles?

From this point on, assume that we have a function that evaluates the win/loss/tie probability given a set of hands, that bruteforce through all possible combinations of community cards. It takes 30 seconds for one such run on my dated MacBook Air with single core and no PyPy, so it's fast enough™️ in tandem with parallelization. Now, the Monte Carlo estimation is being averaged across sampled possible starting hands.

## The surefire way
Intuitively, just sample all other hands, run the simulation, and average outcomes. Simple, right?

Well, yeah. To put it rigorously, let \\( \mathcal{H}=\\\{H_0,\dots, H_{n-1}\\\} \\) be the starting hands, and \\(H_i\\) are the hole cards of the \\(i\\)-th player. Let \\(H\\) (with no suffix) be any particular hand we're trying to evaluate, and \\(Y_0\\) is a Boolean variable showing whether the first player won — excuse my 0-indexing. Then we have:

$$
\begin{align*}
\mathbb{P}[Y_0\,\vert\, H_0=H]
&= \sum_\mathcal{H}\mathbb{P}[Y_0,\mathcal{H}\,\vert\, H_0=H] \\
&= \sum_\mathcal{H}\mathbb{P}[Y_0\,\vert\,\mathcal{H},H_0=H]\times\mathbb{P}[\mathcal{H}\,\vert\,H_0=H] \\
&= \mathbb{E}_{\mathcal{H}\,\vert\,H_0=H}\mathbb{P}[Y_0\,\vert\,\mathcal{H},H_0=H].
\end{align*}
$$

This is just mathspeak for "sample hole cards such that the first hand is what we want, then count many times the first hand won." Also functionally equivalent to "only sample the other players' hands," i.e. just \\(\\\{H_1,\dots, H_{n-1}\\\}\\).

## The shortcut
With that approach, we have to do the whole song and dance above for each hand independently. Imagine we can accumulate counts for other hands in the above simulation! So here is what I did in pseudocode:

1. Sample the whole set of hole cards \\( \mathcal{H}=\\\{H_0,\dots, H_{n-1}\\\} \\)
2. Get all results from the game simulation \\( \mathbf{Y}=\\\{Y_0,\dots, Y_{n-1}\\\} \\)
3. For each hand, update the win/loss/tie statistics.

This would still work nicely, because everything is unique. Any hand in \\(\mathcal{H}\\) can be treated as index 0, and the sampling process for \\(\mathcal{H}\\) can be viewed as a 2-step process where one samples \\(H_0\\) first before the rest. If we condition the second step on the first, that is equivalent to doing one simulation for a (randomly) instantiated \\(H_0\\). As a result, this new approach is simply equivalent to running simulations for all hands, just in an intertwined order. I handwavied a little glossing over how the priors for all hands are identical, and how the hand positions are permutation-invariant, but those minor details don't affect the analysis — as per mathematician's tradition, the rest is left as an exercise to the reader.
<!-- Note that while \\(H_0\\) is randomly determined, the variable is already instantiated when we condition . -->

## The canonical representation
But when things stop being unique is when all the troubles started. See, what we want is not the actual hand — that is just for fancy analyses and real-time TV annotations. What we really want is *canonical forms*, that is, instead of `AdKd` (Ace-King of Diamond), we work with `AKs` (Ace-King suited), which actually is a *set* of possible hands (`AcKc`, `AdKd`, `AhKh`, `AsKs`). This canonical form lets us work with only 169 possibilities, instead of the \\(52C2=1326\\) that only someone like Sheldon Cooper can remember.

Now, what I should have done is that I should have gone for the approach above storing all full hands, then aggregate them later for the canonical version. This is perfectly fine because, let \\( \mathcal{C}=\\\{C_0,\dots, C_{m-1}\\\} \\) be a canonical hand and \\( C_i \\) be the actual hands that satisfy that canonical representation, we have:

$$
\begin{align*}
\mathbb{P}[Y_0\,\vert\, H_0\in\mathcal{C}]
&= \mathbb{P}[H_0\in\mathcal{C}\,\vert\, Y_0]\times \mathbb{P}[Y_0] / \mathbb{P}[H_0\in\mathcal{C}] \\
&= \sum_{i=0}^{m-1}\mathbb{P}[H_0=C_i\,\vert\, Y_0]\times \mathbb{P}[Y_0] \;{\Large /}\; \sum_{i=0}^{m-1}\mathbb{P}[H_0=C_i] & \text{(}C_i\text{ are uniques }\textit{as a pair}\text{)} \\
&= \sum_{i=0}^{m-1}\mathbb{P}[H_0=C_i\,\vert\, Y_0] \;{\Large /}\; \sum_{i=0}^{m-1}\mathbb{P}[H_0=C_i] \times \mathbb{P}[Y_0]\\
&= \frac1m\sum_{i=0}^{m-1}\frac{\mathbb{P}[H_0=C_i\,\vert\, Y_0]}{\mathbb{P}[H_0=C_i]} \times \mathbb{P}[Y_0] & \text{(}C_i\text{ are equally probable)}\\
&=\frac1m\sum_{i=0}^{m-1}\mathbb{P}[Y_0\,\vert\, H_0=C_i].
\end{align*}
$$

While the math seems so trivial, I had to write it down or else _I_ would have been confused. And also because I tried to take _another_ shortcut and was confused why it didn’t give me a good estimate.

## **Only** canonical representations
This is actually my first implementation, and the reason why I started writing this article:

1. Sample the whole set of hole cards \\( \mathcal{H}=\\\{H_0,\dots, H_{n-1}\\\} \\)
2. Get all results from the game simulation \\( \mathbf{Y}=\\\{Y_0,\dots, Y_{n-1}\\\} \\)
3. For each hand, get its canonical representation, then update the statistics.

It made sense, right? This is (roughly) equivalent to just taking the average of the statistics corresponding to all hands of one canonical representation. But but, you may ask, (and _you_ here is me asking myself if it isn’t clear enough already that I may have schizophrenia),

> You can’t lump the counts of different hands altogether! The number of `AsKh` is not equal to which of `AdKc`, so you have to keep the statistics separated, take their means, and _then_ average the means!

When the number of samples goes to infinity, the numbers of available samples for each individual hand will be more equal as they have identical priors. Also,

$$
\begin{align*}
\mathbb{P}[Y_0\,\vert\, H_0\in\mathcal{C}]
&=\mathbb{E}_{H\sim\mathcal{C}}\mathbb{P}[Y_0\,\vert\, H_0=H]\\
&=\mathbb{E}_{H\sim\mathcal{C}}\mathbb{E}_{\mathcal{H}\,\vert\,H_0=H}\mathbb{P}[Y_0\,\vert\,\mathcal{H},H_0=H]\\
&=\mathbb{E}_{H\sim\mathcal{C}}\mathbb{E}_{\bar{\mathcal{H}}}\mathbb{P}[Y_0\,\vert\,H,\bar{\mathcal{H}}]\\
&=\mathbb{E}_{H\sim\mathcal{C},\bar{\mathcal{H}}}\mathbb{P}[Y_0\,\vert\,H,\bar{\mathcal{H}}],
\end{align*}
$$

where \\(\bar{\mathcal{H}} = \mathcal{H}\setminus\\\{H_0\\\}\\). The double expectation here means that we can sample the first canonical hand, then sample the rest of the table; and we then can lump the double sampling together during aggregated averaging at the end.

> But the player has to always be at seat 0 across all simulations!

Nope.

$$
\begin{align*}
\mathbb{P}[Y_0\,\vert\, H_0\in\mathcal{C}]
&=\mathbb{E}_{H\sim\mathcal{C}}\mathbb{P}[Y_0\,\vert\, H_0=H]\\
&=\mathbb{E}_{H_i\sim\mathcal{H}}\mathbb{E}_{H\sim\mathcal{C}}\mathbb{P}[Y_0\,\vert\, H_i=H].
\end{align*}
$$

However, there’s something tricky here. This formulation is only correct if you randomly sample the player seat every time. If I accumulate statistics across all players of every simulation, eventually I’ll run into the case when more than one in that game have the same hole cards, e.g. both have `AKo`. In that case, the independence condition isn’t upheld anymore, and all hell breaks loose.

Let’s first start writing the math for the proposed algorithm above:

$$
\begin{align*}
\frac{\mathbb{E}_{\mathcal{H}}\mathbb{E}_{H_i}\mathbb{P}[Y_i|\mathcal{H},H_i\in\mathcal{C}]}
{\mathbb{E}_{\mathcal{H}}\mathbb{E}_{H_i}\mathbb{P}[H_i\in\mathcal{C}|\mathcal{H}]}
\end{align*}.
$$

Note that the expectation can be equivalently exchanged with summation, or averaging — the normalizing constants are all identical between the numerator and denominator. This differs from our original formulation only at the expectation over players, whereas the original fixed the player at index 0:

$$
\begin{align*}
\frac{\mathbb{E}_{\mathcal{H}}\mathbb{P}[Y_0|\mathcal{H},H_0\in\mathcal{C}]}
{\mathbb{E}_{\mathcal{H}}\mathbb{P}[H_0\in\mathcal{C}|\mathcal{H}]}
&=\frac{\mathbb{E}_{\mathcal{H}}\mathbb{P}[Y_0,\mathcal{H}|H_0\in\mathcal{C}]/\mathbb{P}[\mathcal{H}|H_0\in\mathcal{C}]}
{\mathbb{E}_{\mathcal{H}}\mathbb{P}[\mathcal{H}|H_0\in\mathcal{C}]\times\mathbb{P}[\mathcal{H}]/\mathbb{P}[H_0\in\mathcal{C}]}
\end{align*}.
$$

<!-- $$
\begin{align*}
\sum_{\mathcal{H}}\frac{\sum_{H_i}\mathbb{P}[Y_i|\mathcal{H}]\mathbb{P}[H_i\in\mathcal{C}|\mathcal{H}]}{\sum_{H_i}\mathbb{P}[H_i\in\mathcal{C}|\mathcal{H}]}
&=\sum_{\mathcal{H}}\frac{\sum_{H_i}\mathbb{P}[Y_i|\mathcal{H}]\mathbb{P}[\mathcal{H}|H_i\in\mathcal{C}]}{\sum_{H_i}\mathbb{P}[\mathcal{H}|H_i\in\mathcal{C}]} & (1) \\
&=\sum_{\mathcal{H}}\frac{\sum_{H_i}\mathbb{P}[Y_i|\mathcal{H}]\mathbb{P}[\mathcal{H}|H_i\in\mathcal{C}]}{|\mathcal{H}|\times\mathbb{P}[\mathcal{H}|H_i\in\mathcal{C}]} & (H_i\text{ are equivalent})
\end{align*}$$ -->

<!-- where \\((1)\\) happens when we multiply both the numerator and denominator with \\(\mathbb{P}[\mathcal{H}]/\mathbb{P}[H_i\in\mathcal{C}]\\), and \\(\mathbb{P}[H_i\in\mathcal{C}]\\) is identical for any hand \\(H_i\\). -->

## The result

TBD

## Appendix: How to calculate Equity
To my understanding, Equity is almost like EV, where it measures the portion of the pot that belongs to you given your hand, depending on how the game turn out (i.e. what cards are drawn). This is obviously proportional to the pot, so taking it outside as a common factor we have:

$$
\text{Equity} = \text{pot} \times \left(\mathbb{P}[\text{win}] + \sum_{k=2}^\text{player count}\mathbb{P}[k\text{-way tie}]/k\right).
$$

We can now just take the value within those big parentheses as the output. The difference between Equity and EV is that the former assumes nothing else changes, and the latter measures in an actual monetary quantity, e.g. dollars or big blinds.

The one thing I did not know before turned out to be the simplest, and the things I thought I knew threw me in for a giant loop. Amazing.
