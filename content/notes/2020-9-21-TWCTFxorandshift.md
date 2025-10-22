---
title: TokyoWesterns CTF 2020 XOR and shift Writeup
date: 2020-09-21 19:46:30
tags: [crypto,writeup]
slug: TWCTFxorandshift
---

# XOR and shift encryptor [Crypto]

- Challenge files :<br>
	- problem_pub.py
	- enc.dat

`problem_pub.py`
<script src="https://gist.github.com/S3v3ru5/5c1b7a337f56a7237cc7a1e75d5bf193.js?file=problem_pub.py"></script>

### Understanding the problem_pub.py
The overall execution flow of program is as follows:

- Create a state of 64 elements(64 bitlength) and initialise it to 0..63
- update the state for `31337` rounds
- for each character in the `flag` 
	- generate a random number `(ri)` using the state
	- update the state for `ri` rounds
	- generate a random number `ki` using the state
	- encrypt `flag character` by `xoring` with lower byte of `ki`

`enc.dat` contains the ciphertext, generating the keystream and xoring it with `enc.dat` will give the flag.

state is updated as follows
```python
def randgen():
  global s,p # s -> state
  a = 3
  b = 13
  c = 37
  s0 = s[p]
  p = (p + 1) & 63
  s1 = s[p]
  res = (s0 + s1) & ((1<<64)-1)
  s1 ^= (s1 << a) & ((1<<64)-1)
  s[p] = (s1 ^ s0 ^ (s1 >> b) ^ (s0 >> c))  & ((1<<64)-1)
  return res
```
`jump` function is used to update the state for `n` rounds directly, it's implementation is removed from the given source file.

basic working of `jump` function can be inferred from the `check_jump()` function.<br>  
naive implementation of `jump` function would be
```python
def jump(to):
	for _ in range(to):
		randgen()
```
Running the `problem_pub.py` by replacing the `flag` with `enc.dat` would give us the `flag` if it could complete it's `execution`.<br>

The problem here is `jump(to)` is too slow, as `to` parameter is of size `64 bits`.<br>
It has to iterate that many times for each character, any sane person would agree that it will take ages to complete its execution and more
likely the `process` will be `Killed` much early on.

In order to solve the challenge, all that remains is to optimise the `jump(to)` function.

### reformulating state update operations (randgen())

```python
s1 ^= (s1 << a) & ((1<<64)-1) # s0 = s[p], s1 = s[p+1], p += 1 % 64
s[p] = (s1 ^ s0 ^ (s1 >> b) ^ (s0 >> c))  & ((1<<64)-1)
```
In each round, state value at index p is updated by doing bit-wise operations between state[p] and state[p-1]

One thing that caught my attention is that only `bitwise operations` are used because from the moment I watched [@LiveOverflow](https://twitter.com/LiveOverflow) video writeup for software update challenge [\[Link\]](https://www.youtube.com/results?search_query=live+overflow+software+update) I made sure to try representing bitwise operations especially `xor` used in the challenge as some mathematical operations and to work on them.

Though it's not necessary to understand this solution, I strongly recommend to watch that video.

Coming back to our solution, my intial thoughts were 
- encode each element as a mathematical structure(e.g polynomials) in `GF(2)`
- write each operation used in challenge as a primitive operations of that structure
- Try to construct a `Matrix M` such that `M * state` will give us the `future state`
- using `M`, replace `jump(to)` function with `M**to * state`

**Encoding as Polynomials** \mathbb{F} <br>

`xor` operation can be replaced with `polynomial addition` as it is property of underlying `Field`.

I know that `rotation of bits` can be represented as `primitive operation`
of `polynomials` [\[Link\]](https://math.stackexchange.com/questions/472108/the-rotate-and-shift-operations-in-a-finite-field).
I couldn't find a way to represent `left shift (<<)` and `right shift (>>)`
as a `primitive operation`.

**Encoding as vectors**

similar to `polynomial addition`, `xor` can be replaced with `vector addition`.

Through some googling, I found out about [Shift Matrices](https://en.wikipedia.org/wiki/Shift_matrix). <br>
 A `Shift Matrix` can be used to `shift` a `column vector` upwards or downwards by multiplying it with `column vector`.
- encoding `n bit integer` as `column vector` by converting each bit of `integer` as element in `GF(2)`. 
- shifting `column vector` upwards is equivalent to `left shift (<<) of integer` and shifting downwards is equivalent to `right shift (>>)`.

e.g shift matrix for n = 4

$$
\begin{equation\*}
	U = 
	\begin{pmatrix}
	 0&1&0&0\\\0&0&1&0\\\0&0&0&1\\\0&0&0&0
  \end{pmatrix}
  ,\quad L = 
  \begin{pmatrix}
    0&0&0&0\\\1&0&0&0\\\0&1&0&0\\\0&0&1&0
  \end{pmatrix}
\end{equation\*}
$$

```python
si << 1 = Un * tovec(si) # n = si.bit_length()
si >> 1 = Ln * tovec(si)

# shifting left or right a times is equivalent to repeated multiplication
# of shift matrix

si << a = (Un**a) * tovec(si)
si >> a = (Ln**a) * tovec(si)
```
All the operations used in `randgen()` function can be represented as
```
ti ^ ri = tovec(ti) + tovec(ri)
ti << a = (Un ** a) * tovec(ti)
ti >> a = (Ln ** a) * tovec(ti)
```

### Construct final Matrix M

From now onwards, I will use `si` to represent `integer` as well as a `vector` interchangeably.

For a better understanding, let's consider a small `state` of 4 elements(4 bits each)

let \begin{equation\*}state = \begin{matrix}s0&s1&s2&s3\end{matrix}\end{equation\*}

and I be an Identiry Matrix 

Round 1(p=0):
```python
s1 = s1 ^ (s1 << a) & ((1<<4)-1) # s0 = s[p], s1 = s[p+1], p += 1 % 4
s[p] = (s1 ^ s0 ^ (s1 >> b) ^ (s0 >> c))  & ((1<<4)-1)
```
$$
\begin{align}
& s1 = s1 + (U_{4}^{a} \* s1) = (U_{4}^{a} + I) \* s1\hspace{50cm} \\\
\end{align}
$$

$$
\begin{align}
& s1 = s0 + (U_{4}^{a} + I) \* s1 + L_{4}^{b} \* (U_{4}^{a} + I) \* s1 + L_{4}^{c} \* s0 \hspace{50cm} \\\
\end{align}
$$

$$
\begin{align}
& s1 = (L_{4}^{c} + I) \* s0 + (L_{4}^{b} \* (U_{4}^{a} + I) + (U_{4}^{a} + I)) \* s1\hspace{50cm}\\\
\end{align}
$$

<br>
Let 

$$
\begin{equation\*}
X = (L_{4}^{c} + I) \\\
\end{equation\*}
$$

$$
\begin{equation\*}
Y =  (L_{4}^{b} \* (U_{4}^{a} + I) + (U_{4}^{a} + I)) \\\
\end{equation\*}
$$

then,

$$
\begin{equation\*}
s1 = X \* s0 + Y \* s1 \\\
\end{equation\*}
$$

we can represent this operation as matrix multiplication : <br>
let `state` be a `column vector`
and `N` be a null Matrix

Consider
$$
\begin{equation\*}
	M1 = 
	\begin{pmatrix}
	 I&N&N&N\\\X&Y&N&N\\\N&N&I&N\\\N&N&N&I
  \end{pmatrix}
\end{equation\*}
$$

and

$$
\begin{equation\*}
	M1 \* state = 
	\begin{pmatrix}
	 I&N&N&N\\\X&Y&N&N\\\N&N&I&N\\\N&N&N&I
  \end{pmatrix}
  \quad \* \quad
  \begin{pmatrix}
  s0\\\s1\\\s2\\\s3
  \end{pmatrix}
  \quad = \quad
  \begin{pmatrix}
  s0\\\X \* s0 + Y \* s1\\\s2\\\s3
  \end{pmatrix}
\end{equation\*}
$$

similarly we can generate Mi for Round 2, 3, 4 (p=1,2,3):

$$
\begin{equation\*}
s2 = X \* s1 + Y \* s2 \\\
\end{equation\*}
$$

$$
\begin{equation\*}
s3 = X \* s2 + Y \* s2 \\\
\end{equation\*}
$$

$$
\begin{equation\*}
s0 = X \* s3 + Y \* s0 \\\
\end{equation\*}
$$

And

$$
\begin{equation\*}
	M2 = 
	\begin{pmatrix}
	 I&N&N&N\\\N&I&N&N\\\N&X&Y&N\\\N&N&N&I
  \end{pmatrix}
  ,\quad M3 = 
  \begin{pmatrix}
	 I&N&N&N\\\N&I&N&N\\\N&N&I&N\\\N&N&X&Y
  \end{pmatrix}
  ,\quad M4 = 
  \begin{pmatrix}
	 Y&N&N&X\\\N&I&N&N\\\N&N&I&N\\\N&N&N&I
  \end{pmatrix}
\end{equation\*}
$$

Using Matrices M1, M2, M3, M4 state can be updated by multiplying corresponding matrix based on p value.

suppose p = 0, updating state for 8 rounds i.e `jump(to)` can be written as

$$
\begin{equation\*}
(M4 \* (M3 \* (M2 \* (M1 \* (M4 \* (M3 \* (M2 \* (M1 \* state))))))))
\end{equation\*}
$$

As Matrix multiplication is Associative, we can change above equation as
$$
\begin{equation\*}
((M4 \* M3 \* M2 \* M1) \* (M4 \* M3 \* M2 \* M1)) \* state)
\end{equation\*}
$$

By taking
$$
\begin{equation\*}
M = M4 \* M3 \* M2 \* M1
\end{equation\*}
$$

we can write above equation as
$$
\begin{equation\*}
M^{2} \* state
\end{equation\*}
$$

Using M, `jump(to)` can be written as
```python
def jump(to):
	global pstate, pind
	# make pind(p) equal to zero as
	while pind != 0 and to > 0:
		res = randgen()
		to -= 1
	if to == 0:
		return
	powne = to // 4
	Mn = genMn(powne) # calculates M**pown
	state = (Mn * ints_2_state(pstate, dimension))
	pstate = state_2_ints(state, dimension)
	for _ in range(to % 4):
		randgen()
	return
```

We can extend the above approach very easily for state with length `64`.


`solve.sage`
<script src="https://gist.github.com/S3v3ru5/5c1b7a337f56a7237cc7a1e75d5bf193.js?file=xorandshift-solve.sage"></script>
