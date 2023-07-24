# Markov Decision Processes

How can we *formalize* a reinforcement learning task in a way that is
both *sufficiently general* yet also tractable enough for *fruitful
analysis*?

In this chapter, we'll turn to **Markov decision processes** as a simple
yet general formalism for solving decision problems.

::: definition
Markov Decision Processmdp

The key components of a Markov decision process are:

1.  The **state** (a.k.a. the **environment**) that the agent interacts
    with. We use $\S$ to denote the set of possible states, called the
    **state space**.

2.  The **agent** and the **actions** that it can take. We use $\A$ to
    denote the set of possible actions, called the **action space**.

3.  The **reward** signal. In this course we'll take it to be a
    deterministic function of a state-action pair, i.e.
    $r : \S \times \A \to \mathbb{R}$. In general, though, the reward
    function can also be stochastic, and it can also accept the
    *resulting* state as an argument; that is,
    $r : \S \times \A \times \S \to \Delta(\R)$.

4.  The **state transitions** (a.k.a. **dynamics**) that describe what
    state we **transition to** after taking an action. We'll denote this
    by $P : \S \times \A \to \Delta(\S)$ (as opposed to $\P$ which
    denotes the underlying probability measure.)

5.  A *discount factor* $\gamma \in [0, 1)$. We'll see later that this
    ensures that the *return*, or total reward, is well-defined in
    infinite-horizon problems.

6.  Some **initial state distribution** $\rho \in \Delta(\S)$.

Combined together, we call these a Markov decision process

$$M = (\S, \A, P, r, \gamma, \rho).$$
:::

The reason we call it a *Markov* decision process is that the transition
function only depends on the "current" state and action. Formally, this
implies that the state process satisfies the **Markov property**, that
is,

$$\P(s_{t+1} \mid (s_\tau, a_\tau)_{\tau=0}^t) = P(s_{t+1} \mid s_t, a_t).$$

::: example
Examples of MDPsmdp_examples **Board games and video games** are often
MDPs. For example, in chess or Go, the state of the game only depends on
the pieces on the board and not on the previous history. Several
possible reward functions could be possible, e.g. $+1$ upon winning the
game and $0$ otherwise, or to receive reward upon taking the opponent's
pieces. The state transitions are based on the opponent's moves.

**Robotic control** can be framed as an MDP task. In this setting,
physics provides the state transitions. A possible action might be
activating a motor to move forwards. The reward function could be
designed based on the task; for example, one could reward the robot for
arriving at a desired location.
:::

We'll distinguish between **finite-horizon** MDPs, where the agent
eventually enters a **terminal state**, and **infinite-horizon** MDPs,
where the agent might keep going on and on.

We call the total reward the *return*. For finite-horizon MDPs, we can
just add up the rewards:

$$G_t := R_t + R_{t+1} + \cdots + R_T,$$

where $T$ is the number of time steps and $R_t := r(S_t, A_t)$. However,
for infinite-horizon problems (i.e. $T = \infty$), in order for this to
be well-defined, we need to *discount* future rewards:

$$G_t := R_t + \gamma R_{t+1} + \cdots + \gamma^{\tau-t} R_{\tau} + \cdots = \sum_{\tau = t}^\infty \gamma^{\tau-t} R_\tau.$$

Can you see why this ensures that $G_t$ is finite?

Note that we recover the finite-horizon definition by letting
$\gamma = 1$ and $T$ be finite.

Our key *goal* in a reinforcement learning task is to *maximize expected
return*.

Why can't we just maximize the current reward at each timestep, i.e. use
a greedy strategy? Well, in RL as in real life, often making greedy
decisions (e.g. procrastinating) will leave you worse off than if you
make some short-term sacrifices for long-term gains.

We call the "video recording" of states, actions, and rewards a
**trajectory**

$$\xi_t = (s_\tau, a_\tau, r_\tau)_{\tau=0}^t$$

## Policies and value functions

A **policy** $\pi$ describes the agent's strategy: which actions it
takes in a given situation.

Policies can either be **deterministic** (in the same situation, the
agent will always take the same action) or **stochastic** (in the same
situation, the agent will sample an action from a distribution).

What do I mean by "situation"? In the most general setting, this could
include all of the states, actions, and rewards in the trajectory so
far.

However, due to the Markov assumption, the state transitions only depend
on the current state. Thus a **stationary** policy
$\pi : \S \to \Delta(\A)$ --- one that only depends on the current state
--- can do just as well.

Fix a policy $\pi$. We'd like a concise way to refer to the expected
return when *starting in a given state* and acting according to $\pi$.
We call this the **value function** of $\pi$ and denote it by

$$\begin{aligned}
    V^\pi(s) &:= \E_\pi [G_0 \mid S_0 = s] % \\
    % &= \E_\pi \left[ \sum_{t=0}^\infty \gamma^t R_t \mid S_0 = s \right].
\end{aligned}$$

We start at time $0$ without loss of generality; can you see why we
could have chosen to start at any time?

Similarly, we can define the **action-value function** of $\pi$ (aka the
**Q-function**) as the expected return when starting in a given state
and taking a given action:

$$\begin{aligned}
    Q^\pi(s, a) &:= \E_\pi [G_0 \mid S_0 = s, A_0 = a] % \\
    % &= \E_\pi [\sum_{t=0}^\infty \gamma^t R_t \mid S_0 = s, A_0 = a].
\end{aligned}$$

### Bellman self-consistency equations

Note that we can break down the return as

$$G_t = R_t + \gamma G_{t+1}:$$

the reward from the *current time-step* and that from *future
time-steps*. It turns out that this simple observation gives us a way to
solve for the value function analytically!

Let's expand out the definition of the value function to see what I
mean. Let's first consider the simple case where $\pi : \S \to \A$ is
deterministic:

$$\begin{aligned}
    V^\pi(s) &:= \E_\pi[G_0 \mid S_0 = s] \\
    &= r(s, \pi(s)) + \gamma \E_{s' \sim P(\cdot \mid s, \pi(s))} {\color{blue} \E_\pi [G_1 \mid S_1 = s']} \\
    &= r(s, \pi(s)) + \gamma \sum_{s' \in \S} P(s' \mid s, \pi(s)) {\color{blue} V^\pi(s')}.
\end{aligned}$$

For stochastic policies, we simply average out over the relevant
quantities:

$$\begin{aligned}
    V^\pi(s) &:= \E_\pi [G_0 \mid S_0 = s] \\
    % &= \E_\pi \left[ \sum_{t=0}^\infty \gamma^t R_t \mid S_0 = s \right] \\
    &= \E_\pi \left[ R_0 + \gamma G_1 \mid S_0 = s \right] \\
    &= \E_{a \sim \pi(\cdot \mid s)} \left[ r(s, a) + \gamma \E_{s' \sim P(\cdot \mid s, a)}  {\color{blue} \E_\pi [G_1 \mid S_1 = s']} \right] \\
    &= \sum_a \pi(a \mid s) \left[r(s, a) + \gamma \sum_{s'} P(s' \mid s, a) {\color{blue} V^\pi(s')} \right].
\end{aligned}$$

## Finite MDPs

When the state and action space are finite, we can neatly express
quantities as vectors and matrices:

$$r \in \R^{|\S| \times |\A|}, \quad P \in [0, 1]^{|\S| \times (|\S| \times |\A|)}, \quad \rho \in [0, 1]^{|\S|}, \quad \pi \in [0, 1]^{|\A| \times |\S|}, \quad V^\pi \in \R^{|\S|}, \quad Q^\pi \in \R^{|\S| \times |\A|}.$$

## Exercises

Show that without discounting, the reward

## Optimality

::: theorem
Value Iterationval_iter

Initialize:

$$V^0 \sim \|V^0\|_\infty \in [0, 1/1-\gamma]$$

Iterate until convergence:

$$V^{t+1} \gets \mathcal{J}(V^t)$$

This algorithm runs in $O(|\S|^3)$ time since we need to perform a
matrix inversion.
:::

::: theorem
Exact Policy Evaluationexact_pi_eval

Represent the reward from each state-action pair as a vector

$$R^\pi \in \R^{|\S|} \qquad R^\pi_s = r(s, \pi(s))$$

Also represent the state transitions

$$P^\pi \in \R^{|\S \times \S} \qquad P^\pi_{s, s'} = P(s' | s, \pi(s))$$

That is, row $i$ of $P^\pi$ is a distribution over the *next state*
given that the current state is $s_i$ and we choose an action using
policy $\pi$.

Using this notation, we can express the Bellman consistency equation as

$$\begin{aligned}
    \begin{pmatrix}
        \vdots \\ V^\pi(s) \\ \vdots
    \end{pmatrix}
    &=
    \begin{pmatrix}
        \vdots \\ r(s, \pi(s)) \\ \vdots
    \end{pmatrix}
    +
    \gamma
    \begin{pmatrix}
        & \vdots & \\
        \quad & P(s' \mid s, \pi(s)) & \quad \\
        & \vdots &
    \end{pmatrix}
    \begin{pmatrix}
        \vdots \\ V^\pi(s') \\ \vdots
    \end{pmatrix} \\
    V^\pi &= R^\pi + \gamma P^\pi V^\pi \\
    (I - \gamma P^\pi) V^\pi &= R^\pi \\
    V^\pi &= (I - \gamma P^\pi) R^\pi
\end{aligned}$$

if $I - \gamma P^\pi$ is invertible, which we can prove is the case.
:::

::: theorem
Iterative Policy Evaluationiter_pi_eval

How can we calculate the value function $V^\pi$ of a policy $\pi$?

Above, we saw an exact function that runs in $O(|\S|^2)$. But say we
really need a fast algorithm, and we're okay with having an approximate
answer. Can we do better? Yes!

Using the same notation as above, let's initialize $V^0$ such that the
elements are drawn uniformly from $[0, 1/(1-\gamma)]$.

Then we can iterate the fixed-point equation we found above:

$$V^{t+1} \gets R + \gamma P V^t$$
:::

How can we use this fast approximate algorithm?

::: theorem
Policy Iterationpi_iter

Remember, for now we're only considering policies that are *stationary
and deterministic*. There's $|\S|^{|\A}$ of these, so let's start off by
choosing one at random. Let's call this initial policy $\pi^0$, using
the superscript to indicate the time step.

Now for $t = 0, 1, \dots$, we perform the following:

1.  *Policy Evaluation*: First use the algorithm from earlier to
    calculate $V^{\pi^t}(s)$ for all states $s$. Then use this to
    calculate the state-action values:

    $$Q^{\pi^t}(s, a) = r(s, a) + \gamma \sum_{s'} P(s' \mid s, a) V^{\pi^t} (s')$$

2.  *Policy Improvement*: Update the policy so that, at each state, it
    chooses the action with the highest action-value:

    $$\pi^{t+1}(s) = \argmax_a Q^{\pi^t} (s, a)$$

    In other words, we're setting it to act greedily with respect to the
    new Q-function.

What's the computational complexity of this?
:::

## Finite Horizon MDPs

Suppose we're only able to act for $H$ timesteps.

Now, instead of discounting, all we care about is the (average) total
reward that we get over this time.

$$\E[ \sum_{t=0}^{H-1} r(s_t, a_t) ]$$

To be more precise, we'll consider policies that depend on the time.
We'll denote the policy at timestep $h$ as $\pi_h : \S \to \A$. In other
words, we're dropping the constraint that policies must be stationary.

This is also called an *episodic model*.

Note that since our policy is nonstationary, we also need to adjust our
value function (and Q-function) to account for this. Instead of
considering the total infinite-horizon discounted reward like we did
earlier, we'll instead consider the *remaining* reward from a given
timestep onwards:

$$\begin{aligned}
    V^\pi_h(s) &= \E \left[ \sum_\tau^{H-1} r(s_\tau, a_\tau) \mid s_h = s, a_\tau = \pi_h(s_h) \right] \\
    Q^\pi_h(s, a) &= \E \left[ \sum_\tau^{H-1} r(s_\tau, a_\tau) \mid (s_h, a_h) = (s, a) \right]
\end{aligned}$$

We can also define our Bellman consistency equations, by splitting up
the total reward into the immediate reward (at this time step) and the
future reward, represented by our state value function from that next
time step:

$$Q^\pi_h(s, a) = r(s, a) + \E_{s' \sim P(s, a)}[V^\pi_{h+1}(s')]$$

::: theorem
Computing the optimal policypi_star_dp

We can solve for the optimal policy using dynamic programming.

-   *Base case.* At the end of the episode (time step $H-1$), we can't
    take any more actions, so the $Q$-function is simply the reward that
    we obtain:

    $$Q^\star_{H-1}(s, a) = r(s, a)$$

    so the best thing to do is just act greedily and get as much reward
    as we can!

    $$\pi^\star_{H-1}(s) = \argmax_a Q^\star_{H-1}(s, a)$$

    Then $V^\star_{H-1}(s)$, the optimal value of state $s$ at the end
    of the trajectory, is simply whatever action gives the most reward.

    $$V^\star_{H-1} = \max_a Q^\star_{H-1}(s, a)$$

-   *Recursion.* Then, we can work backwards in time, starting from the
    end, using our consistency equations!

Note that this is exactly just value iteration and policy iteration
combined, since our policy is nonstationary, so we can exactly specify
its decisions at each time step!

Total computation time $O(H |\S|^2 |\A|)$
:::