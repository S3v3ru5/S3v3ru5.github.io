---
title: ZKHack I Puzzle - Soundness of Music Writeup
date: 2025-12-24
tags: [ZK, writeup]
slug: zkhack-I-soundness-of-music
---

# ZKHack I Puzzle: Soundness of Music Writeup

> TL;DR
> The puzzle presents a zkSNARK proof system for verifying `4 * x == y`. The challenge is to forge a proof for the invalid statement `4 * 1 == 1`.
>
> The vulnerability: the trusted setup incorrectly includes alpha-shifted commitments `[α·ρ·L_i(τ)]₁` for public input polynomials. These values are never used by honest provers or verifiers, but their presence allows an attacker to adjust an existing valid proof to work with different public inputs. By adding computed deltas to both the input commitment and its alpha-shifted pair, the attacker can cancel out the verifier's public input contribution while maintaining all knowledge-of-exponent checks, effectively converting any valid proof into a proof for arbitrary public inputs.

**Note:** This writeup describes the vulnerability from the [BCTV security paper](https://eprint.iacr.org/2019/119.pdf) under the assumption of a correct implementation. The actual puzzle implementation contains additional bugs that enable simpler attacks. For the implementation-specific solution, see [this writeup](https://www.zkhack.dev/events/solution6.html).

## The Challenge

We're given a proof system for a simple circuit that checks `4 * x == y`. The goal is to generate a valid proof for an invalid statement: `4 * 1 == 1`.

## Background: How This Proof System Works

### From Circuits to Polynomials

The proof system takes a circuit with $n$ constraints and turns it into a single polynomial $P(x)$. The key idea: $P(x)$ equals zero at $n$ specific points if and only if all constraints are satisfied.

**The circuit**:
```
Public inputs: x, y
Private input: z

Constraints:
  0. x + x == z
  1. z + z == y
```

For the valid inputs $(x=1, y=4, z=2)$:
- Constraint 0: $1 + 1 = 2$
- Constraint 1: $2 + 2 = 4$

### Checking Divisibility

The verifier checks if all constraints are satisfied without computing $P(x)$ fully. This uses a trick with the **vanishing polynomial** $Z(x)$, which equals zero at all the points where constraints are checked:

$$Z(x) = (x - \omega^0)(x - \omega^1) \cdots (x - \omega^{n-1}) = x^n - 1$$

where $\omega$ is an $n$-th root of unity (meaning $\omega^n = 1$). This is a clever choice because at any constraint checking point $\omega^i$:

$$Z(\omega^i) = (\omega^i)^n - 1 = (\omega^n)^i - 1 = 1^i - 1 = 0$$

If $P(x)$ is zero at all constraint points, then $P(x)$ must be perfectly divisible by $Z(x)$ with no remainder. The prover proves this by computing the quotient polynomial:

$$H(x) = \frac{P(x)}{Z(x)}$$

If the constraints aren't satisfied, this division results in a remainder and $H(x)$ such that $P(x) == H(x)*Z(x)$ will not exist

### Building the Constraint Polynomial

The system uses **selector polynomials** to encode which variables appear in which constraints. Consider them as switches that turn on or off at different constraint points.

For each variable, there are two selector polynomials:
- $L_j(x)$: equals 1 at point $\omega^i$ if variable $j$ is on the left side of constraint $i$, otherwise 0
- $O_j(x)$: equals 1 at point $\omega^i$ if variable $j$ is on the right side of constraint $i$, otherwise 0

**Example for variable $x$**:
$$L_x(\omega^0) = 1, \quad L_x(\omega^1) = 0$$
$$O_x(\omega^0) = 0, \quad O_x(\omega^1) = 0$$

This says "$x$ appears on the left of constraint 0, but nowhere else."

Similarly for variables $y$ and $z$:
$$L_y(\omega^0) = 0, \quad L_y(\omega^1) = 0, \quad O_y(\omega^0) = 0, \quad O_y(\omega^1) = 1$$
$$L_z(\omega^0) = 0, \quad L_z(\omega^1) = 1, \quad O_z(\omega^0) = 1, \quad O_z(\omega^1) = 0$$

The constraint polynomial is built as:
$$P(x) = 2 \sum_j v_j L_j(x) - \sum_j v_j O_j(x)$$

where $v_j$ are the actual witness values (like $v_x = 1, v_z = 2, v_y = 4$). The factor of 2 implements the doubling in the `x + x` constraints.

**Checking constraint 0**:
$$P(\omega^0) = 2v_x - v_z = 2(1) - 2 = 0$$

The constraint is satisfied! Similarly for constraint 1:
$$P(\omega^1) = 2v_z - v_y = 2(2) - 4 = 0$$

### The Trusted Setup

If the prover could compute $P(\tau)$ directly, they could cheat. The solution is to hide $\tau$ while still letting the prover work with it.

A **trusted setup** ceremony generates a secret random point $\tau$, uses it to create all the values the prover needs (as elliptic curve points), then destroys $\tau$. The prover can now compute $[H(\tau)]_1$ and $[P(\tau)]_1$ without knowing what $\tau$ actually is.

**What the setup generates**:

- powers_of_tau = $ (\tau^0 * G1, \tau^1 * G1, \tau^2 * G1, ...) $
- inputs[j] = $ \rho * L_j(\tau) * G1 $
- outputs[j] = $ \rho * O_j(\tau) * G1 $
- inputs_prime[j] = $ \alpha_{in} * \rho * L_j(\tau) * G1 $
- outputs_prime[j] = $ \alpha_{out} * \rho * L_j(\tau) * G1 $
- $K[j]$ = $ \beta * \rho * (L_j(\tau) + O_j(\tau)) * G1 $
- alpha_inputs = $ \alpha_i * G2 $
- alpha_outputs = $ \alpha_j * G2 $
- gamma = $ \gamma * G2 $
- beta_gamma = $ \beta * \gamma * G2 $
- rho_Z = $ \rho * Z(\tau) * G2 $

After generation, all the secret values ($\tau, \rho, \alpha_{\text{in}}, \alpha_{\text{out}}, \beta, \gamma$) are destroyed.

#### Why the Random Factors Matter

**The $\rho$ factor** prevents the prover from constructing arbitrary polynomial evaluations. The verification equation requires the prover to provide:

$$\pi_{\text{input}} = \sum_{j \in \text{private}} v_j [\rho L_j(\tau)]_1$$

This represents the prover's contribution to the input polynomial evaluation at $\tau$. The verifier combines this with public input contributions to reconstruct $[\rho P(\tau)]_1$ and verify the divisibility relationship $P(\tau) = H(\tau) \cdot Z(\tau)$ through pairing checks.

The setup provides only $[\rho L_j(\tau)]_1$, never the unblinded values $[L_j(\tau)]_1$. Without knowledge of $\rho$, the prover cannot compute $[\rho Q(\tau)]_1$ for arbitrary polynomials. The prover is restricted to polynomials in the span of the provided selectors:

$$Q(x) \in \text{span}\\{L\_0(x), \ldots, L\_n(x), O\_0(x), \ldots, O\_n(x)\\}$$

While the setup also includes unblinded powers $[\tau^0]_1, [\tau^1]_1, [\tau^2]_1, \ldots$ for computing $H(\tau)$, these cannot be used to compute $[\rho \cdot Q'(\tau)]_1$ for polynomials $Q'(x)$ outside the selector span. The blinding factor $\rho$ cryptographically enforces this restriction.

**The $\alpha$ factors** provide knowledge soundness through the Knowledge of Exponent Assumption (KEA). The setup includes pairs $([\rho L\_i(\tau)]\_1, [\alpha\_{\text{in}} \rho L\_i(\tau)]\_1)$ for each selector polynomial. When a prover produces commitments $(\pi_L, \pi'_L)$ satisfying the verification equation $e(\pi'\_L, [1]\_2) = e(\pi\_L, [\alpha\_{\text{in}}]\_2)$, KEA guarantees the existence of an extractor that can recover witness coefficients $(v_0, \ldots, v_n)$ such that $\pi_L = \sum_i v_i [\rho L_i(\tau)]_1$. This prevents malleability attacks where proofs are copied or algebraically modified without knowledge of the underlying witness.

### Proof Structure

The prover computes:

- pi_H = $ H(\tau) * G1 $
- pi_input = $ \sum{privateInput_j * \rho * privatePolynomial_j(\tau) * G1}$
- pi_input_prime = $ \sum{privateInput_j * \alpha_{in} * \rho * privatePolynomial_j(tau) * G1}$
- pi_K = $\sum{input_j * K[j]}$ = $ \sum{input_j * \beta * \rho * (L_j(\tau) + O_j(\tau)) * G1} $
- pi_output = $ \sum{input_j * \rho * O_j(\tau) * G1}$
- pi_output_prime = $ \sum{input_j * \alpha_{out} * \rho * O_j(\tau) * G1}$

### Verification Checks

The verifier computes the public input contribution and performs four pairing checks:

**1. Input knowledge check**:
$$e(\pi'\_{\text{input}}, [1]\_2) = e(\pi\_{\text{input}}, [\alpha\_{\text{in}}]\_2)$$

**2. Output knowledge check**:
$$e(\pi'\_{\text{output}}, [1]\_2) = e(\pi\_{\text{output}}, [\alpha\_{\text{out}}]\_2)$$

**3. Variable consistency check**:
$$e(\pi\_K, [\gamma]\_2) = e(\text{PI} + \pi\_{\text{input}} + \pi\_{\text{output}}, [\beta\gamma]\_2)$$

where PI = public input contribution computed by verifier.

**4. Divisibility check**:
$$e(2 \cdot \text{PI} + 2 \cdot \pi\_{\text{input}} - \pi\_{\text{output}}, [1]\_2) = e(\pi\_H, [\rho^2 Z(\tau)]\_2)$$

This verifies that $H(\tau) \cdot Z(\tau) = P(\tau)$.

## The Vulnerability

The bug is in what the setup includes. Look at the setup generation code:

```rust
// From setup generation
for i in 0..circuit.inputs.len() {
    let input = &circuit.inputs[i];
    setup.inputs.push(fr_to_g1::<E>(rho * input.evaluate(&tau)));
    setup.inputs_prime.push(fr_to_g1::<E>(alpha_inputs * rho * input.evaluate(&tau)));
}
```

The setup creates $inputs\_prime[j] = [\alpha_{in} * \rho * L_j(\tau)]₁$ for **every** variable, including public inputs.

**What an honest prover does:**
- Computes `π_input` using only **private** input polynomials
- Never touches the `inputs_prime` values for public variables

**What the verifier does:**
- Computes the public input contribution directly from public values
- Never uses `inputs_prime` for public variables either

So these values for public inputs serve no purpose. They're dead weight in the setup. Normally that would just be inefficient. However, these values can be used to forge proofs in this case.

### Why This Breaks Security

The verification protocol requires the verifier to compute the public input contribution:

$$\text{PI} = \sum_{j \in \text{public}} v_j [\rho L_j(\tau)]_1$$

The prover provides the private input contribution $\pi_{\text{input}}$, and the combined value $\text{PI} + \pi_{\text{input}}$ is used in the divisibility and consistency checks.

Consider a valid proof $\pi$ for public inputs $(x_1, y_1)$. An attacker wants to forge a proof for different public inputs $(x_2, y_2)$. The verifier will compute different public contributions:

$$\text{PI}\_1 = \sum\_{j \in \text{public}} v\_{j,1} [\rho L\_j(\tau)]\_1$$

$$\text{PI}\_2 = \sum\_{j \in \text{public}} v\_{j,2} [\rho L\_j(\tau)]\_1$$

The difference $\Delta = \text{PI}\_1 - \text{PI}\_2$ would normally prevent the attack, as modifying public inputs breaks all verification equations. Two security mechanisms should prevent compensation for this difference:

1. **Polynomial restriction**: The prover cannot compute $[\rho Q(\tau)]_1$ for arbitrary polynomials $Q(x)$ due to the blinding factor $\rho$
2. **Knowledge soundness**: The prover cannot produce valid alpha-shifted pairs without knowing the underlying witness

However, the inclusion of $[\alpha_{\text{in}} \rho L_j(\tau)]_1$ for public indices breaks both barriers. The attacker can compute:

$$\delta = \sum\_{j \in \text{public}} (v\_{j,1} - v\_{j,2}) [\rho L\_j(\tau)]\_1$$

$$\delta' = \sum\_{j \in \text{public}} (v\_{j,1} - v\_{j,2}) [\alpha\_{\text{in}} \rho L\_j(\tau)]\_1$$

These values satisfy $\delta' = \alpha_{\text{in}} \cdot \delta$ by construction. Adding $\delta$ to $\pi_{\text{input}}$ exactly cancels the change in public input contribution, while adding $\delta'$ to $\pi'_{\text{input}}$ maintains the knowledge-of-exponent relationship required for verification.

### Step-by-Step Attack

**Given**: Valid proof $\pi_1$ for inputs $(x=1, y=4, z=2)$  
**Goal**: Forge proof $\pi_2$ for invalid inputs $(x=1, y=1)$

**Step 1**: Calculate how the verifier's public input contribution changes.

For $(x=1, y=4)$: verifier computes $\text{PI}\_1 = 1 \cdot [\rho L\_x(\tau)]\_1 + 4 \cdot [\rho L\_y(\tau)]\_1$

For $(x=1, y=1)$: verifier computes $\text{PI}\_2 = 1 \cdot [\rho L\_x(\tau)]\_1 + 1 \cdot [\rho L\_y(\tau)]\_1$

The difference: $\Delta = 3 \cdot [\rho L_y(\tau)]_1$

**Step 2**: Adjust the proof's private input commitment to compensate.

$$\pi\_{\text{input},2} = \pi\_{\text{input},1} + 3 \cdot [\rho L\_y(\tau)]\_1$$

Now $\text{PI}\_2 + \pi\_{\text{input},2} = \text{PI}\_1 + \pi\_{\text{input},1}$, so the divisibility check will pass.

**Step 3**: Maintain the knowledge check relationship.

The knowledge check requires:

$$e(\pi'\_{\text{input}}, [1]\_2) = e(\pi\_{\text{input}}, [\alpha\_{\text{in}}]\_2)$$

If only $\pi_{\text{input}}$ was adjusted, this would fail. But the setup includes the alpha-shifted value for public inputs. Adjust both:

$$\pi'\_{\text{input},2} = \pi'\_{\text{input},1} + 3 \cdot [\alpha\_{\text{in}} \rho L\_y(\tau)]\_1$$

The relationship is preserved:
$$\pi'\_{\text{input},2} = \alpha\_{\text{in}} \cdot \pi\_{\text{input},2}$$

The knowledge check passes.

**Step 4**: All other proof components remain unchanged.

The forged proof passes all verification checks.

### Verification with Forged Proof

The verifier processes the forged proof $\pi_2$ with target inputs $(x=1, y=1)$:

**Public input computation**:
$$\text{PI}\_2 = 1 \cdot [\rho L\_x(\tau)]\_1 + 1 \cdot [\rho L\_y(\tau)]\_1$$

**Divisibility check reconstruction**:
$$[\rho P(\tau)]\_1 = 2 \cdot (\text{PI}\_2 + \pi\_{\text{input},2}) - \pi\_{\text{output},1}$$

Substituting $\pi\_{\text{input},2} = \pi\_{\text{input},1} + 3 \cdot [\rho L\_y(\tau)]\_1$:

$$= 2 \cdot (\text{PI}\_2 + \pi\_{\text{input},1} + 3 \cdot [\rho L\_y(\tau)]\_1) - \pi\_{\text{output},1}$$

$$= 2 \cdot ((1 \cdot [\rho L\_x(\tau)]\_1 + 1 \cdot [\rho L\_y(\tau)]\_1) + \pi\_{\text{input},1} + 3 \cdot [\rho L\_y(\tau)]\_1) - \pi\_{\text{output},1}$$

$$= 2 \cdot (1 \cdot [\rho L\_x(\tau)]\_1 + 4 \cdot [\rho L\_y(\tau)]\_1 + \pi\_{\text{input},1}) - \pi\_{\text{output},1}$$

$$= 2 \cdot (\text{PI}\_1 + \pi\_{\text{input},1}) - \pi\_{\text{output},1}$$

This is exactly the value used in the original valid proof $\pi_1$! The verifier checks:

$$e([\rho P(\tau)]\_1, [1]\_2) = e(\pi\_{H,1}, [\rho^2 Z(\tau)]\_2)$$

Since $\pi_1$ was valid, this equation holds, and the forged proof passes verification.

**Solution Script**

```rust
fn construct_proof(circuit: &Circuit<E>, setup: &Setup<E>) -> Proof<E> {

    // Successfully prove that 1+1+1+1=4
    let two = Fr::one() + Fr::one();
    let four = two + two;
    let public_inputs = [Fr::one(), four];
    let private_inputs = [two];
    let mut proof = prover::prove(&public_inputs, &private_inputs, &circuit, &setup);

    assert!(verifier::verify(&public_inputs, &setup, &proof));

    // Fail to prove that 1+1+1+1=1
    let mut new_pi = proof.pi_input;
    let mut new_pi_prime = proof.pi_input_prime;
    let target_public_inputs = [Fr::one(), Fr::one()];
    for (x, x_, setup_input, setup_input_prime) in izip!(
        public_inputs,
        target_public_inputs,
        setup.inputs.iter(),
        setup.inputs_prime.iter(),
    ) {
        new_pi = new_pi + setup_input.mul(x - x_).into();
        new_pi_prime = new_pi_prime + setup_input_prime.mul(x - x_).into();
    }
    proof.pi_input = new_pi;
    proof.pi_input_prime = new_pi_prime;

    proof
}
```

