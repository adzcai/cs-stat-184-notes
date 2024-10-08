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
  enumerator: 5.%s
---

# 5 Fitted Dynamic Programming Algorithms

## Introduction

We borrow these definitions from the [](./mdps.md) chapter:

```{code-cell}
:tags: [hide-input]

from typing import NamedTuple, Callable, Optional
from jaxtyping import Float, Array
import jax.numpy as np
from jax import grad, vmap
import jax.random as rand
from tqdm import tqdm
import gymnasium as gym

key = rand.PRNGKey(184)


class Transition(NamedTuple):
    s: int
    a: int
    r: float


Trajectory = list[Transition]


def get_num_actions(trajectories: list[Trajectory]) -> int:
    """Get the number of actions in the dataset. Assumes actions range from 0 to A-1."""
    return max(max(t.a for t in τ) for τ in trajectories) + 1


State = Float[Array, "..."]  # arbitrary shape

# assume finite `A` actions and f outputs an array of Q-values
# i.e. Q(s, a, h) is implemented as f(s, h)[a]
QFunction = Callable[[State, int], Float[Array, " A"]]


def Q_zero(A: int) -> QFunction:
    """A Q-function that always returns zero."""
    return lambda s, a: np.zeros(A)


# a deterministic time-dependent policy
Policy = Callable[[State, int], int]


def q_to_greedy(Q: QFunction) -> Policy:
    """Get the greedy policy for the given state-action value function."""
    return lambda s, h: np.argmax(Q(s, h))
```

The [](./mdps.md) chapter discussed the case of **finite** MDPs, where the state and action spaces $\mathcal{S}$ and $\mathcal{A}$ were finite.
This gave us a closed-form expression for computing the r.h.s. of {prf:ref}`the Bellman one-step consistency equation <bellman_consistency>`.
In this chapter, we consider the case of **large** or **continuous** state spaces, where the state space is too large to be enumerated.
In this case, we need to *approximate* the value function and Q-function using methods from **supervised learning**.

We will first take a quick detour to introduce the _empirical risk minimization_ framework for function approximation.
We will then see its application to _fitted_ RL algorithms,
which attempt to learn the optimal value function (and the optimal policy) from a dataset of trajectories.

(erm)=
## Empirical risk minimization

The **supervised learning** task is as follows:
We seek to learn the relationship between some input variables $x$ and some output variable $y$
(drawn from their joint distribution).
Precisely, we want to find a function $\hat f : x \mapsto y$ that minimizes the
_squared error_ of the prediction:

$$
\hat f = \arg\min_{f} \E[(y - f(x))^2]
$$

An equivalent framing is that we seek to approximate the *conditional expectation* of $y$ given $x$:

:::{prf:theorem} Conditional expectation minimizes mean squared error
:label: conditional_expectation_minimizes_mse

$$
\arg\min_{f} \E[(y - f(x))^2] = (x \mapsto \E[y \mid x])
$$
:::

::::{prf:proof}
We can decompose the mean squared error as

$$
\begin{aligned}
\E[(y - f(x))^2] &= \E[ (y - \E[y \mid x] + \E[y \mid x] - f(x))^2 ] \\
&= \E[ (y - \E[y \mid x])^2 ] + \E[ (\E[y \mid x] - f(x))^2 ] + 2 \E[ (y - \E[y \mid x])(\E[y \mid x] - f(x)) ] \\
\end{aligned}
$$

:::{attention}
Use the law of iterated expectations to show that the last term is zero.
:::

The first term is the irreducible error, and the second term is the error due to the approximation,
which is minimized at $0$ when $f(x) = \E[y \mid x]$.
::::

In most applications, the joint distribution of $x, y$ is unknown or extremely complex, and so we can't
analytically evaluate $\E [y \mid x]$.
Instead, our strategy is to draw $N$ samples $(x_i, y_i)$ from the joint distribution of $x$ and $y$,
and then use the _sample average_ $\sum_{i=1}^N (y_i - f(x_i))^2 / N$ to approximate the mean squared error.
Then we use a _fitting method_ to find a function $\hat f$ that minimizes this objective
and thus approximates the conditional expectation.
This approach is called **empirical risk minimization**.

:::{prf:definition} Empirical risk minimization
:label: empirical_risk_minimization

Given a dataset of samples $(x_1, y_1), \dots, (x_N, y_N)$, empirical risk minimization seeks to find a function $f$ (from some class of functions $\mathcal{F}$) that minimizes the empirical risk:

$$
\hat f = \arg\min_{f \in \mathcal{F}} \frac{1}{N} \sum_{i=1}^N (y_i - f(x_i))^2
$$

We will cover the details of the minimization process in [](#the next section <supervised_learning>).
:::

:::{attention}
Why is it important that we constrain our search to a class of functions $\mathcal{F}$?

Hint: Consider the function $f(x) = \sum_{i=1}^N y_i \mathbb{1}_{\{ x = x_i \}}$. What is the empirical risk of this function? Would you consider it a good approximation of the conditional expectation?
:::

## Fitted value iteration

Let us apply ERM to the RL problem of computing the optimal policy / value function.

How did we compute the optimal value function in MDPs with _finite_ state and action spaces?

- In a [](#finite-horizon MDP <finite_horizon_mdps>), we can use {prf:ref}`dynamic programming <pi_star_dp>`, working backwards from the end of the time horizon, to compute the optimal value function exactly.

- In an [](#infinite-horizon MDP <infinite_horizon_mdps>), we can use [](#value iteration <value_iteration>), which iterates the Bellman optimality operator {eq}`bellman_optimality_operator` to approximately compute the optimal value function.

Our existing approaches represent the value function, and the MDP itself,
in matrix notation.
But what happens if the state space is extremely large, or even infinite (e.g. real-valued)?
Then computing a weighted sum over all possible next states, which is required to compute the Bellman operator,
becomes intractable.

Instead, we will need to use *function approximation* methods from supervised learning to solve for the value function in an alternative way.

In particular, suppose we have a dataset of $N$ trajectories $\tau_1, \dots, \tau_N \sim \rho_{\pi}$ from some policy $\pi$ (called the **data collection policy**) acting in the MDP of interest.
Let us indicate the trajectory index in the superscript, so that

$$
\tau_i = \{ s_0^i, a_0^i, r_0^i, s_1^i, a_1^i, r_1^i, \dots, s_{\hor-1}^i, a_{\hor-1}^i, r_{\hor-1}^i \}.
$$

```{code-cell}
def collect_data(
    env: gym.Env, N: int, H: int, key: rand.PRNGKey, π: Optional[Policy] = None
) -> list[Trajectory]:
    """Collect a dataset of trajectories from the given policy (or a random one)."""
    trajectories = []
    seeds = [rand.bits(k).item() for k in rand.split(key, N)]
    for i in tqdm(range(N)):
        τ = []
        s, _ = env.reset(seed=seeds[i])
        for h in range(H):
            # sample from a random policy
            a = π(s, h) if π else env.action_space.sample()
            s_next, r, terminated, truncated, _ = env.step(a)
            τ.append(Transition(s, a, r))
            if terminated or truncated:
                break
            s = s_next
        trajectories.append(τ)
    return trajectories
```

```{code-cell}
env = gym.make("LunarLander-v2")
trajectories = collect_data(env, 100, 300, key)
trajectories[0][:5]  # show first five transitions from first trajectory
```

Can we view the dataset of trajectories as a "labelled dataset" in order to apply supervised learning to approximate the optimal Q-function? Yes!
Recall that we can characterize the optimal Q-function using the {prf:ref}`Bellman optimality equations <bellman_consistency_optimal>`,
which don't depend on an actual policy:

$$
Q_\hi^\star(s, a) = r(s, a) + \E_{s' \sim P(s, a)} [\max_{a'} Q_{\hi+1}^\star(s', a')]
$$

We can think of the arguments to the Q-function -- i.e. the current state, action, and timestep $\hi$ --
as the inputs $x$, and the r.h.s. of the above equation as the label $f(x)$. Note that the r.h.s. can also be expressed as a **conditional expectation**:

$$
f(x) = \E [y \mid x] \quad \text{where} \quad y = r(s_\hi, a_\hi) + \max_{a'} Q^\star_{\hi + 1}(s', a').
$$

Approximating the conditional expectation is precisely the task that [](#erm) is suited for!

Our above dataset would give us $N \cdot \hor$ samples in the dataset:

$$
x_{i \hi} = (s_\hi^i, a_\hi^i, \hi) \qquad y_{i \hi} = r(s_\hi^i, a_\hi^i) + \max_{a'} Q^\star_{\hi + 1}(s_{\hi + 1}^i, a')
$$

```{code-cell}
def get_X(trajectories: list[Trajectory]):
    """
    We pass the state and timestep as input to the Q-function
    and return an array of Q-values.
    """
    rows = [(τ[h].s, τ[h].a, h) for τ in trajectories for h in range(len(τ))]
    return [np.stack(ary) for ary in zip(*rows)]


def get_y(
    trajectories: list[Trajectory],
    f: Optional[QFunction] = None,
    π: Optional[Policy] = None,
):
    """
    Transform the dataset of trajectories into a dataset for supervised learning.
    If `π` is None, instead estimates the optimal Q function.
    Otherwise, estimates the Q function of π.
    """
    f = f or Q_zero(get_num_actions(trajectories))
    y = []
    for τ in trajectories:
        for h in range(len(τ) - 1):
            s, a, r = τ[h]
            Q_values = f(s, h + 1)
            y.append(r + (Q_values[π(s, h + 1)] if π else Q_values.max()))
        y.append(τ[-1].r)
    return np.array(y)
```

```{code-cell}
s, a, h = get_X(trajectories[:1])
print("states:", s[:5])
print("actions:", a[:5])
print("timesteps:", h[:5])
```

```{code-cell}
get_y(trajectories[:1])[:5]
```

Then we can use empirical risk minimization to find a function $\hat f$ that approximates the optimal Q-function.

```{code-cell}
# We will see some examples of fitting methods in the next section
FittingMethod = Callable[[Float[Array, "N D"], Float[Array, " N"]], QFunction]
```

But notice that the definition of $y_{i \hi}$ depends on the Q-function itself!
How can we resolve this circular dependency?
Recall that we faced the same issue [when evaluating a policy in an infinite-horizon MDP](#iterative_pe). There, we iterated the [](#bellman_operator) since we knew that the policy's value function was a fixed point of the policy's Bellman operator.
We can apply the same strategy here, using the $\hat f$ from the previous iteration to compute the labels $y_{i \hi}$,
and then using this new dataset to fit the next iterate.

:::{prf:definition} Fitted Q-function iteration
:label: fitted_q_iteration

1. Initialize some function $\hat f(s, a, h) \in \mathbb{R}$.
2. Iterate the following:
   1. Generate a supervised learning dataset $X, y$ from the trajectories and the current estimate $f$, where the labels come from the r.h.s. of the Bellman optimality operator {eq}`bellman_optimality_operator`
   2. Set $\hat f$ to the function that minimizes the empirical risk:
   
      $$\hat f \gets \arg\min_f \frac{1}{N} \sum_{i=1}^N (y_i - f(x_i))^2.$$
:::

```{code-cell}
def fitted_q_iteration(
    trajectories: list[Trajectory],
    fit: FittingMethod,
    epochs: int,
    Q_init: Optional[QFunction] = None,
) -> QFunction:
    """
    Run fitted Q-function iteration using the given dataset.
    Returns an estimate of the optimal Q-function.
    """
    Q_hat = Q_init or Q_zero(get_num_actions(trajectories))
    X = get_X(trajectories)
    for _ in range(epochs):
        y = get_y(trajectories, Q_hat)
        Q_hat = fit(X, y)
    return Q_hat
```

We can also use this fixed-point interation to *evaluate* a policy using the dataset (not necessarily the one used to generate the trajectories):

:::{prf:definition} Fitted policy evaluation
:label: fitted_evaluation

**Input:** Policy $\pi : \mathcal{S} \times [H] \to \Delta(\mathcal{A})$ to be evaluated.

**Output:** An approximation of the value function $Q^\pi$ of the policy.

1. Initialize some function $\hat f(s, a, h) \in \mathbb{R}$.
2. Iterate the following:
   1. Generate a supervised learning dataset $X, y$ from the trajectories and the current estimate $f$, where the labels come from the r.h.s. of the {prf:ref}`Bellman consistency equation <bellman_consistency>` for the given policy.
   2. Set $\hat f$ to the function that minimizes the empirical risk:
   
      $$\hat f \gets \arg\min_f \frac{1}{N} \sum_{i=1}^N (y_i - f(x_i))^2.$$
:::

```{code-cell}
def fitted_evaluation(
    trajectories: list[Trajectory],
    fit: FittingMethod,
    π: Policy,
    epochs: int,
    Q_init: Optional[QFunction] = None,
) -> QFunction:
    """
    Run fitted policy evaluation using the given dataset.
    Returns an estimate of the Q-function of the given policy.
    """
    Q_hat = Q_init or Q_zero(get_num_actions(trajectories))
    X = get_X(trajectories)
    for _ in tqdm(range(epochs)):
        y = get_y(trajectories, Q_hat, π)
        Q_hat = fit(X, y)
    return Q_hat
```

:::{attention}
Spot the difference between `fitted_evaluation` and `fitted_q_iteration`. (See the definition of `get_y`.)
How would you modify this algorithm to evaluate the data collection policy?
:::

We can use this policy evaluation algorithm to adapt the [](#policy iteration algorithm <policy_iteration>) to this new setting. The algorithm remains exactly the same -- repeatedly make the policy greedy w.r.t. its own value function -- except now we must evaluate the policy (i.e. compute its value function) using the iterative `fitted_evaluation` algorithm.

```{code-cell}
def fitted_policy_iteration(
    trajectories: list[Trajectory],
    fit: FittingMethod,
    epochs: int,
    evaluation_epochs: int,
    π_init: Optional[Policy] = lambda s, h: 0,  # constant zero policy
):
    """Run fitted policy iteration using the given dataset."""
    π = π_init
    for _ in range(epochs):
        Q_hat = fitted_evaluation(trajectories, fit, π, evaluation_epochs)
        π = q_to_greedy(Q_hat)
    return π
```

## Summary


