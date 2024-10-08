---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.16.2
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
numbering:
  enumerator: 6.%s
---

# 6  Policy Optimization

## Introduction

The core task of RL is finding the **optimal policy** in a given environment.
This is essentially an _optimization problem:_
out of some space of policies,
we want to find the one that achieves the maximum total reward (in expectation).

It's typically intractable to compute the optimal policy exactly.
Instead, **policy optimization algorithms** start from some randomly initialized policy,
and then _improve_ it step by step.
We've already seen some examples of these,
namely [](#policy_iteration) for finite MDPs and [](#iterative_lqr) in continuous control.
In particular, we often use policies that can be described by some finite set of _parameters._
For such parameterized policies,
we can approximate the **policy gradient:**
the gradient of the expected total reward with respect to the parameters.
This tells us the direction the parameters should be updated to achieve a higher total reward (in expectation).
Policy gradient methods are responsible for groundbreaking applications including AlphaGo, OpenAI Five, and large language models,
many of which use policies parameterized as deep neural networks.

1. We begin the chapter with a short review of gradient ascent,
a general **optimization method.**
2. We'll then see how to estimate the **policy gradient,**
   enabling us to apply (stochastic) gradient ascent in the RL setting.
3. Then we'll explore some _proximal optimization_ techniques that ensure the steps taken are "not too large".
   This is helpful to stabilize training and widely used in practice.

```{code-cell} ipython3
from utils import plt, Array, Callable, jax, jnp
```

## Gradient Ascent

**Gradient ascent** is a general optimization algorithm for any differentiable function.
A suitable analogy for this algorithm is hiking up a mountain,
where you keep taking steps in the steepest direction upwards.
Here, your vertical position $y$ is the function being optimized,
and your horizontal position $(x, z)$ is the input to the function.
The _slope_ of the mountain at your current position is given by the _gradient_,
written $\nabla y(x, z) \in \mathbb{R}^2$.

```{code-cell} ipython3
def f(x, y):
    """Himmelblau's function"""
    return (x**2 + y - 11)**2 + (x + y**2 - 7)**2

# Create a grid of points
x = jnp.linspace(-5, 5, 400)
y = jnp.linspace(-5, 5, 400)
X, Y = jnp.meshgrid(x, y)
Z = f(X, Y)

# Create the plot
fig, ax = plt.subplots(figsize=(6, 6))

# Plot the function using imshow
img = ax.imshow(Z, extent=[-5, 5, -5, 5], origin='lower')

# Add color bar
fig.colorbar(img, ax=ax)

# Gradient computation using JAX
tx, ty = 1.0, 1.0
gx, gy = jax.grad(f, argnums=(0, 1))(tx, ty)

# Scatter point
ax.scatter(tx, ty, color='red', s=100)

# Add arrow representing the gradient
ax.arrow(tx, ty, gx * 0.01, gy * 0.01, head_width=0.3, head_length=0.3, fc='blue', ec='blue')

# Add plot title
ax.set_title("Himmelblau's Function")

plt.show()
```

For differentiable functions, this can be thought of as the vector of partial derivatives,

$$
\nabla y(x, z) = \begin{pmatrix}
\frac{\partial y}{\partial x} \\
\frac{\partial y}{\partial z}
\end{pmatrix}.
$$

To calculate the _slope_ (aka "directional derivative") of the mountain in a given direction $(\Delta x, \Delta z)$,
you take the dot product of the difference vector with the gradient.
This means that the direction with the highest slope is exactly the gradient itself,
so we can describe the gradient ascent algorithm as follows:

:::{prf:definition} Gradient ascent
$$
\begin{pmatrix}
x^{k+1} \\ z^{k+1}
\end{pmatrix}
= 
\begin{pmatrix}
x^{k} \\ z^{k}
\end{pmatrix}
+
\eta \nabla y(x^{k}, z^{k})
$$
:::

where $k$ denotes the iteration of the algorithm and $\eta > 0$ is a "step size" hyperparameter that controls the size of the steps we take.
(Note that we could also vary the step size across iterations, that is, $\eta^0, \dots, \eta^K$.)

The case of a two-dimensional input is easy to visualize.
But this idea can be straightforwardly extended to higher-dimensional inputs.

From now on, we'll use $J$ to denote the function we're trying to maximize,
and $\theta$ to denote the parameters being optimized over. (In the above example, $\theta = \begin{pmatrix} x & z \end{pmatrix}^\top$).

Notice that our parameters will stop changing once $\nabla J(\theta) = 0.$
Once we reach this **stationary point,** our current parameters are 'locally optimal' in some sense;
it's impossible to increase the function by moving in any direction.
If $J$ is _convex_, then the only point where this happens is at the *global optimum.*
Otherwise, if $J$ is nonconvex, the best we can hope for is a *local optimum.*

:::{note}
How does a computer compute the gradient of a function?

One way is _symbolic differentiation,_
which is similar to the way you might compute it by hand:
the computer applies a list of rules to transform the _symbols_ involved.
Python's `sympy` package supports symbolic differentiation.
However, functions implemented in code may not always have a straightforward symbolic representation.

Another way is _numerical differentiation,_
which is based on the limit definition of a (directional) derivative:

$$
\nabla_{\boldsymbol{u}} J(\boldsymbol{x}) = \lim_{\varepsilon \to 0}
\frac{J(\boldsymbol{x} + \varepsilon \boldsymbol{u}) - J(\boldsymbol{x})}{\varepsilon}
$$

Then, we can substitute a small value of $\varepsilon$ on the r.h.s. to approximate the directional derivative.
How small, though? If we need an accurate estimate,
we may need such a small value of $\varepsilon$ that typical computers will run into rounding errors.
Also, to compute the full gradient,
we would need to compute the r.h.s. once for each input dimension.
This is an issue if computing $J$ is expensive.

**Automatic differentiation** achieves the best of both worlds.
Like symbolic differentiation,
we manually implement the derivative rules for a few basic operations.
However, instead of executing these on the _symbols_,
we execute them on the _values_ when the function gets called,
like in numerical differentiation.
This allows us to differentiate through programming constructs such as branches or loops,
and doesn't involve any arbitrarily small values.
:::

+++

### Stochastic gradient ascent

In real applications,
computing the gradient of the target function is not so simple.
As an example from supervised learning, $J(\theta)$ might be the sum of squared prediction errors across an entire training dataset.
However, if our dataset is very large, it might not fit into our computer's memory!
In these cases, we often compute some _estimate_ of the gradient at each step, $\tilde \nabla J(\theta)$, and walk in that direction instead.
This is called **stochastic** gradient ascent.
In the SL example above, we might randomly choose a *minibatch* of samples and use them to estimate the true prediction error. (This approach is known as **_minibatch_ SGD**.)

```{code-cell} ipython3
def sgd(
    θ_init: Array,
    estimate_gradient: Callable[[Array], Array],
    η: float,
    n_steps: int,
):
    """Perform `n_steps` steps of SGD.

    `estimate_gradient` eats the current parameters and returns an estimate of the objective function's gradient at those parameters.
    """
    θ = θ_init
    for step in range(n_steps):
        θ += η * estimate_gradient(θ)
    return θ
```

What makes one gradient estimator better than another?
Ideally, we want this estimator to be **unbiased;** that is, on average, it matches a single true gradient step:

$$
\E [\tilde \nabla J(\theta)] = \nabla J(\theta).
$$

We also want the _variance_ of the estimator to be low so that its performance doesn't change drastically at each step.

We can actually show that, for many "nice" functions, in a finite number of steps, SGD will find a $\theta$ that is "close" to a stationary point.
In another perspective, for such functions, the local "landscape" of $J$ around $\theta$ becomes flatter and flatter the longer we run SGD.

:::{note} SGD convergence
More formally, suppose we run SGD for $K$ steps, using an unbiased gradient estimator.
Let the step size $\eta^k$ scale as $O(1/\sqrt{k}).$
Then if $J$ is bounded and $\beta$-smooth (see below),
and the _norm_ of the gradient estimator has a bounded second moment $\sigma^2,$

$$\|\nabla J(\theta^K)\|^2 \le O \left( M \beta \sigma^2 / K\right).$$

We call a function $\beta$-smooth if its gradient is Lipschitz continuous with constant $\beta$:

$$\|\nabla J(\theta) - \nabla J(\theta')\| \le \beta \|\theta - \theta'\|.$$
:::

We'll now see a concrete application of gradient ascent in the context of policy optimization.

+++

## Policy (stochastic) gradient ascent

Remember that in RL, the primary goal is to find the _optimal policy_ that achieves the maximimum total reward, which we can express using the value function we defined in {prf:ref}`value`:

:::{math}
:label: objective_fn

\begin{aligned}
    J(\pi) := \E_{s_0 \sim \mu_0} V^{\pi} (s_0) = & \E \sum_{\hi=0}^{\hor-1} r_\hi \\
    \text{where} \quad & s_0 \sim \mu_0 \\
    & s_{t+1} \sim P(s_\hi, a_\hi), \\
    & a_\hi = \pi(s_\hi) \\
    & r_\hi = r(s_\hi, a_\hi).
\end{aligned}
:::

(Note that we'll continue to work in the *undiscounted, finite-horizon case.* Analogous results hold for the *discounted, infinite-horizon case.*)

As shown by the notation, this is exactly the function $J$ that we want to maximize using gradient ascent.
What does $\theta$ correspond to, though?
In general, $\pi$ is a function, and optimizing over the space of arbitrary input-output mappings would be intractable.
Instead, we need to describe $\pi$ in terms of some finite set of _parameters_ $\theta$.

+++

(parameterizations)=
### Example policy parameterizations

What are some ways we could parameterize our policy?

+++

#### Tabular representation

If both the state and action spaces are finite, perhaps we could simply learn a preference value $\theta_{s,a}$ for each state-action pair.
Then to turn this into a valid distribution, we perform a **softmax** operation:
we exponentiate each of them,
and then normalize to form a valid distribution:

$$\pi^\text{softmax}_\theta(a | s) = \frac{\exp(\theta_{s,a})}{\sum_{s,a'} \exp (\theta_{s,a'})}.$$

However, this doesn't make use of any structure in the states or actions,
so while this is flexible, it is also prone to overfitting.

#### Linear in features

Another approach is to map each state-action pair into some **feature space** $\phi(s, a) \in \mathbb{R}^p$. Then, to map a feature vector to a probability, we take a linear combination of the features and take a softmax:

$$\pi^\text{linear in features}_{\theta}(a|s) = \frac{\exp(\theta^\top \phi(s, a))}{\sum_{a'} \exp(\theta^\top \phi(s, a'))}.$$

Another interpretation is that $\theta$ represents the feature vector of the "desired" state-action pair, as state-action pairs whose features align closely with $\theta$ are given higher probability.

The score function for this parameterization is also quite elegant:

$$
\begin{aligned}
        \nabla \log \pi_\theta(a|s) &= \nabla \left( \theta^\top \phi(s, a) - \log \left( \sum_{a'} \exp(\theta^\top \phi(s, a')) \right) \right) \\
        &= \phi(s, a) - \E_{a' \sim \pi_\theta(s)} \phi(s, a')
\end{aligned}
$$
    
Plugging this into our policy gradient expression, we get

$$\begin{aligned}
    \nabla J(\theta) & = \E_{\tau \sim \rho_\theta} \left[
    \sum_{t=0}^{T-1} \nabla \log \pi_\theta(a_\hi | s_\hi) A_\hi^{\pi_\theta}
    \right]                                                                                                                    \\
                     & = \E_{\tau \sim \rho_\theta} \left[
    \sum_{t=0}^{T-1} \left( \phi(s_\hi, a_\hi) - \E_{a' \sim \pi(s_\hi)} \phi(s_\hi, a') \right) A_\hi^{\pi_\theta}(s_\hi, a_\hi)
    \right]                                                                                                                    \\
                     & = \E_{\tau \sim \rho_\theta} \left[ \sum_{t=0}^{T-1} \phi(s_\hi, a_\hi) A_\hi^{\pi_\theta} (s_\hi, a_\hi) \right]
\end{aligned}
$$

Why can we drop the $\E \phi(s_\hi, a')$ term? By linearity of expectation, consider the dropped term at a single timestep: $\E_{\tau \sim \rho_\theta} \left[ \left( \E_{a' \sim \pi(s_\hi)} \phi(s, a') \right) A_\hi^{\pi_\theta}(s_\hi, a_\hi) \right].$ By Adam's Law, we can wrap the advantage term in a conditional expectation on the state $s_\hi.$ Then we already know that $\E_{a \sim \pi(s)} A_\hi^{\pi}(s, a) = 0,$ and so this entire term vanishes.

#### Neural policies

More generally, we could map states and actions to unnormalized scores via some parameterized function $f_\theta : \mathcal{S} \times \mathcal{A} \to \mathbb{R},$ such as a neural network, and choose actions according to a softmax: $$\pi^\text{general}_\theta(a|s) = \frac{\exp(f_{\theta}(s,a))}{\sum_{a'} \exp(f_{\theta}(s,a'))}.$$

The score can then be written as $$\nabla \log \pi_\theta(a|s) = \nabla f_\theta(s, a) - \E_{a \sim \pi_\theta(s)} \nabla f_\theta (s, a')$$

+++

### Continuous action spaces

Consider a continuous $n$-dimensional action space $\mathcal{A} = \mathbb{R}^n$. Then for a stochastic policy, we could use a function to predict the *mean* action and then add some random noise about it. For example, we could use a neural network to predict the mean action $\mu_\theta(s)$ and then add some noise $\epsilon \sim \mathcal{N}(0, \sigma^2 I)$ to it:

$$\pi_\theta(a|s) = \mathcal{N}(\mu_\theta(s), \sigma^2 I).$$

<!-- **Exercise:** Can you extend the "linear in features" policy to continuous action spaces in a similar way? -->

+++

Now that we have seen parameterized policies, we can now write the total reward in terms of the parameters:

$$J(\theta) = \E_{\tau \sim \rho_\theta} R(\tau).$$

Now how do we maximize this function (the expected total reward) over the parameters?
One simple idea would be to directly apply gradient ascent:

$$
\theta^{k+1} = \theta^k + \eta \nabla J(\theta^k).
$$

In order to apply this technique, we need to be able to evaluate the gradient $\nabla J(\theta).$
But $J(\theta)$ is very difficult, or even intractable, to compute exactly, since it involves taking an expectation over all possible trajectories $\tau.$
Can we rewrite it in a form that's more convenient to implement?

+++

(importance_sampling)=
### Importance Sampling

There is a general trick called **importance sampling** for evaluating such expectations.
Suppose we want to estimate $\E_{x \sim p}[f(x)]$ where $p$ is hard or expensive to sample from. We can, however, evaluate the likelihood $p(x)$.
Suppose that we _can_ sample from a different distribution $q$.
Since an expectation is just a weighted average, we can sample $x$ from $q$, compute $f(x)$, and then reweight the results:
if $x$ is very likely under $p$ but unlikely under $q$,
we should boost its weighting,
and if it is common under $q$ but uncommon under $p$,
we should lower its weighting.
The reweighting factor is exactly the **likelihood ratio** between the target distribution $p$ and the sampling distribution $q$:

$$
\E_{x \sim p}[f(x)] = \sum_{x \in \mathcal{X}} f(x) p(x) = \sum_{x \in \mathcal{X}} f(x) \frac{p(x)}{q(x)} q(x) = \E_{x \sim q} \left[ \frac{p(x)}{q(x)} f(x) \right].
$$

Doesn't this seem too good to be true? If there were no drawbacks, we could use this to estimate *any* expectation of any function on any arbitrary distribution! The drawback is that the variance may be very large due to the likelihood ratio term.
If there are values of $x$ that are very rare in the sampling distribution $q$,
but common under $p$,
then the likelihood ratio $p(x)/q(x)$ will cause the variance to blow up.

## The REINFORCE policy gradient

Returning to RL, suppose there is some trajectory distribution $\rho(\tau)$ that is **easy to sample from,** such as a database of existing trajectories.
We can then rewrite $\nabla J(\theta)$, a.k.a. the *policy gradient*, as follows.
All gradients are being taken with respect to $\theta$.

$$
\begin{aligned}
    \nabla J(\theta) & = \nabla \E_{\tau \sim \rho_\theta} [ R(\tau) ]                                                                                         \\
                     & = \nabla \E_{\tau \sim \rho} \left[ \frac{\rho_\theta(\tau)}{\rho(\tau)} R(\tau) \right] &  & \text{likelihood ratio trick}             \\
                     & = \E_{\tau \sim \rho} \left[ \frac{\nabla \rho_\theta(\tau)}{\rho(\tau)} R(\tau) \right] &  & \text{switching gradient and expectation}
\end{aligned}
$$

Note that for $\rho = \rho_\theta$, the inside term becomes

$$
\nabla J(\theta) = \E_{\tau \sim \rho_\theta} [ \nabla \log \rho_\theta(\tau) \cdot R(\tau)].
$$

(The order of operations is $\nabla (\log \rho_\theta)(\tau)$.)

Note that when the state transitions are Markov (i.e. $s_{t}$ only depends on $s_{t-1}, a_{t-1}$) and the policy is time-homogeneous (i.e. $a_\hi \sim \pi_\theta (s_\hi)$), we can write out the *likelihood of a trajectory* under the policy $\pi_\theta$:

:::{math}
:label: trajectory_likelihood

\begin{aligned}
        \rho_\theta(\tau) &= \mu(s_0) \pi_\theta(a_0 | s_0) \\
        &\qquad \times P(s_1 | s_0, a_0) \pi_\theta(a_1 | s_1) \\
        &\qquad \times \cdots \\
        &\qquad \times P(s_{H-1} | s_{H-2}, a_{H-2}) \pi_\theta(a_{H-1} | s_{H-1}).
\end{aligned}
:::

Note that the log-trajectory-likelihood turns into a sum of terms,
of which only the $\pi_\theta(a_\hi | s_\hi)$ terms depend on $\theta,$
so we can simplify even further to obtain the following expression for the policy gradient, known as the "REINFORCE" policy gradient:

:::{math}
:label: reinforce_pg

\begin{aligned}
    \nabla J(\theta) = \E_{\tau \sim \rho_\theta} \left[ \sum_{t=0}^{T-1} \nabla_\theta \log \pi_{\theta}(a_\hi | s_\hi) R(\tau) \right]
\end{aligned}
:::

This expression allows us to estimate the gradient by sampling a few sample trajectories from $\pi_\theta,$
calculating the likelihoods of the chosen actions,
and substituting these into the expression above.
We can then use this gradient estimate to apply stochastic gradient ascent.

```python
def estimate_gradient_reinforce_pseudocode(env, π, θ):
    τ = sample_trajectory(env, π(θ))
    gradient_hat = 0
    for s, a, r in τ:
        def policy_log_likelihood(θ):
            return log(π(θ)(s, a))
        gradient_hat += jax.grad(policy_log_likelihood)(θ) * τ.total_reward
    return gradient_hat
```

In fact, we can perform one more simplification.
Intuitively, the action taken at step $t$ does not affect the reward from previous timesteps, since they're already in the past!
You can also show rigorously that this is the case,
and that we only need to consider the present and future rewards to calculate the policy gradient:

:::{math}
:label: pg_with_q

\begin{aligned}
        \nabla J(\theta) &= \E_{\tau \sim \rho_\theta} \left[ \sum_{t=0}^{T-1} \nabla_\theta \log \pi_{\theta}(a_\hi | s_\hi) \sum_{t' = t}^{T-1} r(s_{t'}, a_{t'}) \right] \\
        &= \E_{\tau \sim \rho_\theta} \left[ \sum_{t=0}^{T-1} \nabla_\theta \log \pi_{\theta}(a_\hi | s_\hi) Q^{\pi_\theta}(s_{t}, a_{t}) \right]
\end{aligned}
:::

**Exercise:** Prove that this is equivalent to the previous definitions. What modification to the expression must be made for the discounted, infinite-horizon setting?

For some intuition into how this method works, recall that we update our parameters according to

$$
\begin{aligned}
    \theta_{t+1} &= \theta_\hi + \eta \nabla J(\theta_\hi) \\
    &= \theta_\hi + \eta \E_{\tau \sim \rho_{\theta_\hi}} [\nabla \log \rho_{\theta_\hi}(\tau) \cdot R(\tau)].
\end{aligned}
$$

Consider the "good" trajectories where $R(\tau)$ is large. Then $\theta$ gets updated so that these trajectories become more likely. To see why, recall that $\rho_{\theta}(\tau)$ is the likelihood of the trajectory $\tau$ under the policy $\pi_\theta,$ so evaluating the gradient points in the direction that makes $\tau$ more likely.

+++

## Baselines and advantages

A central idea from supervised learning is the **bias-variance decomposition**,
which shows that the mean squared error of an estimator is the sum of its squared bias and its variance.
The REINFORCE gradient estimator {eq}`reinforce_pg` is already *unbiased,* meaning that its expectation over trajectories is the true policy gradient.
Can we find ways to reduce its _variance_ as well?

One common way is to subtract a **baseline function** $b_\hi : \mathcal{S} \to \mathbb{R}$ at each timestep $\hi.$ This modifies the policy gradient as follows:

$$
\nabla J(\theta) = \E_{\tau \sim \rho_\theta} \left[
    \sum_{\hi=0}^{H-1} \nabla \log \pi_\theta (a_\hi | s_\hi) \left(
    \left(
    \sum_{\hi' = \hi}^{H-1} r_{\hi'}
    \right)
    - b_\hi(s_\hi)
    \right)
    \right].
\label{eq:pg_baseline}
$$

For example, we might want $b_\hi$ to estimate the average reward-to-go at a given timestep:

$$b_\hi^\theta = \E_{\tau \sim \rho_\theta} R_\hi(\tau).$$

This way, the random variable $R_\hi(\tau) - b_\hi^\theta$ is centered around zero, making certain algorithms more stable.

As a better baseline, we could instead choose the *value function.*
Note that the random variable $Q^\pi_\hi(s, a) - V^\pi_\hi(s),$
where the randomness is taken over the actions, is also centered around zero.
(Recall $V^\pi_\hi(s) = \E_{a \sim \pi} Q^\pi_\hi(s, a).$)
In fact, this quantity has a particular name: the **advantage function.**
This measures how much better this action does than the average for that policy.
(Note that for an optimal policy $\pi^\star,$ the advantage of a given state-action pair is always zero or negative.)

We can now express the policy gradient as follows. Note that the advantage function effectively replaces the $Q$-function from {eq}`pg_with_q`:

:::{math}
:label: pg_advantage

\nabla J(\theta) = \E_{\tau \sim \rho_\theta} \left[
        \sum_{t=0}^{T-1} \nabla \log \pi_\theta(a_\hi | s_\hi) A^{\pi_\theta}_\hi (s_\hi, a_\hi)
\right].
:::

Note that to avoid correlations between the gradient estimator and the value estimator (i.e. baseline), we must estimate them with independently sampled trajectories:

<!-- TODO could use more explanation _why_ we want to avoid correlations -->

::::{prf:definition} Policy gradient with a learned baseline
:label: pg_baseline

```python
def pg_with_learned_baseline_pseudocode(env, π, η, θ_init, K, N):
    θ = θ_init
    for k in range(K):
        trajectories = sample_trajectories(env, π(θ), N)
        V_hat = fit(trajectories)  # estimates the value function of π(θ)
        τ = sample_trajectories(env, π(θ), 1)
        g = jnp.zeros_like(θ)  # gradient estimator

        for h, (s, a) in enumerate(τ):
            def log_likelihood(θ_):
                return jnp.log(π(θ_)(s, a))
            g = g + jax.grad(log_likelihood)(θ) * (return_to_go(τ, h) - V_hat(s))
        
        θ = θ + η * g
    return θ
```

Note that you could also generalize this by allowing the learning rate $\eta$ to vary across steps,
or take multiple trajectories $\tau$ and compute the sample average of the gradient estimates.

The baseline estimation step `fit` can be done using any appropriate supervised learning algorithm.
Note that the gradient estimator will be unbiased regardless of the baseline.
::::

+++

## Comparing policy gradient algorithms to policy iteration

<!-- TODO maybe restructure this part -->

What advantages does the policy gradient algorithm have over [](#policy_iteration)?

:::{note} Policy iteration recap
Recall that policy iteration is an algorithm for MDPs with unknown state transitions where we alternate between these two steps:

- Estimating the $Q$-function (or advantage function) of the current policy;
- Updating the policy to be greedy w.r.t. this approximate $Q$-function (or advantage function).
:::

To analyze the difference between them, we'll make use of the **performance difference lemma**, which provides an expression for comparing the difference between two value functions.

::::{prf:theorem} Performance difference lemma
:label: pdl

Suppose Alice is playing a game (an MDP).
Bob is spectating, and can evaluate how good an action is compared to his own strategy.
(That is, Bob can compute his _advantage function_ $A_\hi^{\text{Bob}}(s_\hi, a_\hi)$).
The performance difference lemma says that Bob can now calculate exactly how much better or worse he is than Alice as follows:

:::{math}
:label: pdl_eq
V_0^{\text{Alice}}(s) - V_0^{\text{Bob}}(s) = \E_{\tau \sim \rho_{\text{Alice}, s}} \left[ \sum_{h=0}^{H-1} A_\hi^{\text{Bob}} (s_\hi, a_\hi) \right]
:::

where $\rho_{\text{Alice}, s}$ denotes the distribution over trajectories starting in state $s$ when Alice is playing.

To see why, consider just a single step $\hi$ of the trajectory.
At this step we compute how much better actions from Bob are than the actions from Alice, on average.
But this is exactly the average Bob-advantage across actions from Alice, as described in the PDL!

Formally, this corresponds to a nice telescoping simplification when we expand out the definition of the advantage function. Note that

$$
\begin{aligned}
A^\pi_\hi(s_\hi, a_\hi) &= Q^\pi_\hi(s_\hi, a_\hi) - V^\pi_\hi(s_\hi) \\
&= r_\hi(s_\hi, a_\hi) + \E_{s_{\hi+1} \sim P(s_\hi, a_\hi)} [V^\pi_{\hi+1}(s_{\hi+1})] - V^\pi_\hi(s_\hi)
\end{aligned}
$$

so expanding out the r.h.s. expression of {eq}`pdl_eq` and grouping terms together gives

$$
\begin{aligned}
\E_{\tau \sim \rho_{\text{Alice}, s}} \left[ \sum_{\hi=0}^{\hor-1} A_\hi^{\text{Bob}} (s_\hi, a_\hi) \right] &= \E_{\tau \sim \rho_{\text{Alice}, s}} \left[ \left( \sum_{\hi=0}^{\hor-1} r_\hi(s_\hi, a_\hi) \right) + \left( V^{\text{Bob}}_1(s_1) + \cdots + V^{\text{Bob}}_\hor(s_\hor) \right) - \left( V^{\text{Bob}_0}(s_0) + \cdots + V^{\text{Bob}}_{\hor-1}(s_{\hor-1}) \right) \right] \\
&= V^{\text{Alice}}_0(s) - V^{\text{Bob}}_0(s)
\end{aligned}
$$

as desired. (Note that the "inner" expectation from expanding the advantage function has the same distribution as the outer one, so omitting it here is valid.)
::::

The PDL gives insight into why fitted approaches such as PI don't work as well in the "full" RL setting.
To see why, let's consider a single iteration of policy iteration, where policy $\pi$ gets updated to $\tilde \pi$. We'll assume these policies are deterministic.
Suppose the new policy $\tilde \pi$ chooses some action with a negative advantage with respect to $\pi$.
That is, when acting according to $\pi$, taking the action from $\tilde \pi$ would perform worse than expected.
Define $\Delta_\infty$ to be the most negative advantage, that is, $\Delta_\infty = \min_{s \in \mathcal{S}} A^{\pi}_\hi(s, \tilde \pi(s))$.
Plugging this into the {prf:ref}`pdl` gives

$$
\begin{aligned}
V_0^{\tilde \pi}(s) - V_0^{\pi}(s) &= \E_{\tau \sim \rho_{\tilde \pi, s}} \left[
\sum_{\hi=0}^{\hor-1} A_\hi^{\pi}(s_\hi, a_\hi)
\right] \\
&\ge H \Delta_\infty \\
V_0^{\tilde \pi}(s) &\ge V_0^{\pi}(s) - H|\Delta_\infty|.
\end{aligned}
$$

That is, for some state $s$, the lower bound on the performance of $\tilde \pi$ is _lower_ than the performance of $\pi$.
This doesn't state that $\tilde \pi$ _will_ necessarily perform worse than $\pi$,
only suggests that it might be possible.
If these worst case states do exist, though,
PI does not avoid situations where the new policy often visits them;
It does not enforce that the trajectory distributions $\rho_\pi$ and $\rho_{\tilde \pi}$ be close to each other.
In other words, the "training distribution" that our prediction rule is fitted on, $\rho_\pi$, may differ significantly from the "evaluation distribution" $\rho_{\tilde \pi}$.

<!-- 
This is an instance of *distributional shift*.
To begin, let's ask, where *do* fitted approaches work well?
They are commonly seen in SL,
where a prediction rule is fit using some labelled training set,
and then assessed on a test set from the same distribution.
But policy iteration isn't performed in the same scenario:
there is now _distributional shift_ between the different iterations of the policy. -->

On the other hand, policy gradient methods _do_, albeit implicitly,
encourage $\rho_\pi$ and $\rho_{\tilde \pi}$ to be similar.
Suppose that the mapping from policy parameters to trajectory distributions is relatively smooth.
Then, by adjusting the parameters only a small distance,
the new policy will also have a similar trajectory distribution.
But this is not very rigorous, and in practice the parameter-to-distribution mapping may not be so smooth.
Can we constrain the distance between the resulting distributions more _explicitly_?

This brings us to the next three methods:
- **trust region policy optimization** (TRPO), which explicitly constrains the difference between the distributions before and after each step;
- the **natural policy gradient** (NPG), a first-order approximation of TRPO;
- **proximal policy optimization** (PPO), a "soft relaxation" of TRPO.

+++

## Trust region policy optimization

We saw above that policy gradient methods are effective because they implicitly constrain how much the policy changes at each iteration.
Can we design an algorithm that _explicitly_ constrains the "step size"?
That is, we want to _improve_ the policy as much as possible,
measured in terms of the r.h.s. of the {prf:ref}`pdl`,
while ensuring that its trajectory distribution does not change too much:

$$
\begin{aligned}
\theta^{k+1} &\gets \arg\max_{\theta^{\text{opt}}} \E_{s_0, \dots, s_{H-1} \sim \pi^{k}} \left[ \sum_{\hi=0}^{\hor-1} \E_{a_\hi \sim \pi^{\theta^\text{opt}}(s_\hi)} A^{\pi^{k}}(s_\hi, a_\hi) \right] \\
& \text{where } \text{distance}(\rho_{\theta^{\text{opt}}}, \rho_{\theta^k}) < \delta
\end{aligned}
$$

Note that we have made a small change to the r.h.s. expression:
we use the *states* sampled from the old policy, and only use the *actions* from the new policy.
It would be computationally infeasible to sample entire trajectories from $\pi_\theta$ as we are optimizing over $\theta$.
On the other hand, if $\pi_\theta$ returns a vector representing a probability distribution over actions,
then evaluating the expected advantage with respect to this distribution only requires taking a dot product.
This approximation also matches the r.h.s. of the PDL to first order in $\theta$.
(We will elaborate more on this later.)

How do we describe the distance between $\rho_{\theta^{\text{opt}}}$ and $\rho_{\theta^k}$?
We'll use the **Kullback-Leibler divergence (KLD)**:

:::{prf:definition} Kullback-Leibler divergence
:label: kld

For two PDFs $p, q$,

$$\kl{p}{q} := \E_{x \sim p} \left[ \log \frac{p(x)}{q(x)} \right]$$

This can be interpreted in many different ways, many stemming from information theory.
One such interpretation is that $\kl{p}{q}$ describes my average "surprise" if I _think_ data is being generated by $q$ but it's actually generated by $p$.
(The **surprise** of an event with probability $p$ is $- \log_2 p$.)
Note that $\kl{p}{q} = 0$ if and only if $p = q$. Also note that it is generally _not_ symmetric.
:::

Both the objective function and the KLD constraint involve a weighted average over the space of all trajectories.
This is intractable in general, so we need to estimate the expectation.
As before, we can do this by taking an empirical average over samples from the trajectory distribution.
This gives us the following pseudocode:

::::{prf:definition} Trust region policy optimization (exact)
:label: trpo


```python
def trpo_pseudocode(env, δ, θ_init, M):
    θ = θ_init
    for k in range(K):
        trajectories = sample_trajectories(env, π(θ), M)
        A_hat = fit(trajectories)
        
        def approximate_gain(θ_):
            total_advantage = 0
            for τ in trajectories:
                for s, _a, _r in τ:
                    for a in env.action_space:
                        total_advantage += π(θ)(s, a) * A_hat(s, a)
            return total_advantage
        
        def constraint(θ_):
            kl_div = 0
            for τ in trajectories:
                for s, a, _r in τ:
                    kl_div += jnp.log(π(θ)(s, a)) - jnp.log(π(θ_)(s, a))
            return kl_div <= δ
        
        θ = optimize(approximate_gain, constraint)

    return θ
```
::::

<!--
Applying importance sampling allows us to estimate the TRPO objective as follows:

::::{prf:definition} Trust region policy optimization (implementation)
:label: trpo_implement

:::{prf:definitionic} TODO
Initialize $\theta^0$

Sample $N$ trajectories from $\rho^k$ to learn a value estimator $\tilde b_\hi(s) \approx V^{\pi^k}_\hi(s)$

Sample $M$ trajectories $\tau_0, \dots, \tau_{M-1} \sim \rho^k$

$$\begin{gathered}
            \theta^{k+1} \gets \arg\max_{\theta} \frac{1}{M} \sum_{m=0}^{M-1} \sum_{h=0}^{H-1} \frac{\pi_\theta(a_\hi \mid s_\hi)}{\pi^k(a_\hi \mid s_\hi)} [ R_\hi(\tau_m) - \tilde b_\hi(s_\hi) ] \\
            \text{where } \sum_{m=0}^{M-1} \sum_{h=0}^{H-1} \log \frac{\pi_k(a_\hi^m \mid s_\hi^m)}{\pi_\theta(a_\hi^m \mid s_\hi^m)} \le \delta
        
\end{gathered}$$
:::
:::: -->

The above isn't entirely complete:
we still need to solve the actual optimization problem at each step.
Unless we know additional properties of the problem,
this might be an intractable optimization.
Do we need to solve it exactly, though?
Instead, if we assume that both the objective function and the constraint are somewhat smooth in terms of the policy parameters,
we can use their _Taylor expansions_ to give us a simpler optimization problem with a closed-form solution.
This brings us to the **natural policy gradient** algorithm.

+++

## Natural policy gradient

We take a _linear_ (first-order) approximation to the objective function and a _quadratic_ (second-order) approximation to the KL divergence constraint about the current estimate $\theta^k$.
This results in the optimization problem

:::{math}
:label: npg_optimization

\begin{gathered}
    \max_\theta \nabla_\theta J(\pi_{\theta^k})^\top (\theta - \theta^k) \\
    \text{where } \frac{1}{2} (\theta - \theta^k)^\top F_{\theta^k} (\theta - \theta^k) \le \delta
\end{gathered}
:::

where $F_{\theta^k}$ is the **Fisher information matrix** defined below.

::::{prf:definition} Fisher information matrix
:label: fisher_matrix

Let $p_\theta$ denote a parameterized distribution.
Its Fisher information matrix $F_\theta$ can be defined equivalently as:

$$
\begin{aligned}
        F_{\theta} & = \E_{x \sim p_\theta} \left[ (\nabla_\theta \log p_\theta(x)) (\nabla_\theta \log p_\theta(x))^\top \right] & \text{covariance matrix of the Fisher score}          \\
                   & = \E_{x \sim p_{\theta}} [- \nabla_\theta^2 \log p_\theta(x)]                                                & \text{average Hessian of the negative log-likelihood}
\end{aligned}
$$

Recall that the Hessian of a function describes its curvature:
for a vector $\delta \in \Theta$,
the quantity $\delta^\top F_\theta \delta$ describes how rapidly the negative log-likelihood changes if we move by $\delta$.
The Fisher information matrix is precisely the Hessian of the KL divergence (with respect to either one of the parameters).

In particular, when $p_\theta = \rho_{\theta}$ denotes a trajectory distribution, we can further simplify the expression:

:::{math}
:label: fisher_trajectory

F_{\theta} = \E_{\tau \sim \rho_\theta} \left[ \sum_{h=0}^{H-1} (\nabla \log \pi_\theta (a_\hi \mid s_\hi)) (\nabla \log \pi_\theta(a_\hi \mid s_\hi))^\top \right]
:::
        
Note that we've used the Markov property to cancel out the cross terms corresponding to two different time steps.
::::

This is a convex optimization problem with a closed-form solution.
To see why, it helps to visualize the case where $\theta$ is two-dimensional:
the constraint describes the inside of an ellipse,
and the objective function is linear,
so we can find the extreme point on the boundary of the ellipse.
We recommend {cite}`boyd_convex_2004` for a comprehensive treatment of convex optimization.

More generally, for a higher-dimensional $\theta$,
we can compute the global optima by setting the gradient of the Lagrangian to zero:

$$
\begin{aligned}
    \mathcal{L}(\theta, \alpha)                     & = \nabla J(\pi_{\theta^k})^\top (\theta - \theta^k) - \alpha \left[ \frac{1}{2} (\theta - \theta^k)^\top F_{\theta^k} (\theta - \theta^k) - \delta \right] \\
    \nabla \mathcal{L}(\theta^{k+1}, \alpha) & := 0                                                                                                                                                             \\
    \implies \nabla J(\pi_{\theta^k})        & = \alpha F_{\theta^k} (\theta^{k+1} - \theta^k)                                                                                                                   \\
    \theta^{k+1}                           & = \theta^k + \eta F_{\theta^k}^{-1} \nabla J(\pi_{\theta^k})                                                                                             \\
    \text{where } \eta                     & = \sqrt{\frac{2 \delta}{\nabla J(\pi_{\theta^k})^\top F_{\theta^k}^{-1} \nabla J(\pi_{\theta^k})}}
\end{aligned}
$$

This gives us the closed-form update.
Now the only challenge is to estimate the Fisher information matrix,
since, as with the KL divergence constraint, it is an expectation over trajectories, and computing it exactly is therefore typically intractable.

::::{prf:definition} Natural policy gradient
:label: npg

How many trajectory samples do we need to accurately estimate the Fisher information matrix?
As a rule of thumb, the sample complexity should scale with the dimension of the parameter space.
This makes this approach intractable in the deep learning setting where we might have a very large number of parameters.
::::

As you can see, the NPG is the "basic" policy gradient algorithm we saw above,
but with the gradient transformed by the inverse Fisher information matrix.
This matrix can be understood as accounting for the **geometry of the parameter space.**
The typical gradient descent algorithm implicitly measures distances between parameters using the typical _Euclidean distance_.
Here, where the parameters map to a *distribution*, using the natural gradient update is equivalent to optimizing over **distribution space** rather than parameter space,
where distance between distributions is measured by the {prf:ref}`kld`.

::::{prf:example} Natural gradient on a simple problem
:label: natural_simple

Let's step away from RL and consider the following optimization problem over Bernoulli distributions $\pi \in \Delta(\{ 0, 1 \})$:

$$
\begin{aligned}
        J(\pi) & = 100 \cdot \pi(1) + 1 \cdot \pi(0)
\end{aligned}
$$

We can think of the space of such distributions as the line between $(0, 1)$ to $(1, 0)$ on the Cartesian plane:

:::{image} shared/npg_line.png
:alt: a line from (0, 1) to (1, 0)
:width: 240px
:align: center
:::

Clearly the optimal distribution is the constant one $\pi(1) = 1$. Suppose we optimize over the parameterized family $\pi_\theta(1) = \frac{\exp(\theta)}{1+\exp(\theta)}$.
Then our optimization algorithm should set $\theta$ to be unboundedly large.
Then the "vanilla" gradient is

$$\nabla_\theta J(\pi_\theta) = \frac{99 \exp(\theta)}{(1 + \exp(\theta))^2}.$$

Note that as $\theta \to \infty$ that the increments get closer and closer to $0$;
the rate of increase becomes exponentially slow.


However, if we compute the Fisher information "matrix" (which is just a scalar in this case), we can account for the geometry induced by the parameterization.

$$
\begin{aligned}
        F_\theta & = \E_{x \sim \pi_\theta} [ (\nabla_\theta \log \pi_\theta(x))^2 ] \\
                 & = \frac{\exp(\theta)}{(1 + \exp(\theta))^2}.
\end{aligned}
$$

This gives the natural gradient update

$$
\begin{aligned}
        \theta^{k+1} & = \theta^k + \eta F_{\theta^k}^{-1} \nabla_ \theta J(\theta^k) \\
                     & = \theta^k + 99 \eta
\end{aligned}
$$

which increases at a constant rate, i.e. improves the objective more quickly than "vanilla" gradient ascent.
::::

Though the NPG now gives a closed-form optimization step,
it requires computing the inverse Fisher information matrix,
which typically scales as $O((\dim \Theta)^3)$.
This can be expensive if the parameter space is large.
Can we find an algorithm that works in _linear time_ with respect to the dimension of the parameter space?

+++

## Proximal policy optimization

We can relax the TRPO optimization problem in a different way:
Rather than imposing a hard constraint on the KL distance,
we can instead impose a *soft* constraint by incorporating it into the objective and penalizing parameter values that drastically change the trajectory distribution.

$$
\begin{aligned}
\theta^{k+1} &\gets \arg\max_{\theta} \E_{s_0, \dots, s_{H-1} \sim \rho_{\pi^{k}}} \left[ \sum_{\hi=0}^{\hor-1} \E_{a_\hi \sim \pi_{\theta}(s_\hi)} A^{\pi^{k}}(s_\hi, a_\hi) \right] - \lambda \kl{\rho_{\theta}}{\rho_{\theta^k}}
\end{aligned}
$$

Here $\lambda$ is a **regularization hyperparameter** that controls the tradeoff between the two terms.

Like the original TRPO algorithm {prf:ref}`trpo`, PPO is not gradient-based; rather, at each step, we try to maximize local advantage relative to the current policy.

How do we solve this optimization?
Let us begin by simplifying the $\kl{\rho_{\pi^k}}{\rho_{\pi_{\theta}}}$ term. Expanding gives

$$
\begin{aligned}
    \kl{\rho_{\pi^k}}{\rho_{\pi_{\theta}}} & = \E_{\tau \sim \rho_{\pi^k}} \left[\log \frac{\rho_{\pi^k}(\tau)}{\rho_{\pi_{\theta}}(\tau)}\right]                                                       \\
                                           & = \E_{\tau \sim \rho_{\pi^k}} \left[ \sum_{h=0}^{H-1} \log \frac{\pi^k(a_\hi \mid s_\hi)}{\pi_{\theta}(a_\hi \mid s_\hi)}\right] & \text{state transitions cancel} \\
                                           & = \E_{\tau \sim \rho_{\pi^k}} \left[ \sum_{h=0}^{H-1} \log \frac{1}{\pi_{\theta}(a_\hi \mid s_\hi)}\right] + c
\end{aligned}
$$

where $c$ is some constant with respect to $\theta$, and can be ignored.
This gives the objective

$$
\ell^k(\theta)
=
\E_{s_0, \dots, s_{H-1} \sim \rho_{\pi^{k}}} \left[ \sum_{\hi=0}^{\hor-1} \E_{a_\hi \sim \pi_{\theta}(s_\hi)} A^{\pi^{k}}(s_\hi, a_\hi) \right] - \lambda \E_{\tau \sim \rho_{\pi^k}} \left[ \sum_{h=0}^{H-1} \log \frac{1}{\pi_{\theta}(a_\hi \mid s_\hi)}\right]
$$

Once again, this takes an expectation over trajectories.
But here we cannot directly sample trajectories from $\pi^k$,
since in the first term, the actions actually come from $\pi_\theta$.
To make this term line up with the other expectation,
we would need the actions to also come from $\pi^k$.

This should sound familiar:
we want to estimate an expectation over one distribution by sampling from another.
We can once again use [](#importance_sampling) to rewrite the inner expectation:

$$
\E_{a_\hi \sim \pi_{\theta}(s_\hi)} A^{\pi^{k}}(s_\hi, a_\hi)
=
\E_{a_\hi \sim \pi^k(s_\hi)} \frac{\pi_\theta(a_\hi \mid s_\hi)}{\pi^k(a_\hi \mid s_\hi)} A^{\pi^{k}}(s_\hi, a_\hi)
$$

Now we can combine the expectations together to get the objective

$$
\ell^k(\theta) = \E_{\tau \sim \rho_{\pi^k}} \left[ \sum_{h=0}^{H-1} \left( \frac{\pi_\theta(a_\hi \mid s_\hi)}{\pi^k(a_\hi \mid s_\hi)} A^{\pi^k}(s_\hi, a_\hi) - \lambda \log \frac{1}{\pi_\theta(a_\hi \mid s_\hi)} \right) \right]
$$

Now we can estimate this function by a sample average over trajectories from $\pi^k$.
Remember that to complete a single iteration of PPO,
we execute

$$
\theta^{k+1} \gets \arg\max_{\theta} \ell^k(\theta).
$$

If $\ell^k$ is differentiable, we can optimize it by gradient ascent, completing a single iteration of PPO.

```python
def ppo_pseudocode(
    env,
    π: Callable[[Params], Callable[[State, Action], Float]],
    λ: float,
    θ_init: Params,
    n_iters: int,
    n_fit_trajectories: int,
    n_sample_trajectories: int,
):
    θ = θ_init
    for k in range(n_iters):
        fit_trajectories = sample_trajectories(env, π(θ), n_fit_trajectories)
        A_hat = fit(fit_trajectories)

        sample_trajectories = sample_trajectories(env, π(θ), n_sample_trajectories)
        
        def objective(θ_opt):
            total_objective = 0
            for τ in sample_trajectories:
                for s, a, _r in τ:
                    total_objective += π(θ_opt)(s, a) / π(θ)(s, a) * A_hat(s, a) + λ * jnp.log(π(θ_opt)(s, a))
            return total_objective / n_sample_trajectories
        
        θ = optimize(objective, θ)

    return θ
```

## Summary

Policy gradient methods are a powerful family of algorithms that directly optimize the total reward by iteratively updating the policy parameters.

TODO

- Vanilla policy gradient
- Baselines and advantages
- Trust region policy optimization
- Natural policy gradient
- Proximal policy optimization
