---
title: TokyoWesterns CTF 2020 Writeups 1
date: 2020-09-21 19:46:30
tags: [crypto,writeup,pwn]
slug: tokyowesternsCTF
---

# TokyoWesterns CTF 2020 Writeups

- Crypto
	- [twin-d](#twind)
	- [sqrt](#sqrt)
	- [The Melancholy of Alice](#themelancholy)
	- [circular](#circular)
- Pwn
	- [nothing more to say 2020](#nothingmore)

---

First things first, kudos to the team [@TokyoWesterns](https://twitter.com/TokyoWesterns) for the excellent CTF, enjoyed playing it.I managed to solve all crypto challenges and a pwn warmup challenge. Our team [@teaminvaders0](https://twitter.com/teaminvaders0) ended up at position #21.

If there's any mistake in the writeup or you're having a hard time understanding the solution or anything else, feel free to DM me [@S3v3ru5\_](https://twitter.com/S3v3ru5_).

# <a name="easyhash"></a> Twin-d [Crypto]

we were given `task.rb` and `output` of it.
<!-- <script src="https://gist.github.com/S3v3ru5/5c1b7a337f56a7237cc7a1e75d5bf193.js"></script> -->

`flag` is encrypted with textbook RSA. As the name of the challenge suggests,
twin private exponents are generated (d1, d1 + 2) and their corresponding public exponents (e1, e2) are given.<br>
`task.rb`
<script src="https://gist.github.com/S3v3ru5/5c1b7a337f56a7237cc7a1e75d5bf193.js?file=twind-task.rb"></script>
`output`
<script src="https://gist.github.com/S3v3ru5/5c1b7a337f56a7237cc7a1e75d5bf193.js?file=small_code_snippets.py"></script>

Using e1 and e2, we can write equations as follows
```python
e1 * d1 = 1 mod phi(n)
e1 * d1 = 1 + k1*phi(n)
d1 = (1 + k1*phi(n)) / e1

d2 = d1 + 2
e2 * d2 = 1 mod phi(n)
e2 * (d1 + 2) = 1 + k2*phi(n)
e2 * d1 + 2*e2 = 1 + k2*phi(n)
((e2 * (1 + k1*phi(n))) / e1) + 2*e2 = 1 + k2*phi(n)
e2 + e2*k1*phi(n) + 2*e1*e2 = e1 + e1*k2*phi(n)
e2 - e1 + 2*e1*e2 = e1*k2*phi(n) - e2*k1*phi(n)
e2 - e1 + 2*e1*e2 = phin(n) * (e1*k2 - e2*k1)
```
by calculating `e2 - e1 + 2 * e1 * e2` we can obtain a multiple of `phi(n)`.
we can treat it as `phi(n)` to calculate the `private exponent d1` and decrypt
the ciphertext.

**FLAG :: TWCTF{even_if_it_is_f4+e}**

---

# <a name="sqrt"></a> Sqrt [Crypto]

we were given `chall.py` and its `output`.<br>
`chall.py`
<script src="https://gist.github.com/S3v3ru5/5c1b7a337f56a7237cc7a1e75d5bf193.js?file=sqrt-chall.py"></script>

```python
p = 6722156186149423473586056936189163112345526308304739592548269432948561498704906497631759731744824085311511299618196491816929603296108414569727189748975204102209646335725406551943711581704258725226874414399572244863268492324353927787818836752142254189928999592648333789131233670456465647924867060170327150559233
c = 5602276430032875007249509644314357293319755912603737631044802989314683039473469151600643674831915676677562504743413434940280819915470852112137937963496770923674944514657123370759858913638782767380945111493317828235741160391407042689991007589804877919105123960837253705596164618906554015382923343311865102111160
```
we have two values `p (prime)` and `c = flag**(2**64) % p`.<br>

To get the flag, we have to calculate `2**64` root of `c` modulo `p`.<br>

we cannot directly calculate the inverse of `2**64` as in the case of RSA cause
the `gcd(2**64, p - 1) != 1`.<br>

Though we can calculate `2**64` root of `c` by repeatedly calculating the `square root` which can done in polynomial time, this is not feasible cause for a given `square` there are `2` possible `square roots x and -x` so, the number of roots will increase exponentially.<br>


we have to look at the structure of the group of integers modulo p.<br>
factoring `(p-1)` results in `2**30 * q` where `q` is a large `prime`.<br>
```python
q = 6260495806252046929286845900294522859477928195432517298076552742112950886324892284005656588584952135860464814694781314411135020941596864321946343173249814754547221898776857696421175805576386605600709481536943693517957341228010065655986626401676101786949671425529135194729299908929006799985530842254243
```
As `(p-1) = 2**30 * q`, there are subgroups with order `q`, `2**30`, `2**29*q`...<br>

Note : order here is multiplicative order <br>

Given a random element `0 < m < p`, it will have order of form `2**i * q**j`
with `0 <= i <= 30 and j = 0 or 1`. most likely the order will be `(p-1) = 2**30 * q`<br>

suppose order of `m` is in form `2**i * q` then 
`ti = m ** (2**i)` will have order `q`.<br>

we can use this fact to reduce the number of square root operations required.<br>

As an example, let order of `m` = `(2**30)*q` <br>

`m**(2**64)` can be written as `(m**(2**30))**(2**34) = ti**(2**34)` where `ti = m**2**30`.`ti` will have multiplicative order of `q` and `GCD(2**34, q) == 1`<br>

we can easily calculate `ti` by exponentiating `c = m**(2**64)` with `inverse(2**34, q)`.<br>

Now, we have `ti = m**(2**30)`, we can use repeated square rooting to calculate `m`.<br>

we can optimise that algorithm by calculating only `one` of the possible roots and generate `remaining roots` using it.
```
given r1 such that r1**(2**30) == t
calculate all xi such that x1**(2**30) == 1
multiplying each xi with r1 generates all remaining roots as

(r1 * xi)**(2**30) = r1**(2**30) * xi**(2**30) = t * 1 = t

xi can be calculated by first calculating generator of subgroup 
with order 2**30 and succesive powers of the generator will give 
the required xi's.  
```

The same approach can be used to calculate the `flag`.

**FLAG :: TWCTF{17s_v3ry_34sy_70_f1nd_th3_n_7h_r007}**

---

# <a name="themelancholy"></a>The Melancholy of Alice [Crypto]

Each character of the `flag` is encrypted separately using `Elgamal Encryption` and we are given the `ciphertexts`
`encrypt.py`
<script src="https://gist.github.com/S3v3ru5/5c1b7a337f56a7237cc7a1e75d5bf193.js?file=melancholy-encrypt.py"></script>

If there's a way to check 
	given m, ElagamalEncryption(m) == (c1, c2)
	without additional information
we can bruteforce values of m as they are comprised only printable characters.
I am not sure but there ain't a way to do that using only p and g has order of large prime.
```
Elagamal Encryption:
	pubkey = (p, q, g, g**x) where q is order of g
	privkey = x

To encrypt message m:
	generate a random number r1 from (1, q)
	c1 = g**r1
	c2 = m * ((g**x)**r1)
To decrypt:
	m = c2 * inverse(c1**x)
```

```python
p = 168144747387516592781620466787069575171940752179672411574452734808497653671359884981272746489813635225263167370526619987842319278446075098036112998679570069486935297242638675590736039429506131690941660748942375274820626186241210376537247501823653926524570571499198040207829317830442983944747691656715907048411
q = 84072373693758296390810233393534787585970376089836205787226367404248826835679942490636373244906817612631583685263309993921159639223037549018056499339785034743467648621319337795368019714753065845470830374471187637410313093120605188268623750911826963262285285749599020103914658915221491972373845828357953524205
g = 2
h = 98640592922797107093071054876006959817165651265269454302952482363998333376245900760045606011965672215605936345612030149799453733708430421685495677502147392514542499678987737269487279698863617849581626352877756515435930907093553607392143564985566046429416461073375036461770604488387110385404233515192951025299
```
In our case q is not a prime and there are few small factors in q <br>
namely :: 3, 5, 19, 2<br>
using `pohlig hellman` attack on `g**x, g` we can calculate the `x` modulo `2*3*5*19` which resulted in `xj = x % 2*3*5*19 = 137`.<br>

Though we cannot decrypt the ct, we can check whether given `m` is a possible 
message as follows
```
t1 = c2 * inverse(m, p)
if m is correct message then
	t1 = (g**r1)**x
	c1 = (g**r1)
	use pohlig hellman attack to calculate x % 2*3*5*19
	if dlog == 137
		add m as a possible correct message
```
There are few false positives while calculating the flag but slight guessing will clean up the junk.

**FLAG:: TWCTF{8d560108444cc360374ef54433d218e9_for_the_first_time_in_9_years!}**

---

# <a name="circular"></a> Circular [Crypto]

To solve this challenge we have to calculate `x`, `y` given `k`, `n`, `message` such that <br>

`x**2 + k*y**2 = message mod n` <br>

I found this paper [https://www.researchgate.net/publication/262234058\_An\_efficient\_solution\_of\_the\_congruence](https://www.researchgate.net/publication/262234058_An_efficient_solution_of_the_congruence)<br>

That paper gives an efficient algorithm to solve the exact equation as in our challenge. so, all we need to do is implement the algorithm described in the paper.

`solve.py`
<script src="https://gist.github.com/S3v3ru5/5c1b7a337f56a7237cc7a1e75d5bf193.js?file=circular-solve.py"></script>


**FLAG :: TWCTF{dbodfs-dbqsjdpso-mjcsb-mfp}**

---

# <a name="nothing"></a>Nothing is more to say 2020 [Pwn]

we were given source code along with the binary,there's a format string bug.
```
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX disabled
    PIE:      No PIE (0x400000)
    RWX:      Has RWX segments
```
`nothing.c`
<script src="https://gist.github.com/S3v3ru5/5c1b7a337f56a7237cc7a1e75d5bf193.js?file=nothing.c"></script>

```
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX disabled
    PIE:      No PIE (0x400000)
    RWX:      Has RWX segments
```
As we have unlimited FSB's
exploit plan is
```
1. Leak buf address, rbp of main function stack frame
2. Overwrite return pointer stored on stack byte by byte with (buf + 8) address
3. send "q"*8 + shellcode as input, "q" breaks the while loop and
 returning from main function, will make to jump to our shellcode
```
`exploit.py`
<script src="https://gist.github.com/S3v3ru5/5c1b7a337f56a7237cc7a1e75d5bf193.js?file=exploit.py"></script>

**FLAG :: TWCTF{kotoshi_mo_hazimarimasita_TWCTF_de_gozaimasu}**

 











