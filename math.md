## Mathematical Foundations of the Ramanujan Cipher

The Ramanujan Cipher is a **symmetric encryption scheme** that transforms each plaintext byte \(x \in \{0,1,\dots,255\}\) into a linear equation of the form  

\[
a \cdot X + b \equiv c \pmod{256}
\]

where \(a,b,c\) are integers in \([0,255]\), \(X\) is the unknown variable representing the plaintext byte, and the equation is written using exotic Unicode symbols (Egyptian hieroglyphs or Tengwar). The coefficients \(a\) and \(b\) are derived from a pseudorandom keystream generated from a secret key, and \(c\) is computed as  

\[
c \equiv a \cdot x + b \pmod{256}.
\]

The ciphertext is the sequence of these equations (one per plaintext byte). Decryption recovers \(x\) by solving the linear congruence given \(a,b,c\).

---

### 1. Modular Arithmetic Over \(\mathbb{Z}_{256}\)

The ring \(\mathbb{Z}_{256} = \mathbb{Z}/256\mathbb{Z}\) consists of residues modulo 256. Addition and multiplication are performed modulo 256.  

**Equation:**  
\[
a \cdot x + b \equiv c \pmod{256}
\]

Given \(a,b,c\) (known from the ciphertext and the key), we want to find \(x \in \mathbb{Z}_{256}\). This is a linear congruence.

#### 1.1 Solving the Congruence

Rewrite as  

\[
a \cdot x \equiv c - b \pmod{256}.
\]

Let \(d = \gcd(a,256)\). The congruence has a solution **if and only if** \(d \mid (c - b)\). In our encryption, \(c\) is defined as \(a x + b \mod 256\), so \(c - b \equiv a x \pmod{256}\) is automatically divisible by \(d\). Thus a solution always exists.

The number of solutions is \(d\). When \(d = 1\) (i.e., \(a\) is odd), the solution is unique and can be found using the modular inverse:

\[
x \equiv a^{-1} \cdot (c - b) \pmod{256}.
\]

When \(d > 1\) (i.e., \(a\) is even), there are multiple solutions. The decryption function in the code uses **brute force** (checking all 256 possibilities) because it is simple and fast for a single byte. For completeness, the set of solutions can be described as:

\[
x \equiv x_0 + t \cdot \frac{256}{d} \pmod{256}, \quad t = 0,1,\dots,d-1
\]

where \(x_0\) is any particular solution. In our case, the encryption ensures that the original \(x\) is one of them, and the receiver recovers it by trying all possibilities.

#### 1.2 Why Brute Force is Acceptable

- Only 256 possibilities per byte → negligible computational cost.
- The cipher is intended for **creative, non‑professional** use, not high‑performance or high‑security applications.

---

### 2. Keystream Generation

A **pseudorandom keystream** is derived from the secret key using SHAKE‑256 (a extendable‑output function). For a plaintext of length \(n\) bytes, we generate \(3n\) bytes:

\[
K = \text{SHAKE256}(\text{key}, 3n)
\]

These bytes are split into triples \((a_i, b_i, \text{unused})\) where

\[
a_i = K[3i] \bmod 256, \quad b_i = K[3i+1] \bmod 256.
\]

To avoid degenerate cases, the code ensures \(a_i\) is never 0 (by taking modulo 255 and adding 1), but in principle \(a=0\) would give \(c \equiv b\) and the equation would be \(b \equiv b\), revealing nothing about \(x\). The chosen method guarantees a valid affine transformation.

The keystream is **deterministic** given the key, allowing the receiver to reproduce the same \(a_i, b_i\) during decryption.

---

### 3. Exotic Digit Mapping

The integers \(a, b, c\) are represented as strings of exotic Unicode symbols (Egyptian hieroglyphs or Tengwar digits). This is a **simple substitution cipher** over the ten digits \(0\)–\(9\). For example:

\[
0 \to \text{𓄤},\quad 1 \to \text{𓋴},\quad 2 \to \text{𓂧},\quad \dots
\]

The mapping is fixed (either Egyptian or Tengwar) and chosen deterministically from the key (lowest bit of the key hash). This adds an extra layer of obscurity: an attacker must know both the digit mapping and the coefficients \(a,b,c\) to recover the plaintext.

The equation format is:

\[
a\text{ (exotic digits)} \; \times \; X \; + \; b\text{ (exotic digits)} \; = \; c\text{ (exotic digits)}
\]

where \(\times\) is represented by a cuneiform symbol, \(+\) by another, and \(=\) by a hieroglyphic combination. The variable \(X\) is written with a Phoenician aleph.

---

### 4. Encryption and Decryption as Mathematical Transformations

#### Encryption  
For each plaintext byte \(x_i\):

1. Draw \(a_i, b_i\) from the keystream.
2. Compute \(c_i = (a_i \cdot x_i + b_i) \mod 256\).
3. Output the equation string \(E_i = \text{format}(a_i, b_i, c_i)\).

#### Decryption  
For each equation \(E_i\):

1. Parse the exotic digits to recover \(a_i, b_i, c_i\) (using the same digit mapping derived from the key).
2. Find \(x_i\) by solving \(a_i \cdot x_i + b_i \equiv c_i \pmod{256}\).  
   In the code, this is done by brute‑force search over \(x = 0,\dots,255\) until the congruence holds.
3. Append \(x_i\) to the plaintext.

---

### 5. Security Notes (Mathematical)

- **Without the key**, an attacker sees only a list of equations in an unknown script. Even if the script is recognized (e.g., Egyptian digits), the coefficients \(a,b,c\) are visible as numbers. However, without the keystream, the attacker cannot separate the \(a,b\) derived from the key from the plaintext \(x\). The system reduces to a **one‑time pad** over the affine group: the pair \((a,b)\) acts as a key per byte. If the same key is reused for multiple messages, the system becomes vulnerable to frequency analysis and known‑plaintext attacks.
- **Modulo 256** is not a field (256 is not prime), so the affine transformation is not bijective for even \(a\). This introduces ambiguity, but the brute‑force search resolves it. For a more robust design, one could use a prime modulus (e.g., 257) or restrict \(a\) to odd numbers.
- **The exotic digit mapping** adds steganographic strength but no cryptographic security; it is a form of obfuscation.

---

### 6. Summary of Key Mathematical Formulas

- **Keystream**: \(K = \text{SHAKE256}(\text{key}, 3n)\)
- **Coefficients**: \(a_i = (K[3i] \bmod 255) + 1,\quad b_i = K[3i+1] \bmod 256\)
- **Ciphertext generation**: \(c_i = (a_i \cdot x_i + b_i) \mod 256\)
- **Equation representation**: \(a_i \times X + b_i = c_i\) (written with exotic symbols)
- **Decryption**: Find \(x_i\) such that \(a_i \cdot x_i + b_i \equiv c_i \pmod{256}\) (brute force)

This mathematics is fully implemented in the provided Python code. The creative use of exotic scripts and mathematical notation makes the cipher appealing for artistic or educational purposes, while the underlying algebra is transparent and easily verifiable.
