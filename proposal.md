## Proposal Details
This is a proposal to introduce intrinsic support for ARM64 SVE (Scalable Vector Extension) instructions. It is a child proposal of #73787. 

SVE is a recent architecture extension introduced to the ARM64 architecture. Its defining feature is a Vector Length Agnostic (VLA) programming model, which allows developers to write SIMD code once and have it scale automatically to the hardware's available vector length, much like standard scalar code. This proposal aims to provide a clean, accessible API that feels idiomatic to Go, mirrors the existing AMD64 `archsimd` API, and integrates smoothly with [Midway](https://github.com/golang/go/issues/78902).

This proposal only covers SVE and some SVE2. Specifically, loads from and stores to register lists are not supported. Each type supported will map to one `Z` or `P` register and we assume their length to be at most 256 bits and 64 bits. SVE2.1 and SME are not within the scope of this proposal.

## API Overview

### Types
Scalable vector types will be represented as `ElementType + "s"`. 
For example, a scalable vector of `int8` elements is typed as `Int8s`. For scalable predicates (masks), the naming convention is `"Mask" + TypeSize + "s"`, such as `Mask8s`.

```go
// Int8s is a scalable vector of int8s.
type Int8s struct {
	vals  [32]int8
}

// Int16s is a scalable vector of int16s.
type Int16s struct {
	vals  [16]int16
}

// Int32s is a scalable vector of int32s.
type Int32s struct {
	vals  [8]int32
}

// Int64s is a scalable vector of int64s.
type Int64s struct {
	vals  [4]int64
}

// Uint8s is a scalable vector of uint8s.
type Uint8s struct {
	vals  [32]uint8
}

// Uint16s is a scalable vector of uint16s.
type Uint16s struct {
	vals  [16]uint16
}

// Uint32s is a scalable vector of uint32s.
type Uint32s struct {
	vals  [8]uint32
}

// Uint64s is a scalable vector of uint64s.
type Uint64s struct {
	vals  [4]uint64
}

// Mask8s is a scalable predicate for 8-bit elements.
type Mask8s struct {
	vals   uint64
}

// Mask16s is a scalable predicate for 16-bit elements.
type Mask16s struct {
	vals   uint64
}

// Mask32s is a scalable predicate for 32-bit elements.
type Mask32s struct {
	vals   uint64
}

// Mask64s is a scalable predicate for 64-bit elements.
type Mask64s struct {
	vals   uint64
}
```
Because Go does not currently support dynamic stack allocations, scalable vectors and predicates are assumed to fit within a predefined maximum bound (currently set to 32 bytes for vector types and 8 bytes for predicate types).

All types also come with these utility functions:
`x.Len() int` Returns the number of elements in the vector (available on all vector types).
`x.String() string` Returns a string representation of the vector or mask (available on all vector and mask types).

ARM provides the `RDVL` instruction to read the hardware's actual vector length at runtime. When the `archsimd` package is imported, `RDVL` will be checked during initialization; if the hardware vector length exceeds 32 bytes, the package will panic. We believe 32 bytes (256 bits) covers the vast majority of SVE chips currently on the market (e.g., Neoverse V1). Please let us know if this constraint needs to be expanded.

### Memory Operations (Load and Stores)

#### Loads

* **`Load<Vector>Slice(s []elem) <Vector>`** Loads a full slice into a vector register.  
  * *Example*: `LoadInt8Slice([]int8) Int8s`  
* **`Load<Vector>SlicePart(s []elem) <Vector>`** Loads a partial slice into a vector register (e.g., when slice length is less than the register width).  
  * *Example*: `LoadInt8SlicePart([]int8) Int8s`  
* **`Load<Mask>(bits *uint64) <Mask>`**
  * *Asm*: `PLDR`
  Load a mask from a bitmask where the $i$-th bit represents the active state of the $i$-th element.

#### Stores

* **`x.StoreSlice(s []elem)`** Stores vector elements back into the destination slice `s`.  
  * *Example*: `(x Int8s) StoreSlice(s []int8)`  
* **`x.StoreSlicePart(s []elem)`** Stores a partial vector back into the destination slice `s`.  
  * *Example*: `(x Int8s) StoreSlicePart(s []int8)`
* **`x.Store(bits *uint64)`**
  * *Asm*: `PSTR`
  Store a mask to a bitmask where the $i$-th bit represents the active state of the $i$-th element.

### Mask Operations

#### Generation

* **`Mask<E>sFromCount(count int) Mask<E>s`**  
  Creates a mask where the first `count` elements are active (true), and the rest are inactive (false).  
  * *Example*: `Mask8sFromCount(x int) Mask8s`  
  * *Asm*: `PWHILELT` / `WHILELT`
* **`Mask<E>sAllTrue() Mask<E>s`**  
  Creates a mask where all elements are active.  
  * *Example*: `Mask8sAllTrue() Mask8s`  
  * *Asm*: `PTRUE`
* **`Mask<E>sAllFalse() Mask<E>s`**  
  Creates a mask where all elements are inactive.  
  * *Example*: `Mask8sAllFalse() Mask8s`  
  * *Asm*: `PFALSE`

*Note: `PTRUE` has more variants, we can support them later if needed.

#### Logic and Bitwise Operations

Element-wise logical operations between masks of the same element type.
* **`m.And(n Mask<E>s) Mask<E>s`** Bitwise AND between two masks.  
  * *Asm*: `PAND`
* **`m.Or(n Mask<E>s) Mask<E>s`** Bitwise OR.  
  * *Asm*: `PORR`
* **`m.Xor(n Mask<E>s) Mask<E>s`** Bitwise XOR.  
  * *Asm*: `PEOR`
* **`m.AndNot(n Mask<E>s) Mask<E>s`** Bitwise AND NOT (`m &^ n`).  
  * *Asm*: `PANDN`
* **`m.Not() Mask<E>s`** Bitwise invert mask.  
  * *Asm*: `PNOT`

#### Reductions

Querying properties of a mask.
* **`m.Any() bool`** Returns true if at least one element in the mask is active.  
  * *Asm*: `PTEST`
* **`m.All() bool`** Returns true if all elements in the mask are active.  
  * *Asm*: `PTEST`
* **`m.None() bool`** Returns true if no elements in the mask are active.  
  * *Asm*: `PTEST`
* **`m.CountActive() int`** Returns the number of active elements.  
  * *Asm*: `CNTP`

#### Conversions

SVE predicates are conceptually layout-identical bitmasks (1 bit per byte), but typed differently for lane width type-safety in Go. We support conversions between them:

##### Widen Elements
Unpack and widens (by padding 0 bits element-wise) a predicate of narrower elements (e.g., 8-bit) into wider elements (e.g., 16-bit).
* **`m.UnpackWidenLo() Mask<Wider>s`** Unpack and widen the low half of mask `m` to the next wider element size mask.
  * *Example*: `(m Mask8s) WidenLo() Mask16s`  
  * *Asm*: `PUNPKLO`
* **`m.UnpackWidenHi() Mask<Wider>s`** Unpack and widen the high half of mask `m` to the next wider element size mask.
  * *Example*: `(m Mask8s) WidenHi() Mask16s`  
  * *Asm*: `PUNPKHI`

##### Narrow Elements
Pack and narrow two masks (low and high halves) into a single mask for elements.
* **`lo.PackNarrow(hi Mask<Wider>s) Mask<Narrower>s`** Pack low and high masks of wider elements into a narrower element size mask.
  * *Example*: `(lo Mask16s) PackNarrow(hi Mask16s) Mask8s`  
  * *Asm*: `PUZP1`

*Note: While `PUZP1` can be used as narrowing operations, it can do interleaving predication too, which we will expose with a different signature (the operand and return types are the same in that op).*

##### Packing Elements
Pack two masks (odd or even elements) into a single mask for elements.
* **`lo.PackEven(hi <Mask>) <Mask>`** Pack the even-indexed elements of `lo` and `hi` into a single mask, in the lower half and upper half respectively.
  * *Example*: `(lo Mask8s) PackEven(hi Mask8s) Mask8s`  
  * *Asm*: `UZP1`
* **`lo.PackOdd(hi <Mask>) <Mask>`** Pack the odd-indexed elements of `lo` and `hi` into a single mask, in the lower half and upper half respectively.
  * *Example*: `(lo Mask8s) PackOdd(hi Mask8s) Mask8s`  
  * *Asm*: `UZP2`

#### Comparions Generating Masks

All scalable vector types support comparison operations that yield their corresponding mask type:
* **`x.Equal(y Vector) Mask`** Element-wise $x == y$.
* **`x.NotEqual(y Vector) Mask`** Element-wise $x \neq y$.
* **`x.Greater(y Vector) Mask`** Element-wise $x > y$.
* **`x.GreaterEqual(y Vector) Mask`** Element-wise $x \ge y$.
* **`x.Less(y Vector) Mask`** Element-wise $x < y$.
* **`x.LessEqual(y Vector) Mask`** Element-wise $x \le y$.
  * *Example*: `(x Int8s) Greater(y Int8s) Mask8s`  
  * *Asm*: `ZCMPGT`, `ZCMPEQ`, `ZCMPGE`, `ZCMPLE`, etc.

### Vector Operations

All SVE vector types support a rich set of vector operations. To leverage SVE's hardware predication, operations can be seamlessly governed by a mask.

#### 1. Element-wise Arithmetic
* **`x.Add(y Vector) Vector`** element-wise $x + y$. (Asm: `ZADD`)
* **`x.Sub(y Vector) Vector`** element-wise $x - y$. (Asm: `ZSUB`)
* **`x.Mul(y Vector) Vector`** element-wise $x \times y$. (Asm: `ZMUL`)
* **`x.Div(y Vector) Vector`** element-wise $x / y$ (floating-point types only). (Asm: `ZFDIV`)
* **`x.Min(y Vector) Vector`** element-wise minimum. (Asm: `SMIN` / `UMIN` / `FMIN`)
* **`x.Max(y Vector) Vector`** element-wise maximum. (Asm: `SMAX` / `UMAX` / `FMAX`)
* **`x.Abs() Vector`** element-wise absolute value. (Asm: `SABS` / `FABS`)
* **`x.Neg() Vector`** element-wise negation. (Asm: `SNEG` / `FNEG`)
* **`x.Sqrt() Vector`** element-wise square root (floating-point types only). (Asm: `FSQRT`)

#### 2. Bitwise Logic
* **`x.And(y Vector) Vector`** bitwise $x \ \& \ y$. (Asm: `AND`)
* **`x.Or(y Vector) Vector`** bitwise $x \ | \ y$. (Asm: `ORR`)
* **`x.Xor(y Vector) Vector`** bitwise $x \ \text{xor} \ y$. (Asm: `EOR`)
* **`x.AndNot(y Vector) Vector`** bitwise $x \ \& \ \sim y$. (Asm: `BIC`)
* **`x.Not() Vector`** bitwise inversion. (Asm: `NOT`)

#### 3. Shifts (Integer types only)
* **`x.ShiftLeft(bits uint) Vector`** shifts lanes left. (Asm: `LSL`)
* **`x.ShiftRightLogical(bits uint) Vector`** logical shift right. (Asm: `LSR`)
* **`x.ShiftRightArithmetic(bits uint) Vector`** arithmetic shift right. (Asm: `ASR`)

#### 4. Type Conversions and Extensions
SVE supports sign/zero extension and truncation:
* **`x.ExtendLo() <WiderVector>`** Widens and sign/zero extends the low half of vector `x` to the next wider vector type. (Asm: `SUNPKLO` / `UUNPKLO`)
* **`x.ExtendHi() <WiderVector>`** Widens and sign/zero extends the high half of vector `x` to the next wider vector type. (Asm: `SUNPKHI` / `UUNPKHI`)
* **`lo.Pack(hi Vector) <NarrowerVector>`** Packs two wider vectors `lo` and `hi` into a narrower vector type. (Asm: `UZP1`)

#### 5. Horizontal Reductions
Reductions compute a scalar value across all lanes of a scalable vector:
* **`x.Sum() elem`** Computes the sum of all active lanes. (Asm: `SADDV` / `UADDV` / `FADDV`)
* **`x.Min() elem`** Finds the minimum value among active lanes. (Asm: `SMINV` / `UMINV` / `FMINV`)
* **`x.Max() elem`** Finds the maximum value among active lanes. (Asm: `SMAXV` / `UMAXV` / `FMAXV`)

#### 6. Fluent Predication and Peephole Optimization
Nearly all SVE arithmetic, logic, and memory instructions can be governed by a predicate. To provide a clean, idiomatic Go API without doubling the method count (e.g., avoiding `AddMasked`, `SubMasked`, etc.), we propose a fluent masking pattern:
* **`x.Masked(m Mask) Vector`** Returns a vector with elements of `x` where mask `m` is active, and `0` otherwise (Zeroing Predication).
* **`x.Merge(y Vector, m Mask) Vector`** Returns `x` where mask `m` is active, and elements of `y` where `m` is inactive (Merging Predication).

**Peephole Compiler Lowering**:  
When the Go compiler encounters an unmasked operation followed immediately by a `.Masked(m)` or `.Merge(y, m)` call, it will optimize it and generate a single predicated instruction on SVE.
* *Example*: `xv.Add(yv).Masked(p)` lowers to a single predicated SVE `ADD` instruction with zeroing predication (`ADD Z0.B, P0/Z, Z0.B, Z1.B`).
* *Example*: `xv.Add(yv).Merge(zv, p)` lowers to a single SVE `ADD` instruction with merging predication (`ADD Z0.B, P0/M, Z0.B, Z1.B`).

## Example
Below is an example test demonstrating a Vector Length Agnostic (VLA) loop that adds two slices of `int8`s together. SVE allows the loop stride to safely scale to the hardware's vector length while automatically masking the tail end of the slice.

```go
func AddSlice(x, y []int8) []int8 {
	// pick any number that's smaller than VL will work.
	// we use 5 here as an example.
	stride := min(5, len(x), len(y))
	commonLen := min(len(x), len(y))
	res := make([]int8, commonLen)
	for i := 0; i < commonLen; i += stride {
		xv := archsimd.LoadInt8sPart(x[i:])
		yv := archsimd.LoadInt8sPart(y[i:])
		zv := xv.Add(yv)
		zv.StorePart(res[i:])
	}
	return res
}
```