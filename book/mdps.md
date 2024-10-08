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
  enumerator: 1.%s
---

# 1 Markov Decision Processes

## Introduction

The field of RL studies how an agent can learn to make sequential decisions in an interactive environment.
This is a very general problem!
How can we *formalize* this task in a way that is both *sufficiently general* yet also tractable enough for *fruitful analysis*?

Let’s consider some examples of sequential decision problems to identify the key common properties we’d like to capture:

- **Board games and video games,** where a player takes actions in a virtual environment.
- **Inventory management,** where a company must efficiently move resources from producers to consumers.
- **Robotic control**, where a robot can move and interact with the real world to complete some task.

In these environments and many others, the **state transitions**,
the “rules” of the environment,
only depend on the *most recent* state and action (generally speaking).
For example, if you want to take a break while playing a game of chess,
you could take a picture of the board,
and later on reset the board to that state and continue playing;
the past history of moves doesn't matter (generally speaking).
This is called the **Markov property.**

:::{prf:definition} Markov property
:label: markov

An interactive environment satisfies the **Markov property** if the
probability of transitioning to a new state only depends on the current
state and action:

$$\pr(s_{\hi+1} \mid s_0, a_0, \dots, s_\hi, a_\hi) = P(s_{\hi+1} \mid s_\hi, a_\hi)$$

where $P : \mathcal{S} \times \mathcal{A} \to \triangle(\mathcal{S})$ describes the state transitions.
(We’ll elaborate on this notation later in the chapter.)
:::

Environments that satisfy the Markov property are called **Markov decision processes** (MDPs).
This chapter will focus on introducing core vocabulary for MDPs that will be useful throughout the book.

:::{attention}
What information might be encoded in the _state_ for each of the above examples?
What might the valid set of _actions_ be?
Describe the _state transitions_ heuristically and verify that they satisfy the Markov property.
:::

MDPs are usually classified as **finite-horizon**, where the interactions end after some finite number of time steps,
or **infinite-horizon**, where the interactions can continue indefinitely.
We’ll begin with the finite-horizon case and discuss the infinite-horizon case in the second half of the chapter.

We’ll describe how to _evaluate_ different strategies, called **policies,** and how to compute (or approximate)
the **optimal policy** for a given MDP.
We’ll introduce the **Bellman consistency condition**, which allows us to analyze the whole sequence of interactions in terms of individual timesteps.

```{code-cell} ipython3
from utils import NamedTuple, Float, Array, partial, jax, jnp, latexify
```

## Finite-horizon MDPs

### Definition

::::{prf:definition} Finite-horizon Markov decision process
:label: finite_horizon_mdp

The components of a finite-horizon Markov decision process are:

1.  The **state** that the agent interacts with. We use $\mathcal{S}$ to denote
    the set of possible states, called the **state space**.

2.  The **actions** that the agent can take. We use $\mathcal{A}$ to denote the
    set of possible actions, called the **action space**.

3.  Some **initial state distribution** $\mu \in \triangle(\mathcal{S})$.

4.  The **state transitions** (a.k.a. **dynamics**)
    $P : \mathcal{S} \times \mathcal{A} \to \triangle(\mathcal{S})$ that describe what state the agent
    transitions to after taking an action.

5.  The **reward** signal. In this course we'll take it to be a
    deterministic function on state-action pairs,
    $r : \mathcal{S} \times \mathcal{A} \to \mathbb{R}$, but in general many results will
    extend to a *stochastic* reward signal.

6.  A time horizon $\hor \in \mathbb{N}$ that specifies the number of
    interactions in an **episode**.

Combined together, these objects specify a finite-horizon Markov
decision process:

$$M = (\mathcal{S}, \mathcal{A}, \mu, P, r, \hor).$$

When there are **finitely** many states and actions, i.e.
$|\mathcal{S}|, |\mathcal{A}| < \infty$, we can express
the relevant quantities as vectors and matrices (i.e. *tables* of
values):

$$
\begin{aligned}
    \mu &\in [0, 1]^{|\mathcal{S}|} &
    P &\in [0, 1]^{(|\mathcal{S} \times \mathcal{A}|) \times |\mathcal{S}|} &
    r &\in \mathbb{R}^{|\mathcal{S}| \times |\mathcal{A}|}
\end{aligned}
$$
::::

:::{attention}
Verify that the types and shapes provided above make sense!
:::

```{code-cell} ipython3
class MDP(NamedTuple):
    """A description of a Markov decision process with finitely many states and actions."""
    S: int  # number of states
    A: int  # number of actions
    μ: Float[Array, " S"]
    P: Float[Array, "S A S"]  # "current" state, "current" action, "next" state
    r: Float[Array, "S A"]
    H: int
    γ: float = 1.0  # discount factor (used later)
```

:::{prf:example} Tidying MDP
:label: tidy_mdp

Let's consider a simple decision problem throughout this chapter:
the task of keeping your room tidy!

Your room has the possible states
$\mathcal{S} = \{ \text{orderly}, \text{messy} \}.$
You can take either of the actions $\mathcal{A} = \{ \text{ignore}, \text{tidy} \}.$
The room starts off orderly.

The **state transitions** are as follows:
if you tidy the room, it becomes (or remains) orderly;
if you ignore the room, it _might_ become messy (see table below).

The **rewards** are as follows: You get penalized for tidying an orderly room (a waste of time) or ignoring a messy room,
but you get rewarded for ignoring an orderly room (since you can enjoy your additional time).
Tidying a messy room is a chore that gives no reward.

These are summarized in the following table:

$$\begin{array}{ccccc}
    s & a & P(\text{orderly} \mid s, a) & P(\text{messy} \mid s, a) & r(s, a) \\
    \text{orderly} & \text{ignore} & 0.7 & 0.3 & 1 \\
    \text{orderly} & \text{tidy} & 1 & 0 & -1 \\
    \text{messy} & \text{ignore} & 0 & 1 & -1 \\
    \text{messy} & \text{tidy} & 1 & 0 & 0 \\
\end{array}$$

Consider a time horizon of $\hor = 7$ days (one interaction per day). Let
$t = 0$ correspond to Monday and $t = 6$ correspond to Sunday.
:::

```{code-cell} ipython3
tidy_mdp = MDP(
    S=2,  # 0 = orderly, 1 = messy
    A=2,  # 0 = ignore, 1 = tidy
    μ=jnp.array([1.0, 0.0]),  # start in orderly state
    P=jnp.array([
        [
            [0.7, 0.3],  # orderly, ignore
            [1.0, 0.0],  # orderly, tidy
        ],
        [
            [0.0, 1.0],  # messy, ignore
            [1.0, 0.0],  # messy, tidy
        ],
    ]),
    r=jnp.array([
        [
            1.0,   # orderly, ignore
            -1.0,  # orderly, tidy
        ],
        [
            -1.0,  # messy, ignore
            0.0,   # messy, tidy
        ]
    ]),
    H=7,
)
```

### Policies

::::{prf:definition} Policies
:label: policy

A **policy** $\pi$ describes the agent's strategy:
which actions it takes in a given situation.
A key goal of RL is to find the **optimal policy** that maximizes the total reward on average.

There are three axes along which policies can vary: their outputs,
inputs, and time-dependence.

1.  **Deterministic or stochastic.** A deterministic policy outputs
    actions while a stochastic policy outputs *distributions* over
    actions.

:::{figure} ./shared/deterministic_policy.png
:align: center

A deterministic policy.
:::

:::{figure} ./shared/stochastic_policy.png
:align: center

A stochastic policy.
:::

2.  **State-dependent or history-dependent.** A state-dependent (a.k.a.
    "Markovian") policy only depends on the current state, while a
    history-dependent policy depends on the sequence of past states,
    actions, and rewards. We'll only consider state-dependent policies
    in this course.

3.  **Stationary or time-dependent.** A stationary (a.k.a. time-homogeneous) policy
    remains the same function at all time steps, while a time-dependent policy can depend on the current timestep.
    For consistency with states and actions, we will denote the timestep as a subscript,
    i.e. $\pi = \{ \pi_0, \dots, \pi_{\hor-1} \}.$
::::

+++

Note that for finite state and action spaces,
we can represent a randomized mapping $\mathcal{S} \to \Delta(\mathcal{A})$
as a matrix $\pi \in [0, 1]^{\mathcal{S} \times \mathcal{A}}$ where each row describes
the policy's distribution over actions for the corresponding state.

A fascinating result is that every finite-horizon MDP has an optimal deterministic time-dependent policy!
Intuitively, the Markov property implies that the current state contains all the information we need to make the optimal decision.
We'll prove this result constructively later in the chapter.

:::{prf:example} Policies for the tidying MDP
:label: tidy_policy

Here are some possible policies for the tidying MDP {prf:ref}`tidy_mdp`:

-   Always tidy: $\pi(s) = \text{tidy}$.

-   Only tidy on weekends: $\pi_\hi(s) = \text{tidy}$ if
    $\hi \in \{ 5, 6 \}$ and $\pi_\hi(s) = \text{ignore}$ otherwise.

-   Only tidy if the room is messy: $\pi_\hi(\text{messy}) = \text{tidy}$
    and $\pi_\hi(\text{orderly}) = \text{ignore}$ for all $\hi$.
:::

```{code-cell} ipython3
# arrays of shape (H, S, A) represent time-dependent policies
tidy_policy_always_tidy = (
    jnp.zeros((7, 2, 2))
    .at[:, :, 1].set(1.0)
)
tidy_policy_weekends = (
    jnp.zeros((7, 2, 2))
    .at[5:7, :, 1].set(1.0)
    .at[0:5, :, 0].set(1.0)
)
tidy_policy_messy_only = (
    jnp.zeros((7, 2, 2))
    .at[:, 1, 1].set(1.0)
    .at[:, 0, 0].set(1.0)
)
```

:::{note}
Array objects in Jax are **immutable,** that is, they cannot be _changed._
This might seem inconvenient, but in larger projects,
immutability makes code much easier to reason about.
:::

+++

(trajectories)=
### Trajectories

:::{prf:definition} Trajectories
:label: trajectory

A sequence of states, actions, and rewards is called a **trajectory**:

$$\tau = (s_0, a_0, r_0, \dots, s_{H-1}, a_{H-1}, r_{H-1})$$

where $r_\hi = r(s_\hi, a_\hi)$.
(Note that some sources omit the reward at the final time step. This is a minor detail.)
:::

```{code-cell} ipython3
class Transition(NamedTuple):
    """A single state-action-reward interaction with the environment.

    A trajectory comprises a sequence of transitions.
    """
    s: int
    a: int
    r: float
```

Once we've chosen a policy,
we can sample trajectories by repeatedly choosing actions according to the policy,
transitioning according to the state transitions, and observing the rewards.

:::{image} shared/trajectory.png
:width: 240px
:align: center
:::

That is, a policy induces a distribution $\rho^{\pi}$ over trajectories.
(We assume that $\mu$ and $P$ are clear from context.)

::::{prf:example} Trajectories in the tidying environment
:label: tidy_traj

Here is a possible trajectory for the tidying example:

:::{table}
|  $\hi$  |    $0$    |    $1$    |    $2$    |   $3$    |   $4$   |    $5$    |    $6$    |
| :-----: | :-------: | :-------: | :-------: | :------: | :-----: | :-------: | :-------: |
|  $s$    |  orderly  |  orderly  |  orderly  |  messy   |  messy  |  orderly  |  orderly  |
|  $a$    |   tidy    |  ignore   |  ignore   |  ignore  |  tidy   |  ignore   |  ignore   |
|  $r$    |   $-1$    |    $1$    |    $1$    |   $-1$   |   $0$   |    $1$    |    $1$    |
:::

Could any of the policies in {prf:ref}`tidy_policy` have generated this trajectory?
::::

Note that for a state-dependent policy, using the Markov property {prf:ref}`markov`,
we can write down the likelihood function of this probability distribution in an **autoregressive** way (i.e. one timestep at a time):

:::{prf:definition} Autoregressive trajectory distribution
:label: autoregressive_trajectories

$$\rho^{\pi}(\tau) := \mu(s_0) \pi_0(a_0 \mid s_0) P(s_1 \mid s_0, a_0) \cdots P(s_{\hor-1} \mid s_{\hor-2}, a_{\hor-2}) \pi_{\hor-1}(a_{\hor-1} \mid s_{\hor-1})$$
:::

```{code-cell} ipython3
def trajectory_log_likelihood(
    mdp: MDP,
    τ: list[Transition],
    π: Float[Array, "S A"],
) -> float:
    """Compute the log-likelihood of a trajectory under a given MDP and policy."""

    # initial distribution and action
    total = jnp.log(mdp.μ[τ[0].s])
    total += jnp.log(π[τ[0].s, τ[0].a])

    # remaining state transitions and actions
    for i in range(1, mdp.H):
        total += jnp.log(mdp.P[τ[i - 1].s, τ[i - 1].a, τ[i].s])
        total += jnp.log(π[τ[i].s, τ[i].a])

    return total
```

:::{attention}
How would you modify this to include stochastic rewards?
:::

For a deterministic policy $\pi$, we have that $\pi_\hi(a \mid s) = \mathbb{I}[a = \pi_\hi(s)]$;
that is, the probability of taking an action is $1$ if it's the unique action prescribed by the policy for that state and $0$ otherwise.
In this case, the only randomness in sampling trajectories comes from the initial state distribution $\mu$ and the state transitions $P$.

+++

### Value functions

The main goal of RL is to find a policy that maximizes the expected total
reward $\E [r_0 + \cdots + r_{\hor-1}]$.

:::{attention}
Note that $r_0 + \cdots + r_{\hor-1}$ is a random variable.
What sources of randomness does it depend on?
Describe the generating process.
:::

Let's introduce some notation for analyzing this quantity.

A policy's **value function** at time $\hi$ is its expected remaining reward *from a given state*:

:::{prf:definition} Value function
:label: value

$$V_\hi^\pi(s) := \E_{\tau \sim \rho^\pi} [r_\hi + \cdots + r_{H-1} \mid s_\hi = s]$$
:::

Similarly, we can define the **action-value function** (aka the
**Q-function**) at time $h$ as the expected remaining reward *from a given state and taking a given action*:

:::{prf:definition} Action-value function
:label: action_value

$$Q_\hi^\pi(s, a) := \E_{\tau \sim \rho^\pi} [r_\hi + \cdots + r_{H-1} \mid s_\hi = s, a_\hi = a]$$
:::

+++

#### Relating the value function and action-value function

Note that the value function is just the expected action-value over
actions drawn from the policy:

$$V_\hi^\pi(s) = \E_{a \sim \pi_\hi(s)} [Q_\hi^\pi(s, a)]$$

```{code-cell} ipython3
def q_to_v(
    policy: Float[Array, "S A"],
    q: Float[Array, "S A"],
) -> Float[Array, " S"]:
    """
    Compute the value function for a given policy in a known finite MDP
    at a single timestep from its action-value function.
    """
    return jnp.average(q, weights=policy, axis=1)
```

and the action-value is the sum of the immediate reward and the expected value of the following
state:

$$Q_\hi^\pi(s, a) = r(s, a) + \E_{s' \sim P(s, a)} [V_{\hi+1}^\pi(s')]$$

```{code-cell} ipython3
def v_to_q(
    mdp: MDP,
    v_next: Float[Array, " S"],
) -> Float[Array, "S A"]:
    """
    Compute the action-value function in a known finite MDP
    at a single timestep from the corresponding value function.
    """
    # the discount factor is relevant later
    return mdp.r + mdp.γ * mdp.P @ v_next


# convert a list of v functions to a list of q functions
v_ary_to_q_ary = jax.vmap(v_to_q, in_axes=(None, 0))
```

#### Greedy policies

For any given $Q \in \mathbb{R}^{|\mathcal{S}| \times |\mathcal{A}|}$, we can define the **greedy policy** $\hat \pi_Q$ as the deterministic policy that selects the action with the highest $Q$-value at each state:

$$
\hat \pi_Q(s) = \arg\max_{a} Q_{sa}
$$

```{code-cell} ipython3
def q_to_greedy(q: Float[Array, "S A"]) -> Float[Array, "S A"]:
    """
    Get the (deterministic) greedy policy w.r.t. an action-value function.
    Return the policy as a matrix of shape (S, A) where each row is a one-hot vector.
    """
    A = q.shape[1]
    a_ary = jnp.argmax(q, axis=1)
    return jnp.eye(A)[a_ary]


def v_to_greedy(mdp: MDP, v: Float[Array, " S"]) -> Float[Array, "S A"]:
    """Get the (deterministic) greedy policy w.r.t. a value function."""
    return q_to_greedy(v_to_q(mdp, v))
```

### The one-step (Bellman) consistency equation

Note that by simply considering the cumulative reward as the sum of the
*current* reward and the *future* cumulative reward, we can describe the
value function recursively (in terms of itself). This is named the
**Bellman consistency equation** after **Richard Bellman** (1920--1984),
who is credited with introducing dynamic programming in 1953.

:::{prf:theorem} Bellman consistency equation for the value function
:label: bellman_consistency

$$
V_\hi^\pi(s) = \E_{\substack{a \sim \pi_\hi(s) \\ s' \sim P(s, a)}} [r(s, a) + V_{\hi+1}^\pi(s')]
$$
:::

```{code-cell} ipython3
def check_bellman_consistency_v(
    mdp: MDP,
    policy: Float[Array, "H S A"],
    v_ary: Float[Array, "H S"],
) -> bool:
    """
    Check that the given (time-dependent) "value function"
    satisfies the Bellman consistency equation.
    """
    return all(
        jnp.allclose(
            # lhs
            v_ary[h],
            # rhs
            jnp.sum(policy[h] * (mdp.r + mdp.γ * mdp.P @ v_ary[h + 1]), axis=1),
        )
        for h in range(mdp.H - 1)
    )
```

:::{attention}
Verify that this equation holds by expanding $V_\hi^\pi(s)$
and $V_{\hi+1}^\pi(s')$.
:::

One can analogously derive the Bellman consistency equation for the
action-value function:

:::{prf:theorem} Bellman consistency equation for action-values
:label: bellman_consistency_action

$$Q_\hi^\pi(s, a) = r(s, a) + \E_{\substack{s' \sim P(s, a) \\ a' \sim \pi_{\hi+1}(s')}} [Q_{\hi+1}^\pi(s', a')]$$
:::

:::{attention}
Write a `check_bellman_consistency_q` function for the action-value function.
:::

:::{prf:remark} The Bellman consistency equation for deterministic policies
:label: bellman_det

Note that for deterministic policies, the Bellman consistency equation
simplifies to

$$
\begin{aligned}
    V_\hi^\pi(s) &= r(s, \pi_\hi(s)) + \E_{s' \sim P(s, \pi_\hi(s))} [V_{\hi+1}^\pi(s')] \\
    Q_\hi^\pi(s, a) &= r(s, a) + \E_{s' \sim P(s, a)} [Q_{\hi+1}^\pi(s', \pi_{\hi+1}(s'))]
\end{aligned}
$$
:::

+++

### The one-step Bellman operator

Fix a policy $\pi$. Consider the higher-order operator that takes in a
"value function" $v : \mathcal{S} \to \mathbb{R}$ and returns the r.h.s. of the Bellman
equation for that "value function":

:::{prf:definition} Bellman operator
:label: bellman_operator

$$[\mathcal{J}^{\pi}(v)](s) := \E_{\substack{a \sim \pi(s) \\ s' \sim P(s, a)}} [r(s, a) + v(s')].$$

This is a crucial tool for reasoning about MDPs.
Intuitively, it answers the following question:
if we evaluate the _next_ state using $v$,
how good is the _current_ state, according to the given policy?
:::

```{code-cell} ipython3
:tags: [hide-input]

def bellman_operator_looping(
    mdp: MDP,
    policy: Float[Array, "S A"],
    v: Float[Array, " S"],
) -> Float[Array, " S"]:
    """
    Looping definition of the Bellman operator.
    Concise version is below
    """
    v_new = jnp.zeros(mdp.S)
    for s in range(mdp.S):
        for a in range(mdp.A):
            for s_next in range(mdp.S):
                v_new[s] += (
                    policy[s, a]
                    * mdp.P[s, a, s_next]
                    * (mdp.r[s, a] + mdp.γ * v[s_next])
                )
    return v_new
```

Note that we can concisely implement this using the `q_to_v` and `v_to_q` utilities from above:

```{code-cell} ipython3
def bellman_operator(
    mdp: MDP,
    policy: Float[Array, "S A"],
    v: Float[Array, " S"],
) -> Float[Array, " S"]:
    """For a known finite MDP, the Bellman operator can be exactly evaluated."""
    return q_to_v(policy, v_to_q(mdp, v))  # equivalent
    return jnp.sum(policy * (mdp.r + mdp.γ * mdp.P @ v), axis=1)
```

We'll call $\mathcal{J}^\pi : \mathbb{R}^\mathcal{S} \to \mathbb{R}^\mathcal{S}$ the **Bellman
operator** of $\pi$.
Note that it's defined on any "value function" mapping states to real numbers;
$v$ doesn't have to be a well-defined value function for some policy (hence the lowercase notation).
The Bellman operator also gives us a concise way to express {prf:ref}`bellman_consistency` for the value function:

$$V_\hi^\pi = \mathcal{J}^{\pi}(V_{\hi+1}^\pi)$$

Intuitively, the output of the Bellman operator, a new "value function",
evaluates states as follows: from a given state, take one action
according to $\pi$, observe the reward, and then evaluate the next state
using the input "value function".

When we discuss infinite-horizon MDPs, the Bellman operator will turn
out to be more than just a notational convenience: We'll use it to
construct algorithms for computing the optimal policy.


(finite_horizon_mdps)=
## Solving finite-horizon MDPs

(eval_dp)=
### Policy evaluation in finite-horizon MDPs

How can we actually compute the value function of a given policy? This
is the task of **policy evaluation**.

:::{prf:definition} DP algorithm to evaluate a policy in a finite-horizon MDP

The Bellman consistency equation
{prf:ref}`bellman_consistency`
gives us a convenient algorithm for
evaluating stationary policies: it expresses the value function at
timestep $\hi$ as a function of the value function at timestep $\hi+1$. This
means we can start at the end of the time horizon, where the value is
known, and work backwards in time, using the Bellman consistency
equation to compute the value function at each time step.
:::

```{code-cell} ipython3
def dp_eval_finite(mdp: MDP, policy: Float[Array, "S A"]) -> Float[Array, "H S"]:
    """Evaluate a policy using dynamic programming."""
    V_ary = [None] * mdp.H + [jnp.zeros(mdp.S)]  # initialize to 0 at end of time horizon
    for h in range(mdp.H - 1, -1, -1):
        V_ary[h] = bellman_operator(mdp, policy[h], V_ary[h + 1])
    return jnp.stack(V_ary[:-1])
```

This runs in time $O(H \cdot |\mathcal{S}|^2 \cdot |\mathcal{A}|)$ by counting the
loops.

:::{attention}
Do you see where we compute $Q^\pi_\hi$ along the way? Make
this step explicit.
:::

:::{prf:example} Tidying policy evaluation
:label: tidy_eval_finite

Let's evaluate the policy from
{prf:ref}`tidy_policy` in the tidying MDP
that tidies if and only if the room is
messy. We'll use the Bellman consistency equation to compute the value
function at each time step.

$$
\begin{aligned}
V_{H-1}^\pi(\text{orderly}) &= r(\text{orderly}, \text{ignore}) \\
&= 1 \\
V_{H-1}^\pi(\text{messy}) &= r(\text{messy}, \text{tidy}) \\
&= 0 \\
V_{H-2}^\pi(\text{orderly}) &= r(\text{orderly}, \text{ignore}) + \E_{s' \sim P(\text{orderly}, \text{ignore})} [V_{H-1}^\pi(s')] \\
&= 1 + 0.7 \cdot V_{H-1}^{\pi}(\text{orderly}) + 0.3 \cdot V_{H-1}^{\pi}(\text{messy}) \\
&= 1 + 0.7 \cdot 1 + 0.3 \cdot 0 \\
&= 1.7 \\
V_{H-2}^\pi(\text{messy}) &= r(\text{messy}, \text{tidy}) + \E_{s' \sim P(\text{messy}, \text{tidy})} [V_{H-1}^\pi(s')] \\
&= 0 + 1 \cdot V_{H-1}^{\pi}(\text{orderly}) + 0 \cdot V_{H-1}^{\pi}(\text{messy}) \\
&= 1 \\
V_{H-3}^\pi(\text{orderly}) &= r(\text{orderly}, \text{ignore}) + \E_{s' \sim P(\text{orderly}, \text{ignore})} [V_{H-2}^\pi(s')] \\
&= 1 + 0.7 \cdot V_{H-2}^{\pi}(\text{orderly}) + 0.3 \cdot V_{H-2}^{\pi}(\text{messy}) \\
&= 1 + 0.7 \cdot 1.7 + 0.3 \cdot 1 \\
&= 2.49 \\
V_{H-3}^\pi(\text{messy}) &= r(\text{messy}, \text{tidy}) + \E_{s' \sim P(\text{messy}, \text{tidy})} [V_{H-2}^\pi(s')] \\
&= 0 + 1 \cdot V_{H-2}^{\pi}(\text{orderly}) + 0 \cdot V_{H-2}^{\pi}(\text{messy}) \\
&= 1.7
\end{aligned}
$$

etc. You may wish to repeat this computation for the
other policies to get a better sense of this algorithm.
:::

```{code-cell} ipython3
V_messy = dp_eval_finite(tidy_mdp, tidy_policy_messy_only)
V_messy
```

(opt_dynamic_programming)=
### Optimal policies in finite-horizon MDPs

We've just seen how to *evaluate* a given policy. But how can we find
the **optimal policy** for a given environment?

:::{prf:definition} Optimal policies
:label: optimal_policy_finite

We call a policy optimal, and denote it by $\pi^\star$, if it does at
least as well as *any* other policy $\pi$ (including stochastic and
history-dependent ones) in all situations:

$$
\begin{aligned}
    V_\hi^{\pi^\star}(s) &= \E_{\tau \sim \rho^{\pi^{\star}}}[r_\hi + \cdots + r_{H-1} \mid s_\hi = s] \\
    &\ge \E_{\tau \sim \rho^{\pi}}[r_\hi + \cdots + r_{H-1} \mid \tau_\hi] \quad \forall \pi, \tau_\hi, \hi \in [H]
\end{aligned}
$$

where we condition on the
trajectory up to time $\hi$, denoted
$\tau_\hi = (s_0, a_0, r_0, \dots, s_\hi)$, where $s_\hi = s$.
:::

Convince yourself that all optimal policies must have the same value
function. We call this the **optimal value function** and denote it by
$V_\hi^\star(s)$. The same goes for the action-value function
$Q_\hi^\star(s, a)$.

It is a stunning fact that **every finite-horizon MDP has an optimal
policy that is time-dependent and deterministic.** In particular, we can
construct such a policy by acting *greedily* with respect to the optimal
action-value function:


:::{prf:theorem} It is optimal to be greedy w.r.t. the optimal value function
:label: optimal_greedy

$$\pi_\hi^\star(s) = \arg\max_a Q_\hi^\star(s, a).$$
:::

::::{prf:proof} Proof
Let $V^{\star}$ and $Q^{\star}$ denote the optimal value and
action-value functions. Consider the greedy policy

$$\hat \pi_\hi(s) := \arg\max_a Q_\hi^{\star}(s, a).$$

We aim to show that
$\hat \pi$ is optimal; that is, $V^{\hat \pi} = V^{\star}$.

Fix an arbitrary state $s \in \mathcal{S}$ and time $\hi \in [H]$.

Firstly, by the definition of $V^{\star}$, we already know
$V_\hi^{\star}(s) \ge V_\hi^{\hat \pi}(s)$. So for equality to hold we just
need to show that $V_\hi^{\star}(s) \le V_\hi^{\hat \pi}(s)$. We'll first
show that the Bellman operator $\mathcal{J}^{\hat \pi}$ never decreases
$V_\hi^{\star}$. Then we'll apply this result recursively to show that
$V^{\star} = V^{\hat \pi}$.

:::{prf:lemma} The Bellman operator never decreases the optimal value function
$\mathcal{J}^{\hat \pi}$ never decreases $V_\hi^{\star}$
(elementwise):

$$[\mathcal{J}^{\hat \pi} (V_{\hi+1}^{\star})](s) \ge V_\hi^{\star}(s).$$

**Proof:**

$$
\begin{aligned}
    V_\hi^{\star}(s) &= \max_{\pi \in \Pi} V_\hi^{\pi}(s) \\
    &= \max_{\pi \in \Pi} \mathop{\mathbb{E}}_{a \sim \pi(\dots)}\left[r(s, a) + \mathop{\mathbb{E}}_{s' \sim P(s, a)} V_{\hi+1}^\pi(s') \right] && \text{Bellman consistency} \\
    &\le \max_{\pi \in \Pi} \mathop{\mathbb{E}}_{a \sim \pi(\dots)}\left[r(s, a) + \mathop{\mathbb{E}}_{s' \sim P(s, a)} V_{\hi+1}^{\star}(s') \right] && \text{definition of } V^\star \\
    &= \max_{a} \left[ r(s, a) + \mathop{\mathbb{E}}_{s' \sim P(s, a)} V_{\hi+1}^{\star}(s') \right] && \text{only depends on } \pi \text{ via } a \\
    &= [\mathcal{J}^{\hat \pi}(V_{\hi+1}^{\star})](s).    
\end{aligned}
$$

Note that the chosen action $a \sim \pi(\dots)$ above
might depend on the past history; this isn't shown in the notation and
doesn't affect our result (make sure you see why).
:::

We can now apply this result recursively to get

$$V^{\star}_t(s) \le V^{\hat \pi}_t(s)$$

as follows. (Note that even
though $\hat \pi$ is deterministic, we'll use the $a \sim \hat \pi(s)$
notation to make it explicit that we're sampling a trajectory from it.)

$$
\begin{aligned}
    V_{t}^{\star}(s) &\le [\mathcal{J}^{\hat \pi}(V_{\hi+1}^{\star})](s) \\
    &= \mathop{\mathbb{E}}_{a \sim \hat \pi(s)} \left[ r(s, a) + \mathop{\mathbb{E}}_{s' \sim P(s, a)} \left[ {\color{blue} V_{\hi+1}^{\star}(s')} \right] \right] && \text{definition of } \mathcal{J}^{\hat \pi} \\
    &\le \mathop{\mathbb{E}}_{a \sim \hat \pi(s)} \left[ r(s, a) + \mathop{\mathbb{E}}_{s' \sim P(s, a)} \left[ {\color{blue}[ \mathcal{J}^{\hat \pi} (V_{t+2}^{\star})] (s')} \right] \right] && \text{above lemma} \\
    &= \mathop{\mathbb{E}}_{a \sim \hat \pi(s)} \left[ r(s, a) + \mathop{\mathbb{E}}_{s' \sim P(s, a)}{\color{blue} \left[ \mathop{\mathbb{E}}_{a' \sim \hat \pi}  r(s', a') + \mathop{\mathbb{E}}_{s''} V_{t+2}^{\star}(s'') \right]} \right] && \text{definition of } \mathcal{J}^{\hat \pi} \\
    &\le \cdots && \text{apply at all timesteps} \\
    &= \mathop{\mathbb{E}}_{\tau \sim \rho^{\hat \pi}} [G_{t} \mid s_\hi = s] && \text{rewrite expectation} \\
    &= V_{t}^{\hat \pi}(s) && \text{definition}
\end{aligned}
$$

And so we have $V^{\star} = V^{\hat \pi}$, making $\hat \pi$ optimal.
::::

Note that this also gives simplified forms of the [Bellman consistency](#bellman_consistency) equations for the optimal policy:

::::{prf:corollary} Bellman consistency equations for the optimal policy
:label: bellman_consistency_optimal

$$
\begin{aligned}
    V_\hi^\star(s) &= \max_a Q_\hi^\star(s, a) \\
    Q_\hi^\star(s, a) &= r(s, a) + \E_{s' \sim P(s, a)} [V_{\hi+1}^\star(s')]
\end{aligned}
$$
::::

Now that we've shown this particular greedy policy is optimal, all we
need to do is compute the optimal value function and optimal policy. We
can do this by working backwards in time using **dynamic programming**
(DP).

:::{prf:definition} DP algorithm to compute an optimal policy in a finite-horizon MDP
:label: pi_star_dp

**Base case.** At the end of the episode (time step $H-1$), we can't
take any more actions, so the $Q$-function is simply the reward that
we obtain:

$$Q^\star_{H-1}(s, a) = r(s, a)$$

so the best thing to do
is just act greedily and get as much reward as we can!

$$\pi^\star_{H-1}(s) = \arg\max_a Q^\star_{H-1}(s, a)$$

Then
$V^\star_{H-1}(s)$, the optimal value of state $s$ at the end of the
trajectory, is simply whatever action gives the most reward.

$$V^\star_{H-1} = \max_a Q^\star_{H-1}(s, a)$$

**Recursion.** Then, we can work backwards in time, starting from the
end, using our consistency equations! i.e. for each
$t = H-2, \dots, 0$, we set

$$
\begin{aligned}
    Q^\star_{t}(s, a) &= r(s, a) + \E_{s' \sim P(s, a)} [V^\star_{\hi+1}(s')] \\
    \pi^\star_{t}(s) &= \arg\max_a Q^\star_{t}(s, a) \\
    V^\star_{t}(s) &= \max_a Q^\star_{t}(s, a)
\end{aligned}
$$
:::

```{code-cell} ipython3
def find_optimal_policy(mdp: MDP):
    Q = [None] * mdp.H
    pi = [None] * mdp.H
    V = [None] * mdp.H + [jnp.zeros(mdp.S)]  # initialize to 0 at end of time horizon

    for h in range(mdp.H - 1, -1, -1):
        Q[h] = mdp.r + mdp.P @ V[h + 1]
        pi[h] = jnp.eye(mdp.S)[jnp.argmax(Q[h], axis=1)]  # one-hot
        V[h] = jnp.max(Q[h], axis=1)

    Q = jnp.stack(Q)
    pi = jnp.stack(pi)
    V = jnp.stack(V[:-1])

    return pi, V, Q
```

At each of the $H$ timesteps, we must compute $Q^{\star}$ for each of
the $|\mathcal{S}| |\mathcal{A}|$ state-action pairs. Each computation takes $|\mathcal{S}|$
operations to evaluate the average value over $s'$. This gives a total
computation time of $O(H \cdot |\mathcal{S}|^2 \cdot |\mathcal{A}|)$.

Note that this algorithm is identical to the policy evaluation algorithm
[`dp_eval_finite`](#eval_dp), but instead of *averaging* over the
actions chosen by a policy, we instead simply take a *maximum* over the
action-values. We'll see this relationship between **policy evaluation**
and **optimal policy computation** show up again in the infinite-horizon
setting.

```{code-cell} ipython3
π_opt, V_opt, Q_opt = find_optimal_policy(tidy_mdp)
assert jnp.allclose(π_opt, tidy_policy_messy_only)
assert jnp.allclose(V_opt, V_messy)
assert jnp.allclose(Q_opt[:-1], v_ary_to_q_ary(tidy_mdp, V_messy)[1:])
"Assertions passed (the 'tidy when messy' policy is optimal)"
```

(infinite_horizon_mdps)=
## Infinite-horizon MDPs

What happens if a trajectory is allowed to continue forever (i.e.
$H = \infty$)? This is the setting of **infinite horizon** MDPs.

In this chapter, we'll describe the necessary adjustments from the
finite-horizon case to make the problem tractable. We'll show that the
[Bellman operator](#bellman_operator) in the discounted reward setting is a
**contraction mapping** for any policy.
We'll discuss how to evaluate
policies (i.e. compute their corresponding value functions). Finally,
we'll present and analyze two iterative algorithms, based on the Bellman
operator, for computing the optimal policy: **value iteration** and
**policy iteration**.

### Discounted rewards

First of all, note that maximizing the cumulative reward
$r_\hi + r_{\hi+1} + r_{\hi+2} + \cdots$ is no longer a good idea since it
might blow up to infinity. Instead of a time horizon $H$, we now need a
**discount factor** $\gamma \in [0, 1)$ such that rewards become less
valuable the further into the future they are:

$$r_\hi + \gamma r_{\hi+1} + \gamma^2 r_{\hi+2} + \cdots = \sum_{k=0}^\infty \gamma^k r_{\hi+k}.$$

We can think of $\gamma$ as measuring how much we care about the future:
if it's close to $0$, we only care about the near-term rewards; it's
close to $1$, we put more weight into future rewards.

You can also analyze $\gamma$ as the probability of *continuing* the
trajectory at each time step. (This is equivalent to $H$ being
distributed by a First Success distribution with success probability
$\gamma$.) This accords with the above interpretation: if $\gamma$ is
close to $0$, the trajectory will likely be very short, while if
$\gamma$ is close to $1$, the trajectory will likely continue for a long
time.

:::{attention}
Assuming that $r_\hi \in [0, 1]$ for all $\hi \in \mathbb{N}$,
what is the maximum **discounted** cumulative reward? You may find it
useful to review geometric series.
:::

The other components of the MDP remain the same:

$$M = (\mathcal{S}, \mathcal{A}, \mu, P, r, \gamma).$$

Code-wise, we can reuse the `MDP` class from before {prf:ref}`finite_horizon_mdp` and set `mdp.H = float('inf')`.

```{code-cell} ipython3
tidy_mdp_inf = tidy_mdp._replace(H=float("inf"), γ=0.95)
```

### Stationary policies

The time-dependent policies from the finite-horizon case become
difficult to handle in the infinite-horizon case. In particular, many of
the DP approaches we saw required us to start at the end of the
trajectory, which is no longer possible. We'll shift to **stationary**
policies $\pi : \mathcal{S} \to \mathcal{A}$ (deterministic) or $\Delta(\mathcal{A})$ (stochastic).

:::{attention}
Which of the policies in {prf:ref}`tidy_policy` are stationary?
:::

### Value functions and Bellman consistency

We also consider stationary value functions $V^\pi : \mathcal{S} \to \mathbb{R}$ and
$Q^\pi : \mathcal{S} \times \mathcal{A} \to \mathbb{R}$. We need to insert a factor of $\gamma$
into the Bellman consistency equation {prf:ref}`bellman_consistency` to account for the discounting:

:::{math}
:label: bellman_consistency_infinite

\begin{aligned}
    V^\pi(s) &= \E_{\tau \sim \rho^\pi} [r_\hi + \gamma r_{\hi+1} + \gamma^2 r_{\hi+2} \cdots \mid s_\hi = s] && \text{for any } \hi \in \mathbb{N} \\
    &= \E_{\substack{a \sim \pi(s) \\ s' \sim P(s, a)}} [r(s, a) + \gamma V^\pi(s')]\\
    Q^\pi(s, a) &= \E_{\tau \sim \rho^\pi} [r_\hi + \gamma r_{\hi+1} + \gamma^2 r_{\hi+2} + \cdots \mid s_\hi = s, a_\hi = a] && \text{for any } \hi \in \mathbb{N} \\
    &= r(s, a) + \gamma \E_{\substack{s' \sim P(s, a) \\ a' \sim \pi(s')}} [Q^\pi(s', a')]
\end{aligned}
:::

:::{attention}
Heuristically speaking, why does it no longer matter which
time step we condition on when defining the value function?
:::

## Solving infinite-horizon MDPs

### The Bellman operator is a contraction mapping

Recall from [](#bellman_operator) that the Bellman operator $\mathcal{J}^{\pi}$
for a policy $\pi$ takes in a "value function" $v : \mathcal{S} \to \mathbb{R}$ and
returns the r.h.s. of the Bellman equation for that "value function". In
the infinite-horizon setting, this is

$$[\mathcal{J}^{\pi}(v)](s) := \E_{\substack{a \sim \pi(s) \\ s' \sim P(s, a)}} [r(s, a) + \gamma v(s')].$$

The crucial property of the Bellman operator is that it is a
**contraction mapping** for any policy. Intuitively, if we start with
two "value functions" $v, u : \mathcal{S} \to \mathbb{R}$, if we repeatedly apply the
Bellman operator to each of them, they will get closer and closer
together at an exponential rate.

:::{prf:definition} Contraction mapping
:label: contraction

Let $X$ be some space with a norm $\|\cdot\|$. We call an operator
$f: X \to X$ a **contraction mapping** if for any $x, y \in X$,

$$\|f(x) - f(y)\| \le \gamma \|x - y\|$$

for some fixed $\gamma \in (0, 1)$.
:::

:::{attention}
Show that for a contraction mapping $f$ with coefficient
$\gamma$, for all $t \in \mathbb{N}$,

$$\|f^{(t)}(x) - f^{(t)}(y)\| \le \gamma^t \|x - y\|,$$

i.e. that any
two points will be pushed closer by at least a factor of $\gamma$ at
each iteration.
:::

It is a powerful fact (known as the **Banach fixed-point theorem**) that
every contraction mapping has a unique **fixed point** $x^\star$ such
that $f(x^\star) = x^\star$. This means that if we repeatedly apply $f$
to any starting point, we will eventually converge to $x^\star$:

:::{math}
:label: contraction_convergence

\|f^{(t)}(x) - x^\star\| \le \gamma^t \|x - x^\star\|.
:::

Let's return to the RL setting and apply this result to the Bellman
operator. How can we measure the distance between two "value functions"
$v, u : \mathcal{S} \to \mathbb{R}$? We'll take the **supremum norm** as our distance
metric:

$$\| v - u \|_{\infty} := \sup_{s \in \mathcal{S}} |v(s) - u(s)|,$$

i.e.
we compare the "value functions" on the state that causes the biggest
gap between them. Then {eq}`contraction_convergence` implies that if we repeatedly
apply $\mathcal{J}^\pi$ to any starting "value function", we will eventually
converge to $V^\pi$:

:::{math}
:label: bellman_convergence

\|(\mathcal{J}^\pi)^{(t)}(v) - V^\pi \|_{\infty} \le \gamma^{t} \| v - V^\pi\|_{\infty}.
:::

We'll use this useful fact to prove the convergence of several
algorithms later on.

:::{prf:theorem} The Bellman operator is a contraction mapping
:label: bellman_contraction

$$
\|\mathcal{J}^{\pi} (v) - \mathcal{J}^{\pi} (u) \|_{\infty} \le \gamma \|v - u \|_{\infty}.
$$
:::

:::{prf:proof} Proof of {prf:ref}`bellman_contraction`

For all states $s \in \mathcal{S}$,

$$
\begin{aligned}
|[\mathcal{J}^{\pi} (v)](s) - [\mathcal{J}^{\pi} (u)](s)|&= \Big| \mathop{\mathbb{E}}_{a \sim \pi(s)} \left[ r(s, a) + \gamma \mathop{\mathbb{E}}_{s' \sim P(s, a)} v(s') \right] \\
&\qquad - \mathop{\mathbb{E}}_{a \sim \pi(s)} \left[r(s, a) + \gamma \mathop{\mathbb{E}}_{s' \sim P(s, a)} u(s') \right] \Big| \\
&= \gamma \left|\mathop{\mathbb{E}}_{s' \sim P(s, a)} [v(s') - u(s')] \right| \\
&\le \gamma \mathop{\mathbb{E}}_{s' \sim P(s, a)}|v(s') - u(s')| \qquad \text{(Jensen's inequality)} \\
&\le \gamma \max_{s'} |v(s') - u(s')| \\
&= \gamma \|v - u \|_{\infty}.
\end{aligned}
$$
:::

### Policy evaluation in infinite-horizon MDPs

The backwards DP technique we used in [the finite-horizon case](#eval_dp) no
longer works since there is no "final timestep" to start from. We'll
need another approach to policy evaluation.

The Bellman consistency conditions yield a system of equations we can
solve to evaluate a deterministic policy *exactly*. For a faster approximate solution,
we can iterate the policy's Bellman operator, since we know that it has
a unique fixed point at the true value function.

#### Matrix inversion for deterministic policies

Note that when the policy $\pi$ is deterministic, the actions can be
determined from the states, and so we can chop off the action dimension
for the rewards and state transitions:

$$
\begin{aligned}
    r^{\pi} &\in \mathbb{R}^{|\mathcal{S}|} & P^{\pi} &\in [0, 1]^{|\mathcal{S}| \times |\mathcal{S}|} & \mu &\in [0, 1]^{|\mathcal{S}|} \\
    \pi &\in \mathcal{A}^{|\mathcal{S}|} & V^\pi &\in \mathbb{R}^{|\mathcal{S}|} & Q^\pi &\in \mathbb{R}^{|\mathcal{S}| \times |\mathcal{A}|}.
\end{aligned}
$$

For $P^\pi$, we'll treat the rows as the states and the
columns as the next states. Then $P^\pi_{s, s'}$ is the probability of
transitioning from state $s$ to state $s'$ under policy $\pi$.

:::{prf:example} Tidying MDP
:label: tidy_tabular

The tabular MDP from before has $|\mathcal{S}| = 2$ and $|\mathcal{A}| = 2$. Let's write
down the quantities for the policy $\pi$ that tidies if and only if the
room is messy:

$$r^{\pi} = \begin{bmatrix} 1 \\ 0 \end{bmatrix}, \quad
        P^{\pi} = \begin{bmatrix} 0.7 & 0.3 \\ 1 & 0 \end{bmatrix}, \quad
        \mu = \begin{bmatrix} 1 \\ 0 \end{bmatrix}$$
        
We'll see how to
evaluate this policy in the next section.
:::

The Bellman consistency equation for a deterministic policy can be
written in tabular notation as

$$V^\pi = r^\pi + \gamma P^\pi V^\pi.$$

(Unfortunately, this notation doesn't simplify the expression for
$Q^\pi$.) This system of equations can be solved with a matrix
inversion:

:::{math}
:label: matrix_inversion_pe

V^\pi = (I - \gamma P^\pi)^{-1} r^\pi.
:::

:::{attention}
Note we've assumed that $I - \gamma P^\pi$ is invertible. Can you see
why this is the case?

(Recall that a linear operator, i.e. a square matrix, is invertible if
and only if its null space is trivial; that is, it doesn't map any
nonzero vector to zero. In this case, we can see that $I - \gamma P^\pi$
is invertible because it maps any nonzero vector to a vector with at
least one nonzero element.)
:::

```{code-cell} ipython3
def eval_deterministic_infinite(
    mdp: MDP, policy: Float[Array, "S A"]
) -> Float[Array, " S"]:
    pi = jnp.argmax(policy, axis=1)  # un-one-hot
    P_π = mdp.P[jnp.arange(mdp.S), pi]
    r_π = mdp.r[jnp.arange(mdp.S), pi]
    return jnp.linalg.solve(jnp.eye(mdp.S) - mdp.γ * P_π, r_π)
```

:::{prf:example} Tidying policy evaluation
:label: tidy_eval_infinite

Let's use the same policy $\pi$ that tidies if and only if the room is
messy. Setting $\gamma = 0.95$, we must invert

$$I - \gamma P^{\pi} = \begin{bmatrix} 1 - 0.95 \times 0.7 & - 0.95 \times 0.3 \\ - 0.95 \times 1 & 1 - 0.95 \times 0 \end{bmatrix} = \begin{bmatrix} 0.335 & -0.285 \\ -0.95 & 1 \end{bmatrix}.$$

The inverse to two decimal points is

$$(I - \gamma P^{\pi})^{-1} = \begin{bmatrix} 15.56 & 4.44 \\ 14.79 & 5.21 \end{bmatrix}.$$

Thus the value function is

$$V^{\pi} = (I - \gamma P^{\pi})^{-1} r^{\pi} = \begin{bmatrix} 15.56 & 4.44 \\ 14.79 & 5.21 \end{bmatrix} \begin{bmatrix} 1 \\ 0 \end{bmatrix} = \begin{bmatrix} 15.56 \\ 14.79 \end{bmatrix}.$$

Let's sanity-check this result. Since rewards are at most $1$, the
maximum cumulative return of a trajectory is at most
$1/(1-\gamma) = 20$. We see that the value function is indeed slightly
lower than this.
:::

```{code-cell} ipython3
eval_deterministic_infinite(tidy_mdp_inf, tidy_policy_messy_only[0])
```

(iterative_pe)=
#### Iterative policy evaluation

The matrix inversion above takes roughly $O(|\mathcal{S}|^3)$ time.
It also only works for deterministic policies.
Can we trade off the requirement of finding the *exact* value function for a faster
*approximate* algorithm that will also extend to stochastic policies?

Let's use the Bellman operator to define an iterative algorithm for
computing the value function. We'll start with an initial guess
$v^{(0)}$ with elements in $[0, 1/(1-\gamma)]$ and then iterate the
Bellman operator:

$$v^{(t+1)} = \mathcal{J}^{\pi}(v^{(t)}),$$

i.e. $v^{(t)} = (\mathcal{J}^{\pi})^{(t)} (v^{(0)})$. Note that each iteration
takes $O(|\mathcal{S}|^2)$ time for the matrix-vector multiplication.

```{code-cell} ipython3
def supremum_norm(v):
    return jnp.max(jnp.abs(v))  # same as jnp.linalg.norm(v, jnp.inf)


def loop_until_convergence(op, v, ε=1e-6):
    """Repeatedly apply op to v until convergence (in supremum norm)."""
    while True:
        v_new = op(v)
        if supremum_norm(v_new - v) < ε:
            return v_new
        v = v_new


def iterative_evaluation(mdp: MDP, pi: Float[Array, "S A"], ε=1e-6) -> Float[Array, " S"]:
    op = partial(bellman_operator, mdp, pi)
    return loop_until_convergence(op, jnp.zeros(mdp.S), ε)
```

Then, as we showed in {eq}`bellman_convergence`, by the Banach fixed-point theorem:

$$\|v^{(t)} - V^\pi \|_{\infty} \le \gamma^{t} \| v^{(0)} - V^\pi\|_{\infty}.$$

```{code-cell} ipython3
iterative_evaluation(tidy_mdp_inf, tidy_policy_messy_only[0])
```

::::{prf:remark} Convergence of iterative policy evaluation
:label: iterations_vi

How many iterations do we need for an $\epsilon$-accurate estimate? We
can work backwards to solve for $t$:

$$
\begin{aligned}
    \gamma^t \|v^{(0)} - V^\pi\|_{\infty} &\le \epsilon \\
    t &\ge \frac{\log (\epsilon / \|v^{(0)} - V^\pi\|_{\infty})}{\log \gamma} \\
    &= \frac{\log (\|v^{(0)} - V^\pi\|_{\infty} / \epsilon)}{\log (1 / \gamma)},
\end{aligned}
$$

and so the number of iterations required for an
$\epsilon$-accurate estimate is

$$
T = O\left( \frac{1}{1-\gamma} \log\left(\frac{1}{\epsilon (1-\gamma)}\right) \right).
$$

Note that we've applied the inequalities
$\|v^{(0)} - V^\pi\|_{\infty} \le 1/(1-\gamma)$ and
$\log (1/x) \ge 1-x$.
::::

### Optimal policies in infinite-horizon MDPs

Now let's move on to solving for an optimal policy in the
infinite-horizon case. As in {prf:ref}`the finite-horizon case <optimal_policy_finite>`, an **optimal policy** $\pi^\star$
is one that does at least as well as any other policy in all situations.
That is, for all policies $\pi$, states $s \in \mathcal{S}$, times
$\hi \in \mathbb{N}$, and initial trajectories
$\tau_\hi = (s_0, a_0, r_0, \dots, s_\hi)$ where $s_\hi = s$,

:::{math}
:label: optimal_policy_infinite

\begin{aligned}
    V^{\pi^\star}(s) &= \E_{\tau \sim \rho^{\pi^{\star}}}[r_\hi + \gamma r_{\hi+1} + \gamma^2 r_{\hi+2}  + \cdots \mid s_\hi = s] \\
    &\ge \E_{\tau \sim \rho^{\pi}}[r_\hi + \gamma r_{\hi+1} + \gamma^2 r_{\hi+2} + \cdots \mid \tau_\hi]
\end{aligned}
:::


Once again, all optimal policies share the same **optimal value function** $V^\star$, and the greedy policy w.r.t. this value function
is optimal.

:::{attention}
Verify this by modifying the proof {prf:ref}`optimal_greedy` from the finite-horizon case.
:::

So how can we compute such an optimal policy? We can't use the backwards
DP approach from the finite-horizon case {prf:ref}`pi_star_dp` since there's no "final timestep" to start
from. Instead, we'll exploit the fact that the Bellman consistency
equation {eq}`bellman_consistency_infinite` for the optimal value
function doesn't depend on any policy:

:::{math}
:label: bellman_optimality

V^\star(s) = \max_a \left[ r(s, a) + \gamma \E_{s' \sim P(s, a)} V^\star(s'). \right]
:::

:::{attention}
Verify this by substituting the greedy policy into the
Bellman consistency equation.
:::

As before, thinking of the r.h.s. of {eq}`bellman_optimality` as an operator on value functions
gives the **Bellman optimality operator**

:::{math}
:label: bellman_optimality_operator

[\mathcal{J}^{\star}(v)](s) = \max_a \left[ r(s, a) + \gamma \E_{s' \sim P(s, a)} v(s') \right]
:::

```{code-cell} ipython3
def bellman_optimality_operator(mdp: MDP, v: Float[Array, " S"]) -> Float[Array, " S"]:
    return jnp.max(mdp.r + mdp.γ * mdp.P @ v, axis=1)


def check_optimal(v: Float[Array, " S"], mdp: MDP):
    return jnp.allclose(v, bellman_optimality_operator(v, mdp))
```

(value_iteration)=
#### Value iteration

Since the optimal policy is still a policy, our result that the Bellman
operator is a contracting map still holds, and so we can repeatedly
apply this operator to converge to the optimal value function! This
algorithm is known as **value iteration**.

```{code-cell} ipython3
def value_iteration(mdp: MDP, ε: float = 1e-6) -> Float[Array, " S"]:
    """Iterate the Bellman optimality operator until convergence."""
    op = partial(bellman_optimality_operator, mdp)
    return loop_until_convergence(op, jnp.zeros(mdp.S), ε)
```

```{code-cell} ipython3
value_iteration(tidy_mdp_inf)
```

Note that the runtime analysis for an $\epsilon$-optimal value function
is exactly the same as [iterative policy evaluation](#iterative_pe)! This is because value iteration is simply
the special case of applying iterative policy evaluation to the
*optimal* value function.

As the final step of the algorithm, to return an actual policy
$\hat \pi$, we can simply act greedily w.r.t. the final iteration
$v^{(T)}$ of our above algorithm:

$$\hat \pi(s) = \arg\max_a \left[ r(s, a) + \gamma \E_{s' \sim P(s, a)} v^{(T)}(s') \right].$$

We must be careful, though: the value function of this greedy policy,
$V^{\hat \pi}$, is *not* the same as $v^{(T)}$, which need not even be a
well-defined value function for some policy!

The bound on the policy's quality is actually quite loose: if
$\|v^{(T)} - V^\star\|_{\infty} \le \epsilon$, then the greedy policy
$\hat \pi$ satisfies
$\|V^{\hat \pi} - V^\star\|_{\infty} \le \frac{2\gamma}{1-\gamma} \epsilon$,
which might potentially be very large.

:::{prf:theorem} Greedy policy value worsening
:label: greedy_worsen

$$\|V^{\hat \pi} - V^\star \|_{\infty} \le \frac{2 \gamma}{1-\gamma} \|v - V^\star\|_{\infty}$$

where $\hat \pi(s) = \arg\max_a q(s, a)$ is the greedy policy w.r.t.

$$q(s, a) = r(s, a) + \E_{s' \sim P(s, a)} v(s').$$
:::

:::{prf:proof} Proof
We first have

$$
\begin{aligned}
        V^{\star}(s) - V^{\hat \pi}(s) &= Q^{\star}(s,\pi^\star(s)) - Q^{\hat \pi}(s, \hat \pi(s))\\
        &= [Q^{\star}(s,\pi^\star(s)) - Q^{\star}(s, \hat \pi(s))] + [Q^{\star}(s, \hat \pi(s)) - Q^{\hat \pi}(s, \hat \pi(s))].
\end{aligned}
$$

Let's bound these two quantities separately.

For the first quantity, note that by the definition of $\hat \pi$, we have

$$q(s, \hat \pi(s)) \ge q(s,\pi^\star(s)).$$

Let's add $q(s, \hat \pi(s)) - q(s,\pi^\star(s)) \ge 0$ to the first term to get

$$
\begin{aligned}
        Q^{\star}(s,\pi^\star(s)) - Q^{\star}(s, \hat \pi(s)) &\le [Q^{\star}(s,\pi^\star(s))- q(s,\pi^\star(s))] + [q(s, \hat \pi(s)) - Q^{\star}(s, \hat \pi(s))] \\
        &= \gamma \E_{s' \sim P(s, \pi^{\star}(s))} [ V^{\star}(s') - v(s') ] + \gamma \E_{s' \sim P(s, \hat \pi(s))} [ v(s') - V^{\star}(s') ] \\
        &\le 2 \gamma \|v - V^{\star}\|_{\infty}.
\end{aligned}
$$


The second quantity is bounded by

$$
\begin{aligned}
        Q^{\star}(s, \hat \pi(s)) - Q^{\hat \pi}(s, \hat \pi(s))
        &=
        \gamma \E_{s'\sim P(s, \hat \pi(s))}\left[ V^\star(s') - V^{\hat \pi}(s') \right] \\
        & \leq 
        \gamma \|V^{\star} - V^{\hat \pi}\|_\infty
\end{aligned}
$$

and thus

$$
\begin{aligned}
        \|V^\star - V^{\hat \pi}\|_\infty &\le 2 \gamma \|v - V^{\star}\|_{\infty} + \gamma \|V^{\star} - V^{\hat \pi}\|_\infty \\
        \|V^\star - V^{\hat \pi}\|_\infty &\le \frac{2 \gamma \|v - V^{\star}\|_{\infty}}{1-\gamma}.
\end{aligned}
$$
:::

So in order to compensate and achieve $\|V^{\hat \pi} - V^{\star}\| \le \epsilon$, we must have

$$\|v^{(T)} - V^\star\|_{\infty} \le \frac{1-\gamma}{2 \gamma} \epsilon.$$

This means, using {prf:ref}`iterations_vi`, we need to run value iteration for

$$T = O\left( \frac{1}{1-\gamma} \log\left(\frac{\gamma}{\epsilon (1-\gamma)^2}\right) \right)$$

iterations to achieve an $\epsilon$-accurate estimate of the optimal value function.


(policy_iteration)=
#### Policy iteration

Can we mitigate this "greedy worsening"? What if instead of approximating the optimal value function and then acting greedily by it at the very end, we iteratively improve the policy and value function *together*? This is the idea behind **policy iteration**. In each step, we simply set the policy to act greedily with respect to its own value function.

```{code-cell} ipython3
def policy_iteration(mdp: MDP, ε=1e-6) -> Float[Array, "S A"]:
    """Iteratively improve the policy and value function."""
    def op(pi):
        return v_to_greedy(mdp, eval_deterministic_infinite(mdp, pi))
    π_init = jnp.ones((mdp.S, mdp.A)) / mdp.A  # uniform random policy
    return loop_until_convergence(op, π_init, ε)
```

```{code-cell} ipython3
policy_iteration(tidy_mdp_inf)
```

Although PI appears more complex than VI, we'll use the same contraction property {prf:ref}`bellman_contraction` to show convergence. This will give us the same runtime bound as value iteration and iterative policy evaluation for an $\epsilon$-optimal value function {prf:ref}`iterations_vi`, although in practice, PI often converges much faster.

::::{prf:theorem} Policy Iteration runtime and convergence
:label: pi_iter_analysis

We aim to show that the number of iterations required for an
$\epsilon$-accurate estimate of the optimal value function is

$$T = O\left( \frac{1}{1-\gamma} \log\left(\frac{1}{\epsilon (1-\gamma)}\right) \right).$$

This bound follows from the contraction property {eq}`bellman_convergence`:

$$\|V^{\pi^{t+1}} - V^\star \|_{\infty} \le \gamma \|V^{\pi^{t}} - V^\star \|_{\infty}.$$

We'll prove that the iterates of PI respect the contraction property by
showing that the policies improve monotonically:

$$V^{\pi^{t+1}}(s) \ge V^{\pi^{t}}(s).$$

Then we'll use this to show
$V^{\pi^{t+1}}(s) \ge [\mathcal{J}^{\star}(V^{\pi^{t}})](s)$. Note that

$$
\begin{aligned}
(s) &= \max_a \left[ r(s, a) + \gamma \E_{s' \sim P(s, a)} V^{\pi^{t}}(s') \right] \\
    &= r(s, \pi^{t+1}(s)) + \gamma \E_{s' \sim P(s, \pi^{t+1}(s))} V^{\pi^{t}}(s')
\end{aligned}
$$

Since
$[\mathcal{J}^{\star}(V^{\pi^{t}})](s) \ge V^{\pi^{t}}(s)$, we then have

:::{math}
:label: pi_iter_proof

\begin{aligned}
    V^{\pi^{t+1}}(s) - V^{\pi^{t}}(s) &\ge V^{\pi^{t+1}}(s) - \mathcal{J}^{\star} (V^{\pi^{t}})(s) \\
    &= \gamma \E_{s' \sim P(s, \pi^{t+1}(s))} \left[V^{\pi^{t+1}}(s') -  V^{\pi^{t}}(s') \right].
\end{aligned}
:::

But note that the
expression being averaged is the same as the expression on the l.h.s.
with $s$ replaced by $s'$. So we can apply the same inequality
recursively to get

$$
\begin{aligned}
    V^{\pi^{t+1}}(s) - V^{\pi^{t}}(s) &\ge  \gamma \E_{s' \sim P(s, \pi^{t+1}(s))} \left[V^{\pi^{t+1}}(s') -  V^{\pi^{t}}(s') \right] \\
    &\ge \gamma^2 \E_{\substack{s' \sim P(s, \pi^{t+1}(s)) \\ s'' \sim P(s', \pi^{t+1}(s'))}} \left[V^{\pi^{t+1}}(s'') -  V^{\pi^{t}}(s'') \right]\\
    &\ge \cdots
\end{aligned}
$$

which implies that $V^{\pi^{t+1}}(s) \ge V^{\pi^{t}}(s)$
for all $s$ (since the r.h.s. converges to zero). We can then plug this
back into
{eq}`pi_iter_proof`
to get the desired result:

$$
\begin{aligned}
    V^{\pi^{t+1}}(s) - \mathcal{J}^{\star} (V^{\pi^{t}})(s) &= \gamma \E_{s' \sim P(s, \pi^{t+1}(s))} \left[V^{\pi^{t+1}}(s') -  V^{\pi^{t}}(s') \right] \\
    &\ge 0 \\
    V^{\pi^{t+1}}(s) &\ge [\mathcal{J}^{\star}(V^{\pi^{t}})](s)
\end{aligned}
$$

This means we can now apply the Bellman convergence result {eq}`bellman_convergence` to get

$$\|V^{\pi^{t+1}} - V^\star \|_{\infty} \le \|\mathcal{J}^{\star} (V^{\pi^{t}}) - V^{\star}\|_{\infty} \le \gamma \|V^{\pi^{t}} - V^\star \|_{\infty}.$$
::::

## Summary

-   Markov decision processes (MDPs) are a framework for sequential
    decision making under uncertainty. They consist of a state space
    $\mathcal{S}$, an action space $\mathcal{A}$, an initial state distribution
    $\mu \in \Delta(\mathcal{S})$, a transition function $P(s' \mid s, a)$, and a
    reward function $r(s, a)$. They can be finite-horizon (ends after
    $H$ timesteps) or infinite-horizon (where rewards scale by
    $\gamma \in (0, 1)$ at each timestep).

-   Our goal is to find a policy $\pi$ that maximizes expected total
    reward. Policies can be **deterministic** or **stochastic**,
    **state-dependent** or **history-dependent**, **stationary** or
    **time-dependent**.

-   A policy induces a distribution over **trajectories**.

-   We can evaluate a policy by computing its **value function**
    $V^\pi(s)$, which is the expected total reward starting from state
    $s$ and following policy $\pi$. We can also compute the
    **state-action value function** $Q^\pi(s, a)$, which is the expected
    total reward starting from state $s$, taking action $a$, and then
    following policy $\pi$. In the finite-horizon setting, these also
    depend on the timestep $\hi$.

-   The **Bellman consistency equation** is an equation that the value
    function must satisfy. It can be used to solve for the value
    functions exactly. Thinking of the r.h.s. of this equation as an
    operator on value functions gives the **Bellman operator**.

-   In the finite-horizon setting, we can compute the optimal policy
    using **dynamic programming**.

-   In the infinite-horizon setting, we can compute the optimal policy
    using **value iteration** or **policy iteration**.
