---
layout: post
title: "Control Variates and the Satisfying Telescoping Sum"
description: "A lot of sidetracking for such a simple algorithm."
thumb_image: 
tags: [math]
---

_This is part 2 of the pokeqt series, where I do poker-related things because I liked a girl who do._
{: style="margin-bottom: 0.5em !important; font-size: .9em" }
1. [How [not] to run a Monte Carlo simulation]({{ page.previous.url }})
2. _Control Variates and the Satisfying Telescoping Sum_
3. [Stacking chips and CUDA cores]({{ page.next.url }})
{: style="font-size: .95em; opacity: 0.9" }

---
{: style="margin-bottom: 2em !important" }

Control variate is a technique to reduce variance in Monte Carlo simulations, by exploiting information on the error of a different known estimate to effectively offset which of the unknown one.

But I'll spare you the boring text, we both had enough of that last time. The gist of this post is that Control Variate is actually _sooooo_ good for poker probability/equity estimation. So here comes the glaze fest in 4 parts.

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

We can trivially show that this new estimator is unbiased:

$$
\mathbb{E}[\bar{x}] = \mathbb{E}[x + c(\mathbb{E}[x']-x')] = \mathbb{E}[x] + c(\mathbb{E}[x'] - \mathbb{E}[x']) = \mathbb{E}[x].
$$

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

## The first great thing about CV for this task is...
...that there is a very obvious choice for $q(x)$, that is, the probabilities for a 2-player (heads-up) game! The logic should be simple enough that you can write a symbolic solver for this problem without the need for MC sampling. And, even if you only get an estimate for the mean of the 2-player distribution, from here on noted as $p_2(x)$, CV still would work well enough, just less effective and without the unbiased guarantee. For the second requirement, it's pretty obvious that the same (canonical) hand in different $n$-player games would have very high correlation. I am not rigorously proving that though, you'd just have to take my word for it.

Then, let's say we're estimating $p_3(x)$. We know that $p_2(x)$ and $p_3(x)$ are highly correlated, so the estimator variance can go down to near zero! That's how magical it is.

> But, that is only for the optimal $c$!

Yes, but we can get a good estimate just by taking the statistics of the current samples. Or, [just set $c=1$ if $p$ and $q$ are close enough](https://www.value-at-risk.net/variance-reduction-with-control-variates-monte-carlo-simulation), which they are!

So with $c=1$, the current algorithm would go something like this:

1. Sample $x\sim p_3(x)$
2. Sample $x'\sim p_2(x)$
3. Augment the original sample by $\bar{x}=x + (\mathbb{E}[x']-x')$.
4. Use $\tilde{x}$ as if it's $x$.

More concretely, (1) we sample a game for the 3-player setting and check to see which hand won, then (2) sample a game for the 2-player setting and also evaluate to see who won, and then finally apply the correction to (1) with (2) and the known win-probability of that hand from the analytic solution of the 2-player setting.

By doing this, we are sampling twice per game, which effectively double the amount of computation needed. That's why CV is usually used when the sampling overhead of $q$ is greatly less than which of $p$, which isn't in our case. But there's another saving grace:

## The second great thing about CV for this task is...
... that we can completely reuse the sampled hands!

But a slight tangent on result reuse first. Let's say we are simulating the 3-player version: we simply sample the board and rank the hands, with two important properties:
- The hole cards for each player are i.i.d. (permutation-invariant), and independent.
- The relative rankings of hands do not change if a player is removed from play.

Which means, after sampling a 3-player game and rank the players' hands, we can trivially drop one player and get a perfectly random 2-player game state and result with zero additional computation! As a result, you can reuse one sampled game for all table sizes if you're running multiple $n$-player simulations concurrently; and coupling with aggregating results from all hands in one game, the savings are massive.

Now for the good part: since we can also use the zero-cost sampled game result from the $n-1$-player game as the CV for the $n$-player simulation, the overhead is basically zero. And the best part is, if we use the same hands and board, the correlation shoot through the roof! It's obvious that if you win a 3-player heads-up game, you're gonna win the same game if one of your opponent folds first. The other way around is slightly tricker, but we'll handwavy a bit as the intuition remains. Consequently, the new optimal CV-corrected variance goes to 0, and we have a near perfect simulation.

We will now simplify notation by dropping the 0 subscript, and focus on only one player at a time; and replace it with the player count of the variable: e.g. $Y_3$ represents whether the 0-index player wins in a 3-player game. The algorithm is then very simple:
1. Sample all hands $\\{H_0, H_1, H_2\\}$ and community (shared) cards $\mathcal{S}$.
2. Rank the hands, and evaluate $Y_{3}$ whether $H_0$ wins the 3-player game, and $Y_{2}$ whether $H_0$ wins against $H_1$ alone (e.g. if $H_2$ folds without looking at anything).
3. Augment the sampled evaluation by $\bar{Y}_3= Y_3 + (\mathbb{E}[Y_2]- Y_2)$.
4. Update the tracker of the corresponding canonical class of $H_0$ with $\bar{Y}_{3}$.

Another way to look at this construction is that we start the estimate of $Y_{3}$ by $\mathbb{E}[Y_{2}]$, then we estimate the expectation of $\Delta_{3} \overset{\Delta}{=} Y_{3} - Y_{2}$ instead:

$$
\mathbb{E}[{Y}_{3}]
= \mathbb{E}[Y_{3} - Y_{2}] + \mathbb{E}[Y_{2}]
= \mathbb{E}[\Delta_{3}] + \mathbb{E}[Y_{2}].
$$

The variance of $\Delta_{3}$ will be much lower, as $Y_{3}$ and $Y_{2}$ tend to agree: if you win a 3-player game, you're guaranteed to win a 2-player game; if you lose a 2-player game, you're guaranteed to lose the 3-player game, and the other direction is likely to be the same as well. To put it in another way, we are tracking only the changes induced by a player folding pre-flop; and, because there is *no* change most of the times, this new random variable desirably has very little noise. This new underlying variable is the reason why $\mathbb{E}[\bar{Y}_{(\cdot)}]$ and $\mathbb{E}[{Y}\_{(\cdot)}]$ both estimate the same entity, but the former gets to a have much smaller variance.

## The third great thing about CV for this task is...
... that you can _cleanly_ bootstrap it across increasing player-count settings to create what's called a multi-level MC.

Perhaps the most natural question to ask is: what happens if I apply CV recursively?  Recall that we have a CV-corrected 3-player estimate of $\bar{Y}_3= Y_3 + (\mathbb{E}[Y_2]- Y_2)$. If we do the same thing for the 4-player setting:

$$
\begin{align*}
\bar{Y}_{4}
&= Y_{4} + (\mathbb{E}[\bar{Y}_{3}]- \bar{Y}_{3}) \\
&= Y_{4} + \mathbb{E}[{Y}_{3}] - (Y_{3} + [\mathbb{E}[Y_{2}]- Y_{2})]
& \text{(}\bar{Y}_{3}\text{ unbiased)} \\
&= Y_{4} + \mathbb{E}[{Y}_{3} - Y_{2}] - (Y_{3} - Y_{2}) \\
&= Y_{4} + \mathbb{E}[{\Delta}_{3}] - \Delta_{3}.
\end{align*}
$$

If we condition $\bar{Y}_4$ on the CV-corrected sample $\bar{Y}_3$, we are actually effectively using $\Delta_3 = Y_3 - Y_2$ as the control variate. By rewriting this CV construction to only contain original (pre-controlled) variables, we can easily point out a few problems:
- We do not have the exact $\mathbb{E}[{Y}_3]$ (and as a result, $\mathbb{E}[\Delta_3]$), so bias is introduced.
- It's not guaranteed that $Y_4$ and $\Delta_3$ are correlated.
- Variance of $\Delta_3$ is low, so coupled with the unknown correlation, the optimal value for $c^*\ne 1$ becomes much less obvious.

To illustrate the point on correlation, $\Delta_{3}$ can only take on the values of 0 or $-1$, where $-1$ implies that the added 3rd player has a hand that beat your currently winning one, and 0 if nothing changes &mdash; you cannot turn a loss into a win with the addition of the 3rd hand. We enumerate all these possibilities:

- If $Y_3-Y_2 = -1$, this means that $Y_3=0$ and $Y_2 = 1$. In this case, you *cannot* win the 4-player game because you already lost the 3-player game, therefore $Y_4=0$.
- If $Y_3-Y_2 = 0$, this doesn't let you know much: you could be already having trash hands and losing regardless (i.e. $Y_2=Y_3=Y_4=0$), or you could be having nuts that are basically unbeatable (i.e. $Y_2=Y_3=Y_4=1$), or just complete ambiguity whether the next player will have something that cracks you (i.e. $Y_2=Y_3=1,Y_4\overset{?}{=}0$).

The last ambiguity, coupled with the fact that $Y_4=0$ more often than $\Delta_3=-1$, is the reason why I only said correlation is not *guaranteed*, even though I imagine it would work somewhat decently. But the bigger issue is with bias; and this same bias issue also affects the possibility of just applying CV on the pre-controlled estimate where $\bar{Y}_4= Y_4 + (\mathbb{E}[Y_3]- Y_3)$. So what do we do instead?

Well, we could cancel out these biases by combining 2 CVs! Let the first CV be $-\Delta_{3}$ (note the minus) and the second CV be our plain old $Y_{3}$. Then taking the average of the two controlled random variables, we have:

$$
\begin{align*}
\bar{Y}_{4}
&= \frac{1}{2}\left[Y_{4} + \mathbb{E}[{Y}_{2} - Y_{3}] - (Y_{2} - Y_{3})\right]
+ \frac{1}{2}\left[Y_{4} + (\mathbb{E}[Y_{3}]- Y_{3})\right] \\
&= Y_{4} + \frac12\mathbb{E}[{Y}_{2}] - \frac12 Y_{2}
\end{align*}
$$

Two things popped up immediately:
1. We now no longer require (a biased estimate of) $\mathbb{E}[Y_3]$; and this new estimator is consistent, i.e. $\mathbb{E}[\bar{Y}_4]=\mathbb{E}[Y_4]$. 
2. This is equivalent to just biting the bullet and just use $Y_2$ as the CV, but with coefficient $c=1/2$.

While $Y_4$ would be slightly less correlated to $Y_2$ than $Y_3$, it should still do the job. Additionally, this new coefficient would actually work better than the default $c=1$! Note that $Y_4$ (and $Y_n$ in general) would have a lot more zeros than ones comparing to $Y_2$, and this trend gets worse as $n$ increases. This implies that we are controlling a low-variance variable with a higher-variance variable; thus using $Y_2$ as-is as the CV may even *increase* the estimator's variance. However, we can offset this by lowering the CV coefficient; and in this case, we reduce the CV's variance by a factor of 4.
<!-- &mdash; which coincides with the approximately-$1/n$ decay of equity. -->

Let's scale up to any $n$:

$$
\begin{align*}
\bar{Y}_{n}
&=\frac1{n-2}\left[Y_n + \mathbb{E}[Y_{n-1}]- {Y}_{n-1}
+\sum_{i=3}^{n-1}\left[Y_n - \mathbb{E}[\Delta_{i}] + \Delta_{i}\right]
\right] \\
&=Y_n
+ \frac1{n-2}\mathbb{E}\left[Y_{n-1}-\sum_{i=3}^{n-1}\Delta_{i}\right]
- \frac1{n-2}\left[Y_{n-1}-\sum_{i=3}^{n-1}\Delta_{i}\right] \\
&=Y_n
+ \frac1{n-2}\mathbb{E}\left[Y_{2}\right]
- \frac1{n-2}Y_2.
\end{align*}
$$

The simplification for the last derivation comes from our promised telescoping sum:

$$
Y_{n-1}-\sum_{i=3}^{n-1}\Delta_{i} = Y_{n-1} - (Y_{n-1} - Y_{n-2}) - \dots - (Y_3 - Y_2) = Y_2.
$$

<sub>Okay, maybe it isn't as exciting as I found it when I first found it.</sub>

There are many ways to combine these CV constructions so that every terms except for $Y_2$ cancels out (somehow this reminds me of conditional convergence...), but there are a couple nice properties with this version:
- The telescoping sum is clean &mdash; you gotta admit it's pretty satisfying...
- Breaking down the gap between $Y_n - Y_2$ as a sum of $\Delta_i$ spreads out the *mass* of the total difference, which significantly reduces the variances of the CVs.
- The "starting point" for the estimator $\mathbb{E}[Y_n]$ is $\frac{1}{n-2}\mathbb{E}[Y_2]$, which is more in line with the $1/n$ fair-share baseline for equity.

With that being said, that selection of $c$ is still probably not the optimal choice; and we used $-\Delta_{(\cdot)}$ mostly to cancel out the bias. But there is a better way...

## The fourth great, and probably best, thing about CV for this task is...
... that you can just use this as a constant-time post-processing variance correction.

Since the formula for control variate is linear, we can actually just apply the correction at the very end on the final estimate. The benefit actually goes beyond the near-zero computation: we can actually estimate the optimal $c^*$! Yes, we have to keep track of the covariances of each variable and its corresponding control variate, but that's a very small price to pay for salvation. In fact, we only have to additionally keep track of only the total of the product of the two variables:

$$
\text{Cov}(X, Y) = \mathbb{E}[(X-\mathbb{E}X)(Y-\mathbb{E}Y)] = \mathbb{E}[XY] - \mathbb{E}X\mathbb{E}Y
$$

<!-- We still run into the issue of rounding error if either of these variables is a `double`, but this error goes away quickly when the sample size gets big enough. -->

Another reason why I think this is the best thing about CV for this task is that it solves a problem that I've been hiding from you this whole time, that is, what do we use as the CV for $k$-way tie? Those statistics are not very correlated to the 2-way tie if $k> >2$, and even less to the (heads-up) winning/losing probability. But, they are correlated to the _equity_; in fact, equity is linear with respect to the $k$-way tie statistics with coefficient $1/k$! We can further assume that the $n$-player equity is correlated to the 2-player counterpart, and use the heads-up equity as the CV for any $k$-way tie. But then, two new issues arise with this construction:
- $k$-way tie probabilities decay rapidly as $k$ increases, so using heads-up equity as the CV for all of them is very suboptimal.
- Equity is floating-point, which means that rounding error will be catastrophic.

There's a magic bullet that fixes all of our problems, that is, we can determine the optimal *mixing* of CVs post-simulation! If we can use the heads-up equity, which is $\mathbb{P}_\mathrm{win} + 1/2\mathbb{P}\_\mathrm{tie}$, as the CV, we can dynamically replace those mixing coefficients so that the final CV is best correlated to the estimating variable at hand (note that the aforementioned CV coefficient $c$ can be lumped into these mixing coefficients). Then, each $k$-way tie in a $n$-player game gets a tailored CV, we only to keep track of the covariances of binary (not floating-point) variables, and no more arbitrary $\Delta_i$ construction just to cancel out the bias! How amazing is that?

And the implementation is dead simple: adapting the optimal coefficient calculation to the multivariate setting, we have:

$$
\mathbf{c}^*=\mathbf{\Sigma}_C^{-1}\Sigma_{CY},
$$

where $\mathbf{\Sigma}\_C$ is the covariance matrix of the CVs, which in this case, the heads-up win and tie probabilities (we drop the loss probability because it is linearly dependent to the other two); and $\Sigma_{CY}$ is the covariance between the CVs and *all* other estimating variables. Yes, you heard me right, one $2\times 2$ matrix inversion which has a closed-form formula, and one matrix multiplication. It's really that simple.

One last quick small disclaimer before we move on: when we use our sampled statistics to compute the optimal CV coefficients, we are introducing a small bias. In order for the CV method to be unbiased, $c$ (usually a constant) and the CV must be independent, but they aren't in this case. There are ways to deal with this issue, but perhaps the most notable one is *splitting*. That is, we split our simulation statistics into two halves $A$ and $B$, then we use $A$ to find the optimal coefficients for $B$, and vice versa; and at the end aggregate back the two halves &mdash; that way, we can use all of our simulation results for the final estimate without having to set out some samples for covariance estimation.
However, I think that this approach might gives us a worse estimate, as our main problem is variance rather than bias: halving the dataset worsens the covariance estimates of both the CVs and the target variables for each half, and aggregating them back later on doesn't help enough to justify both the minimal induced bias and the implementation overhead.
Regardless, if you can't sleep well at night thinking about this imperfection, just do the split; and if I'm wrong about this, please correct me in the chat.

## The final algorithm
Putting it all together, we have our final algorithm: for a $n\ge 2$-player game:

1. Sample the whole game, including hole cards and community cards.
2. Get all players' results from the game simulation.
3. Use said ranking to evaluate the result for all $n$-player games. Also record covariances between the variables and their corresponding control variates.
4. Accumulate across all hands.

Then at the end (or any checkpoint), compute the statistics, and finally apply control variate using the analytical 2-player result &mdash; simple as that. While we lose out of the ability to stratify/upsample per canonical hand, the math and implementation is so satisfying that I'm willing to turn a blind eye. Now, onwards to the code!

## _Bonus:_ Bait-and-switch
A keen reader would probably have realized that the current version of the algorithm actually changed, and the previous proof would not hold anymore. Specifically, for a $n<9$-player game, the $n$ players are not playing against each other anymore; instead, they are playing with the _next_ 7 players to their right!

Actually, that doesn't change anything: the hands are still dealt fairly, and the proof did not require any correlation (or lack thereof) between different players' outcomes within game(s). But this time, now that each player in the table is playing a (partially) different game, variance analysis would actually gets complicated. But that's the tradeoff we pay for both runtime (9 samples per simulation even for 3-player game), and we still have control variates. Analysis on the variance guarantee of this new algorithm is once again left as an exercise to the readers, but this time the it's because the proof seems complicated and I have sunk way too much time and mental wellness on this project; so if it's possible, please do it and drop a comment below &mdash; thank you!

Pushing it to the limit, technically it is possible to be even greedier, by accumulating _all_ results in _each_ subgame. Even greedier, you can evaluate all possible subgames, by running through all possible combination of the players. However, my impression is that the amount of information gained is little comparing to the increased variance induced by reusing old samples; and it becomes a lot more difficult to do CV both design-wise and implementation. As such, I think here is a pretty good stopping point.