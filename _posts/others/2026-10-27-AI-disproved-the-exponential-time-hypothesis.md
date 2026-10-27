---
title: "AI disproved the exponential time hypoothesis"
last_modified_at: 2026-10-27T17:53:13+09:00
categories:
    - others
tags:
    - others

toc: true
toc_label: "My Table of Contents"
author_profile: true
---
OpenAI: https://openai.com/index/building-scientific-understanding-with-academic-communities

OpenAI GitHub: https://github.com/openai/math/tree/main/preprints/The-Exponential-Time-Hypothesis-is-False-October-25-2026

Anthropic Hexagon: https://hexagonmath.org/2610.00577v1

Anthropic Lean: https://github.com/anthropics/formal-math/tree/16a5ce41304fcdaea4e0be7c132e1c361dc26da5/eth

Paper: https://drive.google.com/drive/u/0/folders/1SAqEO1kt6wnA0NXuZecyPhdBgzW3Z0y1

# The Exponential Time Hypothesis Is False? A Breakthrough in 3-SAT Algorithms


## 1. Introduction

In October 2026, OpenAI and Anthropic independently presented new algorithms that challenge one of the most influential conjectures in theoretical computer science: the **Exponential Time Hypothesis (ETH)**.

ETH states that solving the Boolean satisfiability problem known as **3-SAT** requires exponential time in the worst case.

The new research proposes deterministic algorithms that solve 3-SAT in **subexponential time**, potentially refuting ETH.

More importantly, these algorithms solve not only 3-SAT but also **#3-SAT**, which counts the exact number of satisfying assignments.


---

## 2. What Is the 3-SAT Problem?

The Boolean Satisfiability Problem (SAT) asks whether there exists an assignment of Boolean values that makes a logical formula evaluate to true.

In **3-SAT**, the formula is written in Conjunctive Normal Form (CNF), where each clause contains at most three literals.

A general 3-CNF formula can be expressed as:

$$
F(x_1,\ldots,x_n)=\bigwedge_{i=1}^{m}C_i
$$

where each clause $C_i$ contains up to three literals.

### 2.1. Example of 3-SAT

Consider the following formula:

$$
\begin{aligned}
F = &(x_1 \lor \neg x_2 \lor x_3)\\
&\land(\neg x_1 \lor x_2 \lor x_4)\\
&\land(x_1 \lor \neg x_3 \lor \neg x_4)
\end{aligned}
$$

Here:

- $x_i$: Boolean variable (True or False)
- $\lor$: Logical OR
- $\land$: Logical AND
- $\neg$: Logical NOT

The formula contains three clauses:

$$
C_1=x_1\lor\neg x_2\lor x_3
$$

$$
C_2=\neg x_1\lor x_2\lor x_4
$$

$$
C_3=x_1\lor\neg x_3\lor\neg x_4
$$

Since the clauses are connected by AND operations, **every clause must evaluate to True simultaneously**.

For example, consider the assignment:

$$
(x_1,x_2,x_3,x_4)=(1,1,1,0)
$$

Substituting these values:

$$
C_1=1\lor0\lor1=1
$$

$$
C_2=0\lor1\lor0=1
$$

$$
C_3=1\lor0\lor1=1
$$

Therefore:

$$
F=C_1\land C_2\land C_3=1
$$

This assignment satisfies the formula.

### 2.2. Why Is 3-SAT Difficult?

For $n$ Boolean variables, there are:

$$
2^n
$$

possible assignments.

A brute-force algorithm can enumerate every assignment and check whether the formula is satisfied.

Its running time is approximately:

$$
T(n)=O(2^n\operatorname{poly}(n,m))
$$

for $n$ Boolean variables and $m$ clauses.

The number of combinations increases exponentially as the number of variables grows.

However, verifying whether a given assignment satisfies the formula can be done in polynomial time.

This distinction makes 3-SAT a fundamental **NP-complete problem**.

### 2.3. 3-SAT vs. #3-SAT

There are two closely related problems:

**3-SAT (Decision Problem)**

Determine whether at least one satisfying assignment exists.

$$
\exists x\in\{0,1\}^n:F(x)=1
$$

**#3-SAT (Counting Problem)**

Compute the exact number of satisfying assignments.

$$
\#F=\left|\{x\in\{0,1\}^n:F(x)=1\}\right|
$$

The relationship is straightforward:

$$
F\text{ is satisfiable}\iff\#F>0
$$

Therefore, an algorithm that solves #3-SAT exactly can also solve 3-SAT.

While 3-SAT is NP-complete, #3-SAT is #P-complete.

---

## 3. What Is the Exponential Time Hypothesis (ETH)?

The **Exponential Time Hypothesis (ETH)** was introduced by Russell Impagliazzo and Ramamohan Paturi in 2001.

ETH conjectures that **3-SAT cannot be solved in subexponential time with respect to the number of variables**.

It is commonly expressed as:

$$
\boxed{\text{3-SAT}\notin\mathrm{DTIME}(2^{o(n)})}
$$

More explicitly, in the standard input-size formulation, ETH rules out algorithms running in:

$$
2^{o(n)}(n+m)^{O(1)}
$$

where:

- $n$: Number of Boolean variables
- $m$: Number of clauses
- $o(n)$: A function that grows asymptotically slower than $n$
- $O(n)$: A function whose growth is asymptotically bounded above by a constant multiple of \(n\).

### 3.1. Exponential Time vs. Subexponential Time

Consider the following running times:

| Time Complexity | Classification | Compatible with ETH? |
|---|---|---|
| $2^n$ | Exponential | Yes |
| $2^{0.5n}$ | Exponential | Yes |
| $2^{0.01n}$ | Exponential | Yes |
| $2^{n/\log n}$ | Subexponential | No |
| $2^{\sqrt n}$ | Subexponential | No |
| $2^{n^{0.9}}$ | Subexponential | No |
| $n^{100}$ | Polynomial | No |

Here, "No" means that an algorithm with the stated worst-case bound for arbitrary 3-SAT instances would contradict ETH.

For example:

$$
T(n)=2^{n/\log n}
$$

Since:

$$
\lim_{n\rightarrow\infty}\frac{n/\log n}{n}
=
\lim_{n\rightarrow\infty}\frac{1}{\log n}
=0
$$

we have:

$$
T(n)=2^{o(n)}
$$

Thus, an algorithm with this running time would refute ETH.

ETH does not claim that every assignment must be examined. Rather, it conjectures that the exponent of the worst-case running time cannot be reduced to a sublinear function of $n$.

### 3.2. What Is SETH?

The **Strong Exponential Time Hypothesis (SETH)** is a stronger conjecture concerning the general $k$-SAT problem.

Let $s_k$ denote the infimum of constants $c$ for which $k$-SAT can be solved in time $O^*(2^{cn})$.

Then:

$$
\text{ETH}:s_3>0
$$

$$
\text{SETH}:\lim_{k\rightarrow\infty}s_k=1
$$

ETH claims that 3-SAT requires a positive constant in its exponential-time exponent.

SETH makes the stronger claim that, as the clause width $k$ increases, the best possible exponent approaches $n$.

Since SETH implies ETH, refuting ETH also refutes SETH.

---

## 4. New Subexponential-Time Algorithms for 3-SAT

In October 2026, OpenAI and Anthropic presented independent approaches to solving #3-SAT in subexponential time.

The three manuscripts discussed here are:

| Research Group | Paper | Main Technique |
|---|---|---|
| OpenAI | The Exponential Time Hypothesis is False | Polynomial interpolation, linear substitutions, tensor contraction |
| OpenAI | Exactly Counting 3-SAT Solutions in Subexponential Time | Linear substitutions, group factorization, Hermite interpolation |
| Anthropic | A $2^{o(n)}$ Time Algorithm for 3-SAT | Tensor networks, eight-square identity, recursive decomposition |

### 4.1. OpenAI: Exactly Counting 3-SAT Solutions in Subexponential Time
The first OpenAI manuscript, Exactly Counting 3-SAT Solutions in Subexponential Time, dated October 17, 2026, proposes a deterministic algorithm for computing the exact number of satisfying assignments of arbitrary 3-CNF formulas.

Its main theoretical result is:

$$
\boxed{
T(n,m)=(n+m)^{O(1)}
2^{O\left(\frac{n(\log\log n)^2}{(\log n)^{1/6}}\right)}
}
$$

The algorithm uses polynomial space.

Since

$$
\lim_{n\to\infty}
\frac{(\log\log n)^2}{(\log n)^{1/6}}=0,
$$

the running time is subexponential in the number of variables.

### 4.2. OpenAI: The Exponential Time Hypothesis Is False
The second OpenAI manuscript, The Exponential Time Hypothesis is False, was initially dated October 17 and revised on October 25, 2026.

It develops the preceding approach further and extends the result to general fixed-width $k$-SAT problems.

Its main theorem establishes the following claimed running time for exact #k-SAT counting:

$$
\boxed{
T(n,m)=(n+m)^{O_k(1)}
2^{O_k\left(
\frac{n\log\log n}{\sqrt{\log n}}
\right)}
}
$$

The algorithm also uses polynomial space.

The principal additional techniques include:
- Sparse polynomial equations: Reformulating satisfiability conditions as systems of low-degree polynomial equations over $\mathbb{F}_2$.
- Parity tests: Constructing constraints that detect assignments violating the original clauses.
- Polynomial interpolation: Recovering exact counts from efficiently evaluated algebraic expressions.
- Tensor contraction: Improving the running-time bound by performing structured algebraic computations.
The main improvement is that the algorithm achieves a sharper asymptotic exponent while supporting every fixed $k\geq2$.

Although the two OpenAI manuscripts share the same general algebraic framework, the longer manuscript introduces substantial additional constructions.

### 4.2. Anthropic: Tensor Networks and the Eight-Square Identity

Anthropic's algorithm uses an alternative algebraic approach based on tensor networks.

The main idea is to represent a 3-SAT formula as a network of local constraints and consistency conditions.

In this representation:

- Each node represents a local set of constraints.
- Edges encode consistency between shared Boolean variables.
- The solution count is represented as a sum of products over the network.

A simplified form is:

$$
Z=
\sum_z
\prod_a W_a(z_a)
\prod_{\{a,b\}}\Psi_{a,b}(z_a,z_b)
$$

where:

- $W_a$ represents the local constraint function.
- $\Psi_{a,b}$ enforces consistency between nodes.
- $Z$ is the number of satisfying assignments.

The difficulty is that the graph may contain many dependencies between its nodes.

Simply partitioning the graph does not guarantee subexponential-time evaluation.

To address this, the algorithm uses an algebraic decomposition derived from the **eight-square identity**.

For three-bit strings, the paper establishes a rewriting identity of the form:

$$
\begin{aligned}
2[x=v][y=u]
={}&[x\oplus y\oplus u\oplus v=0]\\
&\cdot\big(
\sigma(x,y)\sigma(v,u)
+\sigma(x,u)\sigma(v,y)
\big)
\end{aligned}
$$

This identity rewrites crossing equality constraints into a sum of functions with more favorable factorization properties.

The algorithm then:

1. Represents #3-SAT as a tensor-network-style computation.
2. Rewrites crossing constraints using algebraic identities.
3. Groups nodes by rows or columns.
4. Reduces the information exchanged between groups.
5. Recursively processes smaller networks.
6. Combines the results to recover the exact solution count.


For fixed $k$, the manuscript claims:

$$
\boxed{
T(n)=
2^{O\left(\frac{n\log\log n}{\sqrt{\log n}}\right)}
}
$$

which is also subexponential in $n$.

---

## 5. Comparison of the Proposed Algorithms


| Property | OpenAI (17 Pages) | OpenAI (67 Pages) | Anthropic (49 Pages) |
|---|---|---|---|
| Paper | Exactly Counting 3-SAT Solutions in Subexponential Time | The Exponential Time Hypothesis is False | A $2^{o(n)}$ Time Algorithm for 3-SAT |
| Main problem | #3-SAT | #k-SAT | #k-SAT |
| Algorithm type | Deterministic | Deterministic | Deterministic |
| Main representation | Polynomial counting expressions | Sparse polynomial equations and polynomial counting expressions | Tensor networks and sum-product expressions |
| Key method | Linear substitutions, group factorization, and Hermite interpolation | Linear substitutions, parity tests, interpolation, and tensor contraction | Algebraic decomposition and recursive network reduction |


### Time Complexity Comparison

For fixed $k$, the proposed algorithms have the following asymptotic running-time bounds:

**OpenAI (17 Pages)**

$$
T(n,m)=(n+m)^{O(1)}
2^{O\left(\frac{n(\log\log n)^2}{(\log n)^{1/6}}\right)}
$$

**OpenAI (67 Pages)**

$$
T(n,m)=(n+m)^{O_k(1)}
2^{O_k\left(\frac{n\log\log n}{\sqrt{\log n}}\right)}
$$

**Anthropic (49 Pages)**

$$
T(n)=2^{O_k\left(\frac{n\log\log n}{\sqrt{\log n}}\right)}
$$

The two OpenAI papers share the same fundamental algebraic framework, but the longer manuscript extends the method to general fixed-width $k$-SAT and provides a stronger asymptotic bound.

Anthropic takes a different approach by rewriting network interactions using algebraic identities and recursively reducing the number of shared bits between groups.

Although the stronger OpenAI manuscript and Anthropic's manuscript have similar asymptotic exponents, their mathematical techniques are substantially different.

The similarity of the theoretical bounds does not imply that their actual running times or implementations are equivalent.

All three manuscripts propose exact-counting algorithms with subexponential running time, and each would independently refute ETH if its results are valid.

---

## 6. Does This Prove P = NP?

No.

The Exponential Time Hypothesis and the P vs. NP problem address different questions.

ETH concerns whether 3-SAT necessarily requires exponential time:

$$
T(n)=2^{\Omega(n)}
$$

P vs. NP concerns whether 3-SAT can be solved in polynomial time:

$$
T(n)=n^{O(1)}
$$

The proposed algorithms instead achieve:

$$
T(n)=2^{o(n)}
$$

Subexponential time is still potentially much larger than polynomial time.

Therefore:

$$
P=NP\implies\text{ETH is false}
$$

However, the converse is not known to hold.

It remains possible that:

$$
P\neq NP
$$

even if ETH and SETH are false.

Consequently, the new algorithms do not resolve the P vs. NP problem.

### 6.1 And then what is P vs NP problem?
The P vs. NP problem is one of the most fundamental unsolved problems in theoretical computer science. It asks whether every problem whose solution can be efficiently verified can also be efficiently solved.

To understand this question, we first need to distinguish between two complexity classes: P and NP.

#### P (Polynomial Time)

P is the class of decision problems that can be solved by a deterministic algorithm in polynomial time.

An algorithm runs in polynomial time if its time complexity is bounded by (O(n^k)), where (n) is the input size and (k) is a fixed constant.

#### NP (Nondeterministic Polynomial Time)

NP is the class of decision problems for which a proposed solution (also called a certificate or witness) can be verified in polynomial time.

A problem may be difficult to solve, but if a candidate solution is provided, checking whether it is correct can be computationally efficient.

#### The Relationship Between P and NP

Every problem in P also belongs to NP because a solution that can be found in polynomial time can also be verified in polynomial time.

Therefore:

$$
\boxed{P\subseteq NP}
$$
However, whether the reverse inclusion holds remains unknown.

The central question is:

$$
\boxed{P\overset{?}{=}NP}
$$

There are two possibilities:

1. If P ≠ NP

Some problems have solutions that can be verified in polynomial time but cannot be solved in polynomial time.

This would establish a fundamental distinction between efficiently finding solutions and efficiently verifying them.

2. If P = NP

Every problem whose solution can be verified in polynomial time could also be solved in polynomial time.

For example, NP-complete problems such as SAT and 3-SAT would have polynomial-time algorithms.

Instead of relying on exponential-time approaches such as $O(2^n)$, these problems could theoretically be solved in $O(n^k)$ time for some fixed constant (k).

As of now, neither P = NP nor P ≠ NP has been proven. Although most complexity theorists believe that P ≠ NP, the question remains open.

#### What if P=NP?

If P = NP were proven, polynomial-time algorithms would exist for all problems in NP, including NP-complete problems.

This would also have profound consequences for cryptography. In particular, cryptographic security based on the existence of computationally hard one-way functions would no longer be possible in the usual asymptotic sense.

Problems that currently appear to require exponential or subexponential computational effort could, in principle, become solvable in polynomial time.

However, two distinctions are important:

P = NP does not imply that every exponential-time algorithm automatically becomes polynomial-time. Rather, it establishes the existence of polynomial-time algorithms for problems in NP.

Polynomial time does not necessarily mean practical efficiency. An algorithm with complexity $O(n^{100})$, for example, is polynomial-time but may still be computationally infeasible.

Therefore, proving P = NP would fundamentally change our understanding of computational complexity and undermine the theoretical foundations of much of modern computational cryptography, even if the discovered polynomial-time algorithms were not immediately practical.

---



## 7. Why Does This Matter?

ETH and SETH have been widely used to establish conditional lower bounds in theoretical computer science.

Many results in fine-grained complexity assume that certain problems cannot be solved faster unless ETH or SETH fails.

Examples include:

- Constraint satisfaction problems
- Independent Set and Clique
- Graph coloring
- Parameterized graph algorithms
- Edit distance and longest common subsequence
- Certain combinatorial counting problems

A valid refutation of ETH would require researchers to revisit lower bounds that rely exclusively on this hypothesis.

However, it would not automatically provide faster practical algorithms for all these problems.

Some conditional lower bounds also rely on alternative assumptions, such as the Orthogonal Vectors Conjecture, which are not directly disproved by the proposed results.

### 7.1. Theoretical Complexity vs. Practical Performance

An important limitation is that the improvements are asymptotic.

Although:

$$
2^{o(n)}
$$

eventually grows more slowly than:

$$
2^{cn}
$$

for every fixed $c>0$, the hidden constants in the new algorithms may be extremely large.

Therefore, the algorithms may not outperform conventional SAT solvers for realistic input sizes.

The Anthropic manuscript explicitly emphasizes this limitation, noting that the proved bounds do not lead to practical improvements for ordinary problem sizes.

The primary contribution is therefore to computational complexity theory rather than immediate SAT-solving performance.

---

## 8. Conclusion

The October 2026 manuscripts from OpenAI and Anthropic propose deterministic subexponential-time algorithms for solving 3-SAT and counting its satisfying assignments exactly.

Their main claim is:

$$
\boxed{
\text{3-SAT can be solved in }2^{o(n)}\text{ time}
}
$$

If established, this result directly contradicts the Exponential Time Hypothesis and also refutes the Strong Exponential Time Hypothesis.

The two research groups approach the problem differently:

- **OpenAI:** Polynomial interpolation, linear substitutions, algebraic counting, and tensor contraction.
- **Anthropic:** Tensor networks, algebraic identities, and recursive complexity reduction.

These algorithms do not establish P = NP and currently offer no demonstrated practical advantage over conventional SAT solvers.

Nevertheless, their proposed results represent a potentially significant theoretical breakthrough.

Beyond the implications for ETH, the work also highlights the potential of AI-assisted research in discovering new mathematical constructions and algorithms.

**The central takeaway is that an exponential-time barrier long regarded as fundamental may not actually exist for 3-SAT.**

The final scientific significance of these claims depends on rigorous independent verification.

---

## References

1. OpenAI. *The Exponential Time Hypothesis is False*. October 17, revised October 25, 2026.
2. OpenAI. *Exactly Counting 3-SAT Solutions in Subexponential Time*. October 17, 2026.
3. Anthropic. *A $2^{o(n)}$ Time Algorithm for 3-SAT*. October 26, 2026.
4. Impagliazzo, R., & Paturi, R. (2001). *On the Complexity of k-SAT*. Journal of Computer and System Sciences, 62(2), 367–375.
5. Impagliazzo, R., Paturi, R., & Zane, F. (2001). *Which Problems Have Strongly Exponential Complexity?* Journal of Computer and System Sciences, 63(4), 512–530.

## Apendix
## A. Details About The New Algorithms Proposed in October 2026

In October 2026, OpenAI and Anthropic presented three manuscripts proposing deterministic subexponential-time algorithms for counting the satisfying assignments of Boolean formulas.

The three manuscripts discussed in this section are:

1. **OpenAI:** *Exactly Counting 3-SAT Solutions in Subexponential Time* (October 17, 2026).
2. **OpenAI:** *The Exponential Time Hypothesis is False* (October 17, revised October 25, 2026).
3. **Anthropic:** *A $2^{o(n)}$ Time Algorithm for 3-SAT* (October 26, 2026).

The two OpenAI manuscripts share the same fundamental algebraic framework, while Anthropic develops a substantially different approach based on tensor networks and recursive algebraic decomposition.

Before examining these algorithms individually, we first introduce the mathematical representation of #3-SAT used in the OpenAI manuscripts.

### A.1. Mathematical Foundations: Representing 3-SAT as a Polynomial

The first step is to transform a Boolean satisfiability problem into an algebraic counting problem.

#### Step 1: Polynomial Representation of Clauses

Consider a 3-CNF formula:

$$
F(x_1,\ldots,x_n)=\bigwedge_{e=1}^{m}C_e
$$

where:

- $F$: The entire Boolean formula.
- $n$: Number of Boolean variables.
- $m$: Number of clauses.
- $e$: Index identifying a clause.
- $C_e$: The $e$-th clause.

Each clause contains at most three literals.

For every clause $e$, define a polynomial:

$$
\boxed{
\Phi_e(X)=1-\prod_{v\in I_e}(1-L_{e,v}(X_v))
}
$$

Here:

- $X=(X_1,\ldots,X_n)$: Vector of formal polynomial variables.
- $v$: Index of a Boolean variable.
- $I_e$: Set of variable indices appearing in clause $e$.
- $X_v$: The $v$-th formal variable.
- $L_{e,v}(X_v)$: Polynomial representing the literal involving variable $v$ in clause $e$.
- $\Phi_e(X)$: Polynomial representing whether clause $e$ is satisfied.
- $\prod$: Product over the specified indices.

The literal polynomial is defined as:

$$
L_{e,v}(X_v)=
\begin{cases}
X_v, & \text{if the literal is positive},\\
1-X_v, & \text{if the literal is negative}.
\end{cases}
$$

For Boolean inputs, $\Phi_e(X)$ evaluates to 1 if the clause is satisfied and 0 otherwise.

**Example**

Consider the clause:

$$
C_1=(x_1\lor\neg x_2\lor x_3)
$$

The variables appearing in this clause are:

$$
I_1=\{1,2,3\}
$$

The corresponding literal polynomials are:

$$
L_{1,1}(X_1)=X_1
$$

$$
L_{1,2}(X_2)=1-X_2
$$

$$
L_{1,3}(X_3)=X_3
$$

Substituting them into the satisfaction polynomial gives:

$$
\begin{aligned}
\Phi_1(X)
&=1-(1-X_1)(1-(1-X_2))(1-X_3)\\
&=1-(1-X_1)X_2(1-X_3).
\end{aligned}
$$

For the assignment:

$$
(x_1,x_2,x_3)=(0,1,0)
$$

we obtain:

$$
\Phi_1(x)=1-(1)(1)(1)=0
$$

The clause is not satisfied.

In contrast, if:

$$
(x_1,x_2,x_3)=(1,1,0)
$$

then:

$$
\Phi_1(x)=1-(0)(1)(1)=1
$$

The clause is satisfied.

Thus, the polynomial provides an algebraic representation of the logical OR operation.

#### Step 2: Counting All Satisfying Assignments

Once each clause has been converted into a polynomial, the exact number of satisfying assignments can be expressed as:

$$
\boxed{
\#F=
\sum_{x\in\{0,1\}^n}
\prod_{e=1}^{m}\Phi_e(x)
}
$$

where:

- $\#F$: Exact number of satisfying assignments of $F$.
- $x=(x_1,\ldots,x_n)$: A Boolean assignment vector.
- $\{0,1\}^n$: Set of all $2^n$ Boolean assignments.
- $\sum$: Summation over every possible assignment.
- $\prod$: Multiplication of the satisfaction indicators of all clauses.

For example, if the formula contains three clauses:

$$
F=C_1\land C_2\land C_3
$$

then:

$$
\prod_{e=1}^{3}\Phi_e(x)
=
\Phi_1(x)\Phi_2(x)\Phi_3(x)
$$

Since each polynomial evaluates to either 0 or 1 on Boolean inputs, the product equals 1 only when all three clauses are satisfied simultaneously.

Summing over all assignments therefore produces the exact number of satisfying assignments.

However, direct evaluation still requires enumerating all $2^n$ assignments.

The goal of the new algorithms is to evaluate this sum more efficiently without explicitly testing every global assignment.

#### Step 3: Linear Substitution

The OpenAI approach introduces a polynomial parameter $T$ and replaces each input variable with a carefully constructed linear form.

For every occurrence of variable $v$ in clause $e$, define:

$$
\boxed{
P_{e,v}(T)x
}
$$

where:

- $T$: Polynomial interpolation parameter.
- $P_{e,v}(T)$: A row vector whose entries are polynomials in $T$.
- $x$: Boolean assignment vector.
- $P_{e,v}(T)x$: Scalar resulting from the multiplication of the row vector and assignment vector.

More precisely:

$$
P_{e,v}(T)\in\mathbb{Q}[T]^{1\times n}
$$

and:

$$
x\in\{0,1\}^{n\times1}
$$

Therefore:

$$
P_{e,v}(T)x
=
\sum_{j=1}^{n}p_{e,v,j}(T)x_j
$$

where $p_{e,v,j}(T)$ is the $j$-th polynomial coefficient of $P_{e,v}(T)$.

**Example**

Suppose there are four variables, and consider the following illustrative substitution:

$$
P_{e,1}(T)=
\begin{bmatrix}
1&T&0&-T
\end{bmatrix}
$$

Then:

$$
P_{e,1}(T)x
=
x_1+Tx_2-Tx_4
$$

Instead of using only $x_1$, the transformed clause now receives a linear combination of multiple variables.

This is only an illustrative example; the actual paper constructs the substitution vectors using additional algebraic conditions.

The substitutions must satisfy:

$$
\boxed{
P_{e,v}(0)=\delta_v
}
$$

where $\delta_v$ is the row vector whose $v$-th element equals 1 and all other elements equal 0.

For example:

$$
\delta_1=
\begin{bmatrix}
1&0&0&0
\end{bmatrix}
$$

Therefore:

$$
P_{e,v}(0)x=x_v
$$

This condition guarantees that the original variable is recovered when $T=0$.

Notice that when $T\neq0$, the substituted values are not necessarily Boolean. They may be arbitrary rational values, so the transformed clause polynomials no longer necessarily represent logical satisfaction indicators.

#### Step 4: Constructing the Polynomial $H(T)$

After applying the linear substitutions, the original counting expression becomes:

$$
\boxed{
H(T)=
\sum_{x\in\{0,1\}^n}
\prod_{e=1}^{m}
\Phi_e\left(
(P_{e,v}(T)x)_{v\in I_e}
\right)
}
$$

The notation:

$$
(P_{e,v}(T)x)_{v\in I_e}
$$

represents the collection of substituted inputs for all variables appearing in clause $e$.

For example, if:

$$
I_e=\{1,2,3\}
$$

then:

$$
(P_{e,v}(T)x)_{v\in I_e}
=
\left(
P_{e,1}(T)x,
P_{e,2}(T)x,
P_{e,3}(T)x
\right)
$$

Because every substitution satisfies:

$$
P_{e,v}(0)x=x_v
$$

setting $T=0$ restores the original counting expression:

$$
\begin{aligned}
H(0)
&=
\sum_{x\in\{0,1\}^n}
\prod_{e=1}^{m}
\Phi_e((x_v)_{v\in I_e})\\
&=
\sum_{x\in\{0,1\}^n}
\prod_{e=1}^{m}\Phi_e(x)\\
&=\#F.
\end{aligned}
$$

Thus:

$$
\boxed{H(0)=\#F}
$$

The key idea is to avoid computing $H(0)$ directly.

Instead, the algorithm constructs substitutions that make $H(T)$ easier to evaluate at selected nonzero values of $T$.

Information collected at those points is then used to reconstruct $H(0)$ through polynomial interpolation.

This mathematical framework forms the foundation of both OpenAI manuscripts.

---

### A.2. OpenAI: Exactly Counting 3-SAT Solutions in Subexponential Time

The first OpenAI manuscript, *Exactly Counting 3-SAT Solutions in Subexponential Time*, proposes a deterministic algorithm that counts the satisfying assignments of an arbitrary 3-CNF formula in subexponential time.

Its main theorem establishes the following running-time bound:

$$
\boxed{
T(n,m)=
(n+m)^{O(1)}
2^{O\left(
\frac{n(\log\log n)^2}{(\log n)^{1/6}}
\right)}
}
$$

The algorithm uses polynomial space.

Since:

$$
\lim_{n\to\infty}
\frac{(\log\log n)^2}{(\log n)^{1/6}}=0,
$$

the exponent grows asymptotically slower than $n$.

The algorithm therefore runs in subexponential time with respect to the number of variables, contradicting ETH if correct.

The algorithm consists of several major stages.

#### A.2.1. Sparsification and Variable Copying

The algorithm first transforms the original 3-CNF formula into a collection of formulas in which each variable appears only a limited number of times.

This is achieved through two standard reductions.

**Counting-Preserving Sparsification**

The sparsification lemma partitions the satisfying assignments of the original formula among multiple sparse formulas.

The transformation preserves the total solution count, which can be recovered by combining the counts of the resulting formulas.

**Variable Copying**

Variables that appear in many clauses are replaced by separate copies.

Additional constraints ensure that these copies always have the same Boolean value.

For example, if variable $x_1$ appears three times, it can be replaced by $x_{1,1}$, $x_{1,2}$, and $x_{1,3}$.

An implication cycle enforces equality:

$$
(\neg x_{1,1}\lor x_{1,2})
\land
(\neg x_{1,2}\lor x_{1,3})
\land
(\neg x_{1,3}\lor x_{1,1})
$$

These clauses ensure that all copies have identical values.

After the transformation, each variable appears in at most three clauses.

This restriction gives a useful bound on the total degree of the polynomial representation.

#### A.2.2. Partitioning Variables into Smaller Groups

After constructing the polynomial $H(T)$, the algorithm partitions the variables into $r$ groups:

$$
\{1,\ldots,n\}
=
S_1\sqcup S_2\sqcup\cdots\sqcup S_r
$$

where:

- $r$: Number of variable groups.
- $S_g$: Set of variable indices assigned to group $g$.
- $\sqcup$: Disjoint union.

The groups are constructed so that each contains at most approximately $2n/r$ variables.

If each transformed clause depended only on variables within a single group, the entire counting expression could be factorized into smaller sums.

However, clauses generally share variables across different groups.

To overcome this difficulty, the algorithm introduces a limited number of shared linear measurements.

For group $g$, these measurements are expressed as:

$$
y_g=C_gx_g
$$

where:

- $x_g$: Boolean assignment restricted to group $S_g$.
- $C_g$: Integer measurement matrix associated with the group.
- $y_g$: Vector of measurement values.

The substitutions are constructed so that, once the shared measurement values are fixed, each transformed clause depends only on the variables in its assigned group.

The overall sum can then be evaluated through smaller independent computations.

#### A.2.3. Factoring Boolean Sums

Suppose the transformed expression can be separated into functions $A_g$, one for each group.

After fixing the measurement vector $y$, the counting expression takes the form:

$$
H(t)=
\sum_y
\prod_{g=1}^{r}
\left(
\sum_{\substack{x_g\in\{0,1\}^{|S_g|}\\C_gx_g=y_g}}
A_g(x_g;y)
\right)
$$

where:

- $t$: A selected nonzero value of the interpolation parameter $T$.
- $y$: Collection of measurement values across all groups.
- $A_g$: Product of transformed clause factors assigned to group $g$.
- $x_g$: Boolean assignment within group $g$.

Instead of enumerating all $2^n$ global assignments together, the algorithm enumerates assignments within individual groups.

This is the main mechanism that reduces the computational cost.

#### A.2.4. Recovering the Count Using Hermite Interpolation

Although the algorithm can efficiently evaluate the transformed expression at selected nonzero values of $T$, the desired answer is:

$$
H(0)=\#F
$$

The algorithm therefore reconstructs $H(T)$ using Taylor coefficients.

For a selected interpolation point $t$, consider:

$$
H(t+z)=
\sum_{j=0}^{\deg H}
\frac{H^{(j)}(t)}{j!}z^j
$$

where:

- $t$: Selected interpolation point.
- $z$: Formal variable representing a displacement from $t$.
- $H^{(j)}(t)$: The $j$-th derivative of $H$ at $t$.
- $\deg H$: Degree of the polynomial $H$.

The algorithm computes sufficiently many Taylor coefficients at several interpolation points.

It then applies **Hermite interpolation** to reconstruct $H(T)$ and recover its constant coefficient.

The resulting value is the exact number of satisfying assignments.

The key contribution is the combination of low-degree linear substitutions, group-wise Boolean summation, and polynomial interpolation to obtain a deterministic subexponential-time exact-counting algorithm.

---

### A.3. OpenAI: The Exponential Time Hypothesis Is False

The second OpenAI manuscript, *The Exponential Time Hypothesis is False*, extends and strengthens the preceding approach.

Rather than focusing only on 3-SAT, the longer manuscript establishes results for general fixed-width $k$-SAT problems.

It retains the same basic idea:

1. Express satisfiability as an algebraic counting problem.
2. Construct carefully designed linear substitutions.
3. Separate Boolean sums through shared measurements.
4. Recover the exact count using polynomial interpolation.

However, the longer manuscript introduces additional techniques to obtain a sharper asymptotic running-time bound.

#### A.3.1. Sparse Polynomial Equations

The algorithm reformulates the counting problem using systems of low-degree polynomial equations over the finite field $\mathbb{F}_2$.

Here:

$$
\mathbb{F}_2=\{0,1\}
$$

with addition and multiplication performed modulo 2.

The objective is to count Boolean assignments satisfying a system:

$$
P_1(x)=P_2(x)=\cdots=P_s(x)=0
$$

where each $P_i$ is a sparse polynomial.

The construction ensures that the common zeros of the polynomial system correspond to the satisfying assignments of the original formula.

This reformulation makes it possible to work with a controlled number of algebraic constraints.

#### A.3.2. Parity Tests

The algorithm introduces parity tests that detect assignments violating the original clauses.

For each clause $e$, define a violation indicator:

$$
v_e(x)=
\begin{cases}
1, & \text{if clause }e\text{ is violated},\\
0, & \text{otherwise}.
\end{cases}
$$

A parity test over a subset $S$ of clauses checks:

$$
\sum_{e\in S}v_e(x)=0\pmod 2
$$

A satisfying assignment passes every such test because it violates no clauses.

The algorithm constructs a family of tests that also excludes every nonsatisfying assignment.

Consequently, counting assignments that pass all the tests is equivalent to counting the satisfying assignments of the original formula.

#### A.3.3. Tensor Contraction and Improved Complexity

The longer manuscript also develops more efficient algebraic reconstruction through tensor contraction.

Tensor contraction evaluates sums over shared indices, allowing structured products and summations to be combined without explicitly materializing all combinations.

In this context, the method improves the cost of recovering the desired polynomial coefficients while preserving exact counting.

The resulting main theorem gives, for every fixed $k\geq2$:

$$
\boxed{
T(n,m)=
(n+m)^{O_k(1)}
2^{O_k\left(
\frac{n\log\log n}{\sqrt{\log n}}
\right)}
}
$$

with polynomial space.

Since:

$$
\lim_{n\to\infty}
\frac{\log\log n}{\sqrt{\log n}}=0,
$$

the running time is subexponential in $n$.

Compared with the 17-page manuscript, the longer paper provides a stronger asymptotic exponent and extends the algorithm to #k-SAT for every fixed $k\geq2$.

---

### A.4. Anthropic: Tensor Networks and Algebraic Decomposition

Anthropic's paper takes a different route by transforming #3-SAT into a sum-product computation over a graph, closely related to tensor network contraction.

In this representation:

- Nodes represent local constraints or groups of clauses.
- Edges enforce consistency between shared Boolean variables.
- The final answer is obtained by summing products of local functions over compatible assignments.

The number of satisfying assignments is represented as:

$$
Z=
\sum_z
\prod_a W_a(z_a)
\prod_{\{a,b\}}\Psi_{a,b}(z_a,z_b)
$$

where:

- $Z$: Exact number of satisfying assignments.
- $z$: Collection of local variable assignments.
- $a,b$: Indices of graph nodes.
- $z_a$: Local assignment associated with node $a$.
- $W_a(z_a)$: Local constraint function at node $a$.
- $\Psi_{a,b}(z_a,z_b)$: Consistency function between nodes $a$ and $b$.
- $\prod_{\{a,b\}}$: Product over node pairs connected by consistency constraints.

The main challenge is that dependencies between different parts of the graph can be expensive to evaluate.

#### A.4.1. Algebraic Decomposition Using the Eight-Square Identity

Anthropic's approach introduces an algebraic decomposition based on the classical **eight-square identity**, associated with the multiplicative norm of the octonions.

This identity makes it possible to rewrite pairs of crossing equality constraints as sums of other functions with more favorable factorization properties.

For three-bit strings $x,y,u,v$, the paper derives the identity:

$$
\begin{aligned}
2[x=v][y=u]
={}&[x\oplus y\oplus u\oplus v=0]\\
&\cdot\big(
\sigma(x,y)\sigma(v,u)
+\sigma(x,u)\sigma(v,y)
\big)
\end{aligned}
$$

where:

- $[A]$: Indicator function equal to 1 when condition $A$ is true and 0 otherwise.
- $\oplus$: Bitwise XOR operation.
- $\sigma$: A specially constructed sign function taking values in $\{-1,1\}$.
- $x,y,u,v$: Three-bit Boolean strings.

The identity rewrites a product of two crossing equality constraints as a sum of algebraic terms.

These terms can be factorized more efficiently after grouping network nodes.

#### A.4.2. Recursive Network Decomposition

The algorithm then groups nodes, reduces the number of bits that must be exchanged between groups, and recursively evaluates the transformed problem.

Its main operations include:

1. Representing the original Boolean formula as a network of local constraints.
2. Rewriting crossing equality constraints using algebraic identities.
3. Merging nodes along rows or columns.
4. Reducing the number of shared bits between the resulting groups.
5. Recursively processing the smaller networks.
6. Combining the results to recover the exact number of satisfying assignments.

The key idea is that an algebraic transformation can make the dependencies between groups less expensive to evaluate, even though the exact value of the original counting expression remains unchanged.

#### A.4.3. Running-Time Complexity

In the simplified construction, a key reduction factor is:

$$
\rho=\frac{11}{12}
$$

The stronger construction in the appendix uses additional algebraic techniques to improve the asymptotic bound.

For fixed $k$, the manuscript claims a running time of:

$$
\boxed{
T(n)=
2^{O_k\left(
\frac{n\log\log n}{\sqrt{\log n}}
\right)}
}
$$

The result is subexponential in $n$, contradicting ETH if correct.

One particularly interesting aspect is the reported research process: the manuscript states that the core algorithm was discovered by an internal Claude research model before researchers helped refine its presentation.