# [Crypto] Free Candy

**Target:** `nc 0.cloud.chals.io 19521`  
The server signs JSON tickets using **ECDSA/secp256k1**. Each ticket is a Base64 JSON blob:

```json
{"payload":{"ticket_id": <int>}, "signature": "<64-byte hex r||s>"}
```

A quirky PRNG powers **both** the `ticket_id` stream and the ECDSA nonces `k` (two instances with the **same transition**, different seeds).

---

## TL;DR

- Collect **four consecutive** tickets `t0..t3` (use the “even ticket → new ticket” menu trick).
- With three signatures `(r_i,s_i)` over `t_i` and digests `z_i`, and with the fact that **nonces are an affine combo of consecutive tickets**
  \[ k_i = \alpha t_i + \beta t_{i+1} \ (\bmod N), \]
  you get **three linear equations** in unknowns `(d, α, β)` from ECDSA:
  \[ r_i d - s_i\alpha t_i - s_i\beta t_{i+1} \equiv -z_i \pmod N. \]
- The ticket stream itself obeys a **2nd‑order linear recurrence**
  \[ t_{i+2} = T t_{i+1} + D t_i \pmod N. \]
  Two more equations with `t0..t3` give `(T, D)`. Solve the 5×5 modular system → recover **private key `d`**.
- Forge a “winning” ticket where `ticket_id = sha256("I'd like the flag please")` and submit to get the flag.

---

## Setup

- Curve: secp256k1 with standard `(P, N, A=0, B=7, G)`.
- Hash: SHA‑256 of canonical JSON bytes of `{"ticket_id": tid}` (compact separators, sorted keys).
- Signature: 64 hex bytes = `r||s` (big‑endian, 32 bytes each), embedded as a JSON string.

---

## Vulnerability

The service creates two PRNG instances with **identical transition parameters**; one generates tickets, one generates the ECDSA nonces used immediately after each ticket is minted. For linear order‑2 recurrences (think Lucas sequences), **any sequence produced under the same transition lies in the same 2‑dimensional span**, so there exist constants `(α, β)` such that:
\[ k_i = \alpha t_i + \beta t_{i+1} \pmod N. \]

Because ECDSA is linear in `k^{-1}` and `d`, this correlation makes the private key **linearly recoverable** from a handful of signatures.

---

## Exploit plan

1. **Harvest 4 consecutive tickets:**  
   Use *Get a free ticket* to get `(t0, r0, s0)`. If `t0` is **even**, submit it via *Claim*; the server returns a brand‑new ticket `(t1, r1, s1)` in the response. Repeat until you obtain `t2` and `t3`. Empirically ~1/8 sessions succeed (need three evens in a row).

2. **Build the linear system modulo `N`:**
   
   - For `i = 0,1,2`, ECDSA gives:
     \[ r_i d - s_i\alpha t_i - s_i\beta t_{i+1} \equiv -z_i. \]
   - Recurrence on tickets gives:
     \[ t_2 = T t_1 + D t_0,\quad t_3 = T t_2 + D t_1. \]
   - Stack into matrix form `A x = b (mod N)` with unknowns `x = (d, α, β, T, D)` and solve via modular Gaussian elimination.

3. **Forge the winning ticket:**
   
   - `target_tid = int.from_bytes(sha256(b"I'd like the flag please").digest(), "big")`
   - Pick small `k` (e.g., 1337), compute `R = kG`, `r = R.x mod N`, then
     \[ s \equiv k^{-1} (z + r d) \pmod N. \]
   - Emit base64 JSON with the forged `(r||s)` and submit via *Claim* to receive the flag.

---

## Math details

From ECDSA verification:
\[ s_i \equiv k_i^{-1}(z_i + r_i d) \Rightarrow r_i d - s_i k_i \equiv -z_i \ (\bmod N). \]
Substitute \( k_i = \alpha t_i + \beta t_{i+1} \). With three consecutive signatures you get 3 equations in `(d, α, β)`.
The ticket generator obeys a fixed 2nd‑order recurrence, hence two more equations determine `(T, D)`. Using all five equations makes the system well‑conditioned and avoids accidental singularities.

---

## Minimal solver (essentials only)

```python
from hashlib import sha256
import json, base64

# secp256k1
P  = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F
N  = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141
A, B = 0, 7
GX = 0x79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798
GY = 0x483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8
G  = (GX, GY)

def inv_mod(x, m): return pow(x % m, -1, m)

def ec_add(Pt, Qt):
    if Pt is None: return Qt
    if Qt is None: return Pt
    x1,y1 = Pt; x2,y2 = Qt
    if x1 == x2 and (y1 + y2) % P == 0: return None
    if Pt != Qt:
        lam = ((y2 - y1) * inv_mod((x2 - x1) % P, P)) % P
    else:
        lam = ((3*x1*x1 + A) * inv_mod((2*y1) % P, P)) % P
    x3 = (lam*lam - x1 - x2) % P
    y3 = (lam*(x1 - x3) - y1) % P
    return (x3, y3)

def ec_mul(k, Pt):
    if k % N == 0 or Pt is None: return None
    Q, addend = None, Pt
    while k:
        if k & 1: Q = ec_add(Q, addend)
        addend = ec_add(addend, addend)
        k >>= 1
    return Q

def gauss_solve_mod(A, b, mod):
    M = [row[:] + [bv] for row, bv in zip(A, b)]
    R, C = len(M), len(M[0]) - 1
    r, piv = 0, [-1]*C
    for c in range(C):
        sel = next((i for i in range(r, R) if M[i][c] % mod != 0), None)
        if sel is None: continue
        M[r], M[sel] = M[sel], M[r]
        inv = inv_mod(M[r][c] % mod, mod)
        for j in range(c, C+1): M[r][j] = (M[r][j]*inv) % mod
        for i in range(R):
            if i != r and M[i][c] % mod != 0:
                f = M[i][c] % mod
                for j in range(c, C+1): M[i][j] = (M[i][j] - f*M[r][j]) % mod
        piv[c] = r; r += 1
        if r == R: break
    x = [0]*C
    for c in range(C):
        if piv[c] != -1: x[c] = M[piv[c]][C] % mod
    return x

def solve_private_key(tickets):
    (t0,r0,s0,z0) = tickets[0]
    (t1,r1,s1,z1) = tickets[1]
    (t2,r2,s2,z2) = tickets[2]
    t3 = tickets[3][0]
    A, b = [], []
    A.append([ r0 % N, (-s0*t0) % N, (-s0*t1) % N, 0, 0 ]); b.append((-z0) % N)
    A.append([ r1 % N, (-s1*t1) % N, (-s1*t2) % N, 0, 0 ]); b.append((-z1) % N)
    A.append([ r2 % N, (-s2*t2) % N, (-s2*t3) % N, 0, 0 ]); b.append((-z2) % N)
    A.append([ 0, 0, 0,  t1 % N, (-t0) % N ]); b.append(t2 % N)
    A.append([ 0, 0, 0,  t2 % N, (-t1) % N ]); b.append(t3 % N)
    d, alpha, beta, T, D = gauss_solve_mod(A, b, N)
    return d

def forge_with_d(d):
    target_tid = int.from_bytes(sha256(b"I'd like the flag please").digest(), "big")
    payload = {"ticket_id": target_tid}
    pb = json.dumps(payload, separators=(',', ':'), sort_keys=True).encode()
    z = int.from_bytes(sha256(pb).digest(), "big") % N
    k = 1337
    R = ec_mul(k, G); r = R[0] % N
    s = (inv_mod(k, N) * (z + r * d)) % N
    sig_hex = (int(r).to_bytes(32, "big") + int(s).to_bytes(32, "big")).hex()
    return base64.b64encode(json.dumps({"payload": payload, "signature": sig_hex}, separators=(',', ':'), sort_keys=True).encode()).decode()
```

### Notes

- **Canonical JSON** matters: use `sort_keys=True` and compact separators to match server hashing.
- If a session doesn’t produce 4 consecutive tickets (three evens), just restart; success rate is ≈ 1/8.
- Rare ECDSA edge cases (e.g., `r=0`) are re‑sampled by the server; simply retry if the solve fails.

---

## Flag

After forging a ticket for `ticket_id = sha256("I'd like the flag please")`, submit it via *Claim* to receive:

```
FortID{W1nn3r_Winn3r_Ch1ck3n_D1nn3r_64277d4d7650896a}
```
