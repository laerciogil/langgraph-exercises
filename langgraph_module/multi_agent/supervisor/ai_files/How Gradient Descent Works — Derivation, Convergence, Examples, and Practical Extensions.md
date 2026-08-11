Introduction

Gradient descent is the workhorse of numerical optimization and machine learning. From fitting a simple linear model to training deep neural networks, variants of gradient descent power modern algorithms. This post explains how gradient descent works: the intuitive idea, a clean derivation of the update rule from Taylor expansion and surrogate minimization, precise statements about step-size choices, convergence rates (sublinear and linear) with a proof sketch for convex L‑smooth functions, worked numerical examples (1‑D and 2‑D quadratics) including exact line search, and practical variants: momentum (Polyak heavy‑ball), Nesterov acceleration, stochastic gradient descent and adaptive optimizers (AdaGrad, RMSProp, Adam). The target reader is comfortable with undergraduate calculus and linear algebra; I aim for clarity with mathematical rigor and reproducible examples (NumPy‑style code).

Intuition: steepest descent and local quadratic models

Suppose you want to minimize a differentiable function f: R^n → R. The gradient ∇f(x) is the vector of partial derivatives and points in the direction of steepest ascent (in Euclidean norm). Taking a small step in the negative gradient direction therefore decreases the function most rapidly for infinitesimal steps. Concretely, the directional derivative along displacement s is ∇f(x)^T s, so choosing s = −t ∇f(x) (t>0 scalar) gives the first‑order decrease −t ||∇f(x)||^2.

Why not take a huge step? Because the linear approximation breaks down for large steps. A simple way to control step length is to minimize a short‑range surrogate that couples the linear model with a quadratic proximity term:

m_t(y) = f(x) + ∇f(x)^T (y − x) + (1/(2t)) ||y − x||^2.

Minimizing m_t over y balances making progress along −∇f(x) while staying near x. Solving ∇_y m_t(y)=0 gives the familiar gradient‑descent update:

x_{new} = x − t ∇f(x).

Derivation of the update: 1D and vector cases

1D derivation (Taylor): For scalar x and small increment s,

f(x + s) = f(x) + f'(x) s + o(s).

To decrease f choose s proportional to −f'(x). The first‑order approximate decrease for s = −t f'(x) is

f(x − t f'(x)) ≈ f(x) − t (f'(x))^2.

This motivates the 1D update x ← x − t f'(x).

Vector derivation (surrogate minimization / quadratic approximation): Use first‑order Taylor

f(y) ≈ f(x) + ∇f(x)^T (y − x).

Add a quadratic penalty to obtain m_t(y) above. Setting ∇_y m_t(y) = 0 gives

0 = ∇f(x) + (1/t) (y − x) ⇒ y = x − t ∇f(x),

so the vector update is

(1) x^{(k+1)} = x^{(k)} − t_k ∇f(x^{(k)}).

Interpretation: move along the steepest descent direction with step length tuned by t_k.

Learning rate (step size) choices and line search

The scalar t_k is critical.

- Too small: progress is tiny. - Too large: the quadratic surrogate becomes a poor model and the iterate may overshoot and diverge or oscillate.

Common strategies

- Fixed step: t_k = t. Simple; choose t ∈ (0, 1/L] when ∇f is L‑Lipschitz to guarantee descent (details below). - Exact line search: t_k = argmin_{s≥0} f(x_k − s ∇f(x_k)). For quadratics this has a closed form (see examples), but is often expensive otherwise. - Backtracking (Armijo) line search: start from an initial t and reduce by factor β ∈ (0,1) until

f(x − t ∇f(x)) ≤ f(x) − α t ||∇f(x)||^2

holds (α ∈ (0, 1/2)). Backtracking ensures sufficient decrease and is practical.

Convergence for convex L‑smooth functions (O(1/k)) — statement and proof sketch

Assumptions:

- f is convex and differentiable with minimizer x^* and optimal value f^* = f(x^*). - Gradient is L‑Lipschitz: ||∇f(x) − ∇f(y)|| ≤ L ||x − y|| for all x,y (equivalently, when twice differentiable, ∇^2 f(x) ⪯ L I).

Theorem (sublinear rate). For gradient descent with fixed step t satisfying 0 < t ≤ 1/L, the iterates x^{(k)} obey

(2) f(x^{(k)}) − f^* ≤ ||x^{(0)} − x^*||^2 / (2 t k).

Thus gradient descent attains O(1/k) convergence in objective suboptimality.

Proof sketch (key inequalities)

1. Descent lemma (from Lipschitz gradient): for any x,y

f(y) ≤ f(x) + ∇f(x)^T (y − x) + (L/2) ||y − x||^2.

Set y = x − t ∇f(x). Then

f(x − t ∇f(x)) ≤ f(x) − t ||∇f(x)||^2 + (L t^2 /2) ||∇f(x)||^2 = f(x) − t(1 − (L t)/2) ||∇f(x)||^2.

If 0 < t ≤ 1/L then 1 − (L t)/2 ≥ 1/2, so

(3) f(x^{(k+1)}) ≤ f(x^{(k)}) − (t/2) ||∇f(x^{(k)})||^2.

2. Convexity implies for any z (in particular z = x^*)

f(x) − f(z) ≤ ∇f(x)^T (x − z) ≤ ||∇f(x)|| ||x − z||.

Combining algebraic identities (expand ||x − x^*||^2 − ||x^{+} − x^*||^2 after one step) and telescoping yields (2). See Boyd & Vandenberghe for the full algebraic steps.

Remarks on the rate: O(1/k) is sublinear — the objective gap decays inversely with iteration count. For nonconvex L‑smooth f, one typically obtains stationarity bounds (min_k ||∇f(x^{(k)})||^2 = O(1/k)).

Strong convexity: linear (geometric) convergence

If f is µ‑strongly convex (µ > 0) and ∇f is L‑Lipschitz, stronger guarantees hold. Strong convexity means for all x,y:

f(y) ≥ f(x) + ∇f(x)^T (y − x) + (µ/2) ||y − x||^2.

With fixed step t ∈ (0, 2/(µ + L)] (common practical choice t = 1/L), gradient descent converges linearly: there exists ρ ∈ (0,1) such that

f(x^{(k)}) − f^* ≤ C ρ^k

for some constant C depending on the initialization. A common form (t = 1/L) gives ρ = 1 − µ/L = 1 − 1/κ, where κ = L/µ is the condition number; more refined bounds give ρ ≈ (1 − sqrt(µ/L))/(1 + sqrt(µ/L)) for optimally tuned momentum/acceleration.

Intuition: strong convexity provides a quadratic lower bound around the minimizer, ensuring that gradient information yields a uniform contraction per step; the contraction factor depends on curvature (µ,L) and so on conditioning κ.

Worked numerical examples (reproducible)

1) 1D quadratic example

Consider f(x) = (1/2) a x^2 + b x + c with a > 0. Then f'(x) = a x + b and minimizer x^* = −b/a. Gradient step: x_{k+1} = x_k − t (a x_k + b). Let r = 1 − t a. The recurrence solves to

x_k = r^k (x_0 − x^*) + x^*.

Converges iff |r| < 1 ⇔ 0 < t < 2/a; monotone decrease in value holds for t ≤ 1/a = 1/L.

Numeric instance: a = 2, b = −4 (so x^* = 2), x_0 = 0. L = a = 2.

Table: x_k for k = 0..5 and several t values

t = 0.1 (r = 0.8)
0 0.0000
1 0.4000
2 0.7200
3 0.9760
4 1.1808
5 1.3446

t = 0.5 (r = 0)
0 0.0000
1 2.0000
2 2.0000
3 2.0000
4 2.0000
5 2.0000

t = 0.8 (r = -0.6)
0 0.0000
1 3.2000
2 1.2800
3 2.4320
4 1.7408
5 2.1555

t = 1.2 (r = -1.4) — diverging
0 0.0000
1 4.8000
2 -1.9200
3 7.4880
4 -9.3312
5 18.1786

Reproducible NumPy snippet (1D):

import numpy as np

a = 2.0; b = -4.0
x0 = 0.0
for t in [0.1, 0.5, 0.8, 1.2]:
    x = x0
    print('\nstep t =', t)
    for k in range(6):
        print(k, np.round(x, 6))
        g = a*x + b
        x = x - t*g

2) 2D diagonal quadratic (conditioning effects)

Let f(x) = (1/2) x^T A x + b^T x with A = diag(1,10) and b = [−2, −20]^T. Minimizer x^* = −A^{-1} b = [2,2]^T. L = 10, condition number κ = 10.

Because A is diagonal, updates decouple componentwise: the shrink factors are r_1 = 1 − t·1 and r_2 = 1 − t·10. Poor conditioning (λ_1 ≪ λ_2) means the coordinates converge at very different rates and iterates can zig‑zag in level sets.

Take x_0 = [0,0]. Table for k = 0..5

t = 0.05 (r = [0.95, 0.5])
k [x1, x2]
0 [0.0000, 0.0000]
1 [0.1000, 1.0000]
2 [0.1950, 1.5000]
3 [0.2853, 1.7500]
4 [0.3720, 1.8750]
5 [0.4534, 1.9375]

t = 0.1 (r = [0.9, 0])
0 [0.0000, 0.0000]
1 [0.2000, 2.0000]
2 [0.3800, 2.0000]
3 [0.5420, 2.0000]
4 [0.6878, 2.0000]
5 [0.8190, 2.0000]

t = 0.15 (r = [0.85, -0.5]) — oscillatory in second coord
0 [0.0000, 0.0000]
1 [0.3000, 3.0000]
2 [0.5550, 1.5000]
3 [0.9268, 2.2500]
4 [1.1958, 1.8750]
5 [1.4164, 2.0625]

NumPy snippet (2D):

import numpy as np
A = np.diag([1.0, 10.0])
b = np.array([-2., -20.])
for t in [0.05, 0.1, 0.15]:
    x = np.array([0., 0.])
    print('\n t =', t)
    for k in range(6):
        print(k, np.round(x, 6))
        g = A.dot(x) + b
        x = x - t * g

3) Exact line search for quadratics (closed form)

For quadratic f(x) = (1/2) x^T A x + b^T x + c and steepest descent direction p_k = −g_k, the scalar function φ(t) = f(x_k + t p_k) is a univariate quadratic. Set φ'(t) = 0 to obtain the optimal step

(4) t_k = (g_k^T g_k) / (g_k^T A g_k).

This is cheap for moderate‑scale problems and can dramatically accelerate convergence on quadratics.

Numeric demonstration with the 2D example and x0 = [0,0]:

g0 = b = [−2, −20] ⇒ g0^T g0 = 404, g0^T A g0 = 4004, so t0 = 404/4004 ≈ 0.1009. The exact line step yields x1 ≈ [0.2018, 2.018]. Repeating exact line searches yields rapid convergence.

Practical extensions and variants

Momentum (Polyak heavy‑ball)

Update (heavy‑ball):

x_{k+1} = x_k − α ∇f(x_k) + β (x_k − x_{k-1}).

Equivalently, define velocity v_{k+1} = β v_k − α ∇f(x_k) and x_{k+1} = x_k + v_{k+1}. Intuition: momentum accumulates past gradients to build inertia, helps traverse flat valleys and damp oscillations across steep curvature. For strongly convex quadratics optimal α,β can be chosen to minimize the spectral radius; the method can achieve accelerated linear rates empirically, but global O(1/k^2) guarantees like Nesterov's do not hold in the same generality.

Nesterov accelerated gradient (NAG)

One common form:

t_{k+1} = (1 + sqrt(1 + 4 t_k^2))/2,    β_k = (t_k − 1) / t_{k+1}

y_k = x_k + β_k (x_k − x_{k-1})

x_{k+1} = y_k − (1/L) ∇f(y_k).

Nesterov acceleration evaluates the gradient at a look‑ahead point y_k and achieves the optimal O(1/k^2) rate for convex L‑smooth objectives (f(x_k) − f* = O(1/k^2)). For µ‑strongly convex problems, accelerated linear rates improve dependence on condition number (roughly (1 − 1/√κ)^k behavior in the exponent).

Stochastic gradient descent (SGD) basics

SGD replaces the full gradient by a cheap unbiased estimator g_k with E[g_k|x_k] = ∇F(x_k). Update:

x_{k+1} = x_k − α_k g_k.

Step‑size schedules matter: constant α gives fast initial progress but a nonzero steady‑state variance; diminishing α_k (e.g. α_k = α_0/(1 + γ k) or α_k = α_0/k) yields asymptotic convergence under Robbins–Monro conditions (∑ α_k = ∞, ∑ α_k^2 < ∞). For strongly convex problems, SGD with appropriate decay achieves O(1/k) rates in expectation.

Adaptive optimizers: AdaGrad, RMSProp, Adam

These methods adapt per‑coordinate step sizes using accumulated gradient statistics. They are especially useful in high‑dimensional or sparse problems.

AdaGrad (per‑coordinate scaling)

s_t = s_{t−1} + g_t ⊙ g_t
x_{t+1} = x_t − α (g_t / (sqrt(s_t) + ε)).

Pros: automatic per‑coordinate decay; good for sparse features. Cons: s_t accumulates forever and may shrink steps too much.

RMSProp

v_t = ρ v_{t−1} + (1 − ρ) g_t ⊙ g_t
x_{t+1} = x_t − α g_t / (sqrt(v_t) + ε).

RMSProp fixes AdaGrad’s aggressive accumulation via exponential averaging; widely used in deep learning.

Adam (Kingma & Ba)

m_t = β1 m_{t−1} + (1 − β1) g_t
v_t = β2 v_{t−1} + (1 − β2) (g_t ⊙ g_t)

Bias corrections: m̂_t = m_t/(1 − β1^t), v̂_t = v_t/(1 − β2^t)

x_{t+1} = x_t − α m̂_t / (sqrt(v̂_t) + ε).

Adam combines momentum (first moment) with adaptive scaling (second moment). It is fast and requires little tuning, but has known caveats: possible nonconvergence in some pathological settings and sometimes worse generalization than SGD with momentum. Variants such as AMSGrad provide stronger convergence guarantees.

Comparing convergence rates and when they apply

- Plain gradient descent (convex L‑smooth): f(x_k) − f* = O(1/k) with t ∈ (0, 1/L]. - Nesterov accelerated gradient (convex L‑smooth): f(x_k) − f* = O(1/k^2) with step α = 1/L and the t_k/β_k schedule above. - Strongly convex (µ > 0): gradient descent is linear with contraction factor roughly 1 − µ/L per step (depends on t). Accelerated methods improve the dependence on κ = L/µ (roughly replacing κ by sqrt(κ) in the exponent). - SGD (stochastic, convex): with appropriate decaying stepsizes, sublinear rates like O(1/k) (strongly convex) or O(1/√k) (general convex/stationarity) are typical; constant stepsizes trade bias for variance.

Practical recommendations (short)

- For small/medium convex problems: use gradient descent with backtracking or Nesterov acceleration if you want fast objective decrease. - For large‑scale nonconvex problems (deep learning): SGD with momentum + learning rate schedule is a robust baseline; Adam/RMSProp are good when sparse/ill‑scaled features exist or for fast prototyping. - Tune step sizes or use line search when feasible; prefer t ≤ 1/L when L is known/estimated. - For ill‑conditioned problems, preconditioning (diagonal scaling, L‑BFGS or second‑order methods) helps.

References

- S. Boyd and L. Vandenberghe, Convex Optimization. - Y. Nesterov, Introductory Lectures on Convex Optimization (and original 1983 paper). - W. Su, S. Boyd, E. J. Candès, "A differential equation for modeling Nesterov's accelerated gradient method," 2014. - D. Kingma and J. Ba, "Adam: A Method for Stochastic Optimization," ICLR 2015. - B. T. Polyak, "Some methods of speeding up the convergence of iteration methods," 1964.

Conclusion

Gradient descent is simple, well understood, and highly effective when applied with appropriate step‑size choices and preconditioning. The theoretical landscape gives clear guidance: use t ≤ 1/L for reliable descent, expect O(1/k) for plain gradient descent on convex L‑smooth problems, and use acceleration or momentum when you need faster convergence or to mitigate conditioning. In stochastic settings, step‑size schedules and variance reduction are the keys. The examples and snippets above give a concrete starting point to explore these behaviors numerically.