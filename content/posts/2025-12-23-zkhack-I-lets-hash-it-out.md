---
title: ZKHack I Puzzle - Let's Hash It Out Writeup
date: 2025-12-23
tags: [ZK, writeup]
slug: zkhack-I-lets-hash-it-out
---


# ZKHack I Puzzle: Let's Hash It Out Writeup

> TL;DR
> The puzzle provides 256 BLS signatures from Alice, each signing a different message. The challenge is to forge Alice's signature for a target username without knowing her private key. The vulnerability: Pedersen hash is a linear combination of message chunks with fixed generators, and BLS signatures scale linearly with the hash. By solving a system of linear equations, any signature can be constructed as a linear combination of the provided signatures.

## The Challenge

The puzzle provides BLS signatures for 256 messages signed by Alice. The goal is to generate a valid signature for a target username (e.g., "S3v3ru5") without access to Alice's private key.

## Background

### Pedersen Hash

Pedersen hash computes a curve point as a linear combination of message chunks with fixed generators:

**Construction:**
1. Divide message into $n$ chunks: $m_1, m_2, \ldots, m_n$
2. Sample $n$ fixed generators: $G_1, G_2, \ldots, G_n$
3. Compute hash: $H(m) = \sum_{i=1}^{n} m_i \cdot G_i$

The puzzle uses `ZkHackPedersenWindow` with configuration:
- `NUM_WINDOWS = 256` (number of chunks)
- `WINDOW_SIZE = 1` (bits per chunk)

This maps each bit of the message to a separate generator.

### BLS Signatures

BLS signature scheme signs a message by hashing it to a curve point and multiplying by the private key $\sigma$:

**Key generation:**
$$pk = \sigma \cdot G_2$$

**Signing:**
$$sig = \sigma \cdot H(m)$$

**Verification:**
$$e(H(m), pk) \stackrel{?}{=} e(sig, G_2)$$

The pairing equation ensures the signature was created using the same private key that corresponds to the public key. The signature proves knowledge of $\sigma$ without revealing it.

The puzzle implementation uses Pedersen hash to map messages to curve points for BLS signing.

## The Vulnerability

**Pedersen hash is a linear combination**: For a message $m$ with bits $(b_1, b_2, \ldots, b_{256})$, the hash is the inner product with fixed generators:

$$H(m) = b_1 \cdot G_1 + b_2 \cdot G_2 + \cdots + b_{256} \cdot G_{256}$$

This linear structure allows computing the hash of any message as a linear combination of other message hashes. Given different messages $m_1, m_2, \ldots, m_n$ and coefficients $x_1, x_2, \ldots, x_n$:

$$x_1 \cdot H(m_1) + x_2 \cdot H(m_2) + \cdots + x_n \cdot H(m_n) = H(m_{target})$$

where the $j$-th bit of $m_{target}$ is computed as:

$$b_j^{(target)} = x_1 \cdot b_j^{(1)} + x_2 \cdot b_j^{(2)} + \cdots + x_n \cdot b_j^{(n)}$$

with $b_j^{(i)}$ denoting the $j$-th bit of message $m_i$.

**BLS signatures preserve this structure**: A BLS signature is scalar multiplication of the message hash with the private key:

$$sig(m) = \sigma \cdot H(m)$$

Taking the same linear combination of signatures:

$$x_1 \cdot sig(m_1) + x_2 \cdot sig(m_2) + \cdots + x_n \cdot sig(m_n)$$

$$= \sigma \cdot (x_1 \cdot H(m_1) + x_2 \cdot H(m_2) + \cdots + x_n \cdot H(m_n))$$

$$= \sigma \cdot H(m_{target}) = sig(m_{target})$$

This produces a valid signature for $m_{target}$. If we can find coefficients such that $m_{target}$ matches our desired username, we can forge the signature.

## Solution

Given signatures $s_1, s_2, \ldots, s_n$ for messages $m_1, m_2, \ldots, m_n$, each signature has the form:

$$s_i = \sigma \cdot H(m_i) = \sigma \cdot \sum_{j=1}^{256} m_{ij} \cdot G_j$$

where $m_{ij}$ is the $j$-th bit of message $m_i$.

Expanding the signatures:

$$s_1 = \sigma \cdot (m_{11} \cdot G_1 + m_{12} \cdot G_2 + \cdots + m_{1,256} \cdot G_{256})$$
$$s_2 = \sigma \cdot (m_{21} \cdot G_1 + m_{22} \cdot G_2 + \cdots + m_{2,256} \cdot G_{256})$$
$$\vdots$$
$$s_{256} = \sigma \cdot (m_{256,1} \cdot G_1 + m_{256,2} \cdot G_2 + \cdots + m_{256,256} \cdot G_{256})$$

The target signature for username with bit representation $(z_1, z_2, \ldots, z_{256})$ is:

$$sig = \sigma \cdot (z_1 \cdot G_1 + z_2 \cdot G_2 + \cdots + z_{256} \cdot G_{256})$$

### Finding the Linear Combination

The target signature can be written as:

$$sig = x_1 \cdot s_1 + x_2 \cdot s_2 + \cdots + x_{256} \cdot s_{256}$$

Substituting the signature expressions:

$$\sigma \cdot \sum_{j=1}^{256} z_j \cdot G_j = \sigma \cdot \sum_{i=1}^{256} x_i \left(\sum_{j=1}^{256} m_{ij} \cdot G_j\right)$$

$$= \sigma \cdot \sum_{j=1}^{256} \left(\sum_{i=1}^{256} x_i \cdot m_{ij}\right) G_j$$

Since the generators $G_j$ are linearly independent, this requires:

$$z_j = \sum_{i=1}^{256} x_i \cdot m_{ij} \quad \text{for all } j \in \{1, \ldots, 256\}$$

This is a system of 256 linear equations in 256 unknowns.

### Matrix Formulation

Construct matrix $M$ where column $i$ contains the bits of message $m_i$:

$$M = \begin{bmatrix}
m_{1,1} & m_{2,1} & \cdots & m_{n,1} \\\\
m_{1,2} & m_{2,2} & \cdots & m_{n,2} \\\\
\vdots & \vdots & \ddots & \vdots \\\\
m_{1,256} & m_{2,256} & \cdots & m_{n,256}
\end{bmatrix}$$

where $m_{i,j}$ is the $j$-th bit of message $m_i$.

The target vector $\vec{z} = (z_1, z_2, \ldots, z_{256})$ represents the bits of the target username.

Solve for coefficient vector $\vec{x} = (x_1, x_2, \ldots, x_n)$:

$$M \vec{x} = \vec{z}$$

The forged signature is:

$$sig = \sum_{i=1}^{n} x_i \cdot s_i$$


```rust
fn solve_puzzle() {
    let (pk, ms, sigs) = puzzle_data();
    
    // Step 1: Hash all messages to get 256-bit hashes and G1 points
    let mut hash_points: Vec<(Vec<u8>, G1Affine)> = Vec::new();
    
    for m in ms.iter() {
        let (hash_bytes, point) = hash_to_curve(m);
        hash_points.push((hash_bytes, point));
    }
    
    // Step 2: Construct matrix A with each 256-bit hash as a column
    // Each column is 256 bits (32 bytes), so we have a 256x256 matrix
    let mut matrix: Vec<Vec<Fr>> = vec![vec![Fr::zero(); 256]; 256];
    
    for col in 0..256 {
        let hash_bytes = &hash_points[col].0;
        
        for byte_idx in 0..32 {
            for bit_idx in 0..8 {
                let row = byte_idx * 8 + bit_idx;
                let bit = (hash_bytes[byte_idx] >> bit_idx) & 1;
                matrix[row][col] = Fr::from(bit as u64);
            }
        }
    }
    
    // Step 3: Generate hash for target message "S3v3ru5"
    let target_message = b"S3v3ru5";
    let (target_hash_bytes, target_point) = hash_to_curve(target_message);
    
    // Step 4: Create target column vector b
    let mut target_column: Vec<Fr> = vec![Fr::zero(); 256];
    
    for byte_idx in 0..32 {
        for bit_idx in 0..8 {
            let row = byte_idx * 8 + bit_idx;
            let bit = (target_hash_bytes[byte_idx] >> bit_idx) & 1;
            target_column[row] = Fr::from(bit as u64);
        }
    }
    
    // Solve A*x = b using Gaussian elimination
    let solution = gaussian_elimination(&mut matrix, &mut target_column)
        .expect("Failed to solve linear system");
    
    // Step 5: Compute linear combination of points using solution vector
    // Result = x1*P1 + x2*P2 + ... + x256*P256
    let mut result = G1Projective::default();
    
    for i in 0..256 {
        let point = hash_points[i].1;
        let scalar = solution[i];
        if scalar.is_zero() {
            continue;
        }
        
        let point_proj = G1Projective::from(point);
        
        let scaled_point = point_proj.mul(scalar.into_repr());
        result += scaled_point;
    }
    
    let result_affine = result.into_affine();
    
    println!("Verifying result...");
    assert_eq!(
        result_affine, 
        target_point,
        "Linear combination of points does not match target point!"
    );

    // Compute forged signature for "S3v3ru5"
    // forged_sig = x1*Sig1 + x2*Sig2 + ... + x256*Sig256
    println!("\nComputing forged signature...");
    let mut forged_signature = G1Projective::zero();
    
    for i in 0..256 {
        let sig = sigs[i];
        let scalar = solution[i];

        if scalar.is_zero() {
            continue;
        }

        let sig_proj = sig.into_projective();

        let scaled_sig = sig_proj.mul(scalar.into_repr());
        forged_signature += scaled_sig;
    }
    
    let forged_sig_affine = forged_signature.into_affine();
    
    println!("Forged signature computed: {:?}", forged_sig_affine);
    
    // Verify the forged signature
    println!("\nVerifying forged signature for 'S3v3ru5'...");
    verify(pk, target_message, forged_sig_affine);
    println!("Forged signature is valid!");
}
```
