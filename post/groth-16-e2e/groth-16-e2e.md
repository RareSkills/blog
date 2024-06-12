# Example Groth16 End-to-End

Suppose we have a simple graph below that we wish to prove is bipartite (can be two-colored).

![Untitled](Example%20Groth16%20End-to-End%20a91850d868f24f0093ac46e33a4460a6/Untitled.png)

The color constraints are as follows (each node can be colored 1 or 2):

$$\displaylines{(x_1-1)(x_1-2)=0\\\ (x_2-1)(x_2-2)=0\\\ (x_3-1)(x_3-2)=0 \\\ (x_4-1)(x_4-2)=0}$$



The neighboring constraints are as follows:

$$x_1x_2-2=0\newline x_1x_4-2=0\\x_2x_3-2=0$$

We can arrange this as a systems of equations with only one multiplication on the left hand side:

$$
x_1x_1=3x_1-2\\x_2x_2=3x_2-2\\\\x_3x_3=3x_3-2\\x_4x_4=3x_4-2\\x_1x_2=2\\x_1x_4=2\\x_2x_3=2
$$

Assuming our columns are labeled $[1, x_1, x_2, x_3, x_4]$ then the R1CS will be the following:

$$
\small{\begin{bmatrix}0 & 1 & 0 & 0 & 0\\0 & 0 & 1 & 0 & 0\\0 & 0 & 0 & 1 & 0\\0 & 0 & 0 & 0 & 1\\0 & 1 & 0 & 0 & 0\\0 & 1 & 0 & 0 & 0\\0 & 0 & 1 & 0 & 0\end{bmatrix}\mathbf{a}\circ
\begin{bmatrix}0 & 1 & 0 & 0 & 0\\0 & 0 & 1 & 0 & 0\\0 & 0 & 0 & 1 & 0\\0 & 0 & 0 & 0 & 1\\0 & 0 & 1 & 0 & 0\\0 & 0 & 0 & 0 & 1\\0 & 0 & 0 & 1 & 0\end{bmatrix}\mathbf{a}=
\begin{bmatrix} -2 & 3 & 0 & 0 & 0\\-2 & 0 & 3 & 0 & 0\\-2 & 0 & 0 & 3 & 0\\-2 & 0 & 0 & 0 & 3\\2 & 0 & 0 & 0 & 0\\2 & 0 & 0 & 0 & 0\\2 & 0 & 0 & 0 & 0\end{bmatrix}\mathbf{a}}
$$

where **a** is the witness vector and ∘ is the Hadamard product.

Using the [code to convert an R1CS to a QAP](https://www.rareskills.io/post/r1cs-to-qap), we get the following set of polynomials for the 5 columns of **L**:

$$
\small{0}\\\small{0.01389 x^6 - 0.3333 x^5 + 3.139 x^4 - 14.75 x^3 + 36.35 x^2 - 44.42 x + 21}\\\small{-0.006944 x^6 + 0.1875 x^5 - 2.007 x^4 + 10.81 x^3 - 30.49 x^2 + 41.5 x - 20}\\\small{0.02083 x^6 - 0.5208 x^5 + 5.146 x^4 - 25.4 x^3 + 64.83 x^2 - 79.08 x + 35}\\\small{-0.02778 x^6 + 0.6667 x^5 - 6.278 x^4 + 29.33 x^3 - 70.69 x^2 + 82 x - 35}
$$

5 polynomials for the columns of R:

$$
\small{0}\\\small{0.001389 x^6 - 0.0375 x^5 + 0.4097 x^4 - 2.312 x^3 + 7.089 x^2 - 11.15 x + 7}\\\small{0.0125 x^6 - 0.2625 x^5 + 2.062 x^4 - 7.437 x^3 + 11.93 x^2 - 6.3 x}\\\small{0.02222 x^6 - 0.55 x^5 + 5.389 x^4 - 26.42 x^3 + 67.09 x^2 - 81.53 x + 36}
\\\small{-0.03611 x^6 + 0.85 x^5 - 7.861 x^4 + 36.17 x^3 - 86.1 x^2 + 98.98 x - 42}

$$

5 polynomials for the columns of O:

$$
\small{0.05556 x^6 - 1.3 x^5 + 11.89 x^4 - 53.83 x^3 + 126.1 x^2 - 142.9 x + 58}\\\small{0.004167 x^6 - 0.1125 x^5 + 1.229 x^4 - 6.938 x^3 + 21.27 x^2 - 33.45 x + 21}\\\small{-0.025 x^6 + 0.65 x^5 - 6.75 x^4 + 35.5 x^3 - 98.22 x^2 + 131.8 x - 63}\\\small{0.0625 x^6 - 1.562 x^5 + 15.44 x^4 - 76.19 x^3 + 194.5 x^2 - 237.2 x + 105}\\\small{-0.08333 x^6 + 2 x^5 - 18.83 x^4 + 88 x^3 - 212.1 x^2 + 246 x - 105}
$$

Here is the code:

```python
import numpy as np

L = np.array([
    [0, 1, 0, 0, 0],
    [0, 0, 1, 0, 0],
    [0, 0, 0, 1, 0],
    [0, 0, 0, 0, 1],
    [0, 1, 0, 0, 0],
    [0, 1, 0, 0, 0],
    [0, 0, 1, 0, 0]
])

R = np.array([
    [0, 1, 0, 0, 0],
    [0, 0, 1, 0, 0],
    [0, 0, 0, 1, 0],
    [0, 0, 0, 0, 1],
    [0, 0, 1, 0, 0],
    [0, 0, 0, 0, 1],
    [0, 0, 0, 1, 0]
])

O = np.array([
    [-2, 3, 0, 0, 0],
    [-2, 0, 3, 0, 0],
    [-2, 0, 0, 3, 0],
    [-2, 0, 0, 0, 3],
    [2, 0, 0, 0, 0],
    [2, 0, 0, 0, 0],
    [2, 0, 0, 0, 0]
])

x1 = 1
x2 = 2
x3 = 1
x4 = 2
a = np.array([1, x1, x2, x3, x4])

assert all(np.equal(np.matmul(L, a) * np.matmul(R, a), np.matmul(O, a))), "not equal"

from scipy.interpolate import lagrange

def interpolate_column(col):
    xs = np.array(range(1, len(col) + 1))
    return lagrange(xs, col)

print("L Columns")
for col in L.T:
    print(interpolate_column(col))

print("R Columns")
for col in R.T:
    print(interpolate_column(col))

print("O Columns")
for col in O.T:
    print(interpolate_column(col))
```

However, we use polynomials over a finite field instead, as floats are not compatible with elliptic curves. We will not show the polynomials done over the curve order of our elliptic curve since the numbers will be very large.

```python
import galois
from py_ecc.bn128 import curve_order
GF = galois.GF(curve_order) # this takes a while

# Adding the curve order converts the negative numbers
# into their additive inverses and leaves the non-negative
# numbers unchanged

# TODO: assert all elements of L, R, S are > -curve_order

L_galois = GF((L + curve_order) % curve_order)
R_galois = GF((R + curve_order) % curve_order)
O_galois = GF((O + curve_order) % curve_order)

a_galois = GF(a)

assert all(np.equal(np.matmul(L_galois, a_galois) * np.matmul(R_galois, a_galois), np.matmul(O_galois, a_galois))), "not equal"

def interpolate_column_galois(col):
    xs = GF(np.array(range(1, len(col) + 1)))
    return galois.lagrange_poly(xs, col)

U_polys_galois = np.apply_along_axis(interpolate_column_galois, 0, L_galois)
V_polys_galois = np.apply_along_axis(interpolate_column_galois, 0, R_galois)
W_polys_galois = np.apply_along_axis(interpolate_column_galois, 0, O_galois)
```

## Evaluating a polynomial as an inner product

A polynomial can be evaluated as the inner product of its coefficients and successive powers of the 𝑥.

For example, p(𝑥) = 5𝑥² - 3𝑥 + 1 can be evaulated as

$$
5x^2 -3x + 1=\langle [5, -3, 1], [x^2,x,1]\rangle
$$

The code below accomplishes the operation above.

```python
def inner_product(a, b):
    mul_ = lambda x, y: x * y
    sum_ = lambda x, y: x + y
    return reduce(sum_, map(mul_, a, b))
```

$$
\langle \mathbf{a}, \mathbf{u}\rangle= \sum_{i=1}^ma_iu_i(x)
$$

## Quadratic Arithmetic Program

$$
\sum_{i=1}^ma_iu_i(x)\sum_{i=1}^ma_iv_i(x)=\sum_{i=1}^ma_iw_i(x)+h(x)t(x)
$$

We can compute h(x) by rearranging the terms:

$$
\frac{\sum_{i=1}^ma_iu_i(x)\sum_{i=1}^ma_iv_i(x)-\sum_{i=1}^ma_iw_i(x)}{t(x)}=h(x)
$$

In our example, the matrices have 7 rows, so t(x) will be

$$
t(x)=(x-1)(x-2)(x-3)(x-4)(x-5)(x-6)(x-7)
$$

```python
sum_au = inner_product(U_polys_galois, a_galois)
sum_bu = inner_product(V_polys_galois, a_galois)
sum_cu = inner_product(W_polys_galois, a_galois)

t = galois.Poly([1, curve_order - 1], field = GF) * 
    galois.Poly([1, curve_order - 2], field = GF) *
    galois.Poly([1, curve_order - 3], field = GF) *
    galois.Poly([1, curve_order - 4], field = GF) * 
    galois.Poly([1, curve_order - 5], field = GF) * 
    galois.Poly([1, curve_order - 6], field = GF) *
    galois.Poly([1, curve_order - 7], field = GF)
    
h = (sum_au * sum_bu - sum_cu) // t
assert term_1 * term_2 == term_3 + h * t, "division has a remainder"
```

## Trusted Setup

We select secret values for (τ, α, β, γ, δ).

$$
ptau = [\tau^6 G_1, \tau^5 G_1, \tau^4 G_1, \tau^3 G_1, \tau^2 G_1, \tau G_1, G_1]
$$

$$
ptau = [\tau^6 G_2, \tau^5 G_2, \tau^4 G_2, \tau^3 G_2, \tau^2 G_2, \tau G_2, G_2]
$$

$$
t(\tau) = [\tau^6 t(\tau), \tau^5 t(\tau), \tau^4 t(\tau), \tau^3 t(\tau), \tau^2 t(\tau), \tau t(\tau) G_1, t(\tau) G_1]
$$

$$
[h_5 * \tau^5, h_4, h_3, h_2, h_1,h_0]
$$
