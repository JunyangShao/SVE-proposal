## Proposal Details
This is a proposal to introduce intrinsic support for ARM64 SVE (Scalable Vector Extension) instructions. It is a child proposal of #73787. 

SVE is a recent architecture extension introduced to the ARM64 architecture. Its defining feature is a Vector Length Agnostic (VLA) programming model, which allows developers to write SIMD code once and have it scale automatically to the hardware's available vector length, much like standard scalar code. This proposal aims to provide a clean, accessible API that feels idiomatic to Go, mirrors the existing AMD64 `archsimd` API, and integrates smoothly with [Midway](https://github.com/golang/go/issues/78902).

This proposal only covers SVE and some SVE2. Specifically, loads from and stores to register lists are not supported. Each type supported will map to one `Z` or `P` register and we assume their length to be at most 256 bits and 64 bits. As a result, `PN` registers are also not supported. SVE2.1 and SME are not within the scope of this proposal.

## API Overview

### Types
Scalable vector types will be represented as `ElementType + "s"`. 
For example, a scalable vector of `int8` elements is typed as `Int8s`. For scalable predicates (masks), the naming convention is `"Mask" + TypeSize + "s"`, such as `Mask8s`.

| Element Type | Bit Width | Vector Type | Mask Type |
| :---- | :---: | :---- | :---- |
| **Signed Integer** | 8-bit | `Int8s` | `Mask8s` |
|  | 16-bit | `Int16s` | `Mask16s` |
|  | 32-bit | `Int32s` | `Mask32s` |
|  | 64-bit | `Int64s` | `Mask64s` |
| **Unsigned Integer** | 8-bit | `Uint8s` | `Mask8s` |
|  | 16-bit | `Uint16s` | `Mask16s` |
|  | 32-bit | `Uint32s` | `Mask32s` |
|  | 64-bit | `Uint64s` | `Mask64s` |
| **Floating-Point** | 32-bit | `Float32s` | `Mask32s` |
|  | 64-bit | `Float64s` | `Mask64s` |

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

* **`<Mask>FromCount(count int) <Mask>`**  
  Creates a mask where the first `count` elements are active (true), and the rest are inactive (false).  
  * *Example*: `Mask8sFromCount(x int) Mask8s`  
  * *Asm*: `PWHILELT`
* **`<Mask>AllTrue() <Mask>`**  
  Creates a mask where all elements are active.  
  * *Example*: `Mask8sAllTrue() Mask8s`  
  * *Asm*: `PTRUE`
* **`<Mask>AllFalse() <Mask>`**  
  Creates a mask where all elements are inactive.  
  * *Example*: `Mask8sAllFalse() Mask8s`  
  * *Asm*: `PFALSE`

*Note: `PTRUE` has more variants, we can support them later if needed.

#### Logic and Bitwise Operations

Element-wise logical operations between masks of the same element type.
* **`m.And(n <Mask>) <Mask>`** Bitwise AND between two masks.  
  * *Asm*: `PAND`
* **`m.Or(n <Mask>) <Mask>`** Bitwise OR.  
  * *Asm*: `PORR`
* **`m.Xor(n <Mask>) <Mask>`** Bitwise XOR.  
  * *Asm*: `PEOR`

#### Reductions

* **`m.CountActive() int`** Returns the number of active elements.  
  * *Asm*: `CNTP`

#### Conversions

SVE predicates are conceptually layout-identical bitmasks (1 bit per byte), but typed differently for lane width type-safety in Go. We support conversions between them:

##### Widen Elements
Unpack and widens (by padding 0 bits element-wise) a predicate of narrower elements (e.g., 8-bit) into wider elements (e.g., 16-bit).
* **`m.UnpackWidenLo() <MaskWider>`** Unpack and widen the low half of mask `m` to the next wider element size mask.
  * *Example*: `(m Mask8s) WidenLo() Mask16s`  
  * *Asm*: `PUNPKLO`
* **`m.UnpackWidenHi() <MaskWider>`** Unpack and widen the high half of mask `m` to the next wider element size mask.
  * *Example*: `(m Mask8s) WidenHi() Mask16s`  
  * *Asm*: `PUNPKHI`

##### Narrow Elements
Pack and narrow two masks (low and high halves) into a single mask for elements.
* **`lo.PackNarrow(hi <MaskWider>) <MaskNarrower>`** Pack low and high masks of wider elements into a narrower element size mask.
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
* **`x.Equal(y <Vector>) Mask`** Element-wise $x == y$.
* **`x.NotEqual(y <Vector>) Mask`** Element-wise $x \neq y$.
* **`x.Greater(y <Vector>) Mask`** Element-wise $x > y$.
* **`x.GreaterEqual(y <Vector>) Mask`** Element-wise $x \ge y$.
* **`x.Less(y <Vector>) Mask`** Element-wise $x < y$.
* **`x.LessEqual(y <Vector>) Mask`** Element-wise $x \le y$.
  * *Example*: `(x Int8s) Greater(y Int8s) Mask8s`  
  * *Asm*: `ZCMPGT`, `ZCMPEQ`, `ZCMPGE`, `ZCMPLE`, etc.

### Vector Operations

All SVE vector types support a rich set of vector operations. To leverage SVE's hardware predication, operations can be seamlessly governed by a mask.

#### 1. Element-wise Arithmetic
* **`x.Add(y <Vector>) <Vector>`** element-wise $x + y$. (Asm: `ZADD` / `ZFADD`)
* **`x.Sub(y <Vector>) <Vector>`** element-wise $x - y$. (Asm: `ZSUB`/ `ZFSUB`)
* **`x.Mul(y <Vector>) <Vector>`** element-wise $x \times y$. (Asm: `ZMUL` / `ZFMUL`)
* **`x.Div(y <Vector>) <Vector>`** element-wise $x / y$ (floating-point types only). (Asm: `ZSDIV`, `ZUDIV`, `ZFDIV`)
* **`x.Min(y <Vector>) <Vector>`** element-wise minimum. (Asm: `ZSMIN` / `ZUMIN` / `ZFMIN`)
* **`x.Max(y <Vector>) <Vector>`** element-wise maximum. (Asm: `ZSMAX` / `ZUMAX` / `ZFMAX`)
* **`x.Abs() <Vector>`** element-wise absolute value. (Asm: `ZABS` / `ZFABS`)
* **`x.Neg() <Vector>`** element-wise negation. (Asm: `ZNEG` / `ZFNEG`)
* **`x.Sqrt() <Vector>`** element-wise square root (floating-point types only). (Asm: `ZFSQRT`)

#### 2. Bitwise Logic
* **`x.And(y <Vector>) <Vector>`** bitwise $x \\& y$. (Asm: `ZAND`)
* **`x.Or(y <Vector>) <Vector>`** bitwise $x | y$. (Asm: `ZORR`)
* **`x.Xor(y <Vector>) <Vector>`** bitwise $x \oplus \ y$. (Asm: `ZEOR`)

#### 3. Shifts (Integer types only)
* **`x.ShiftAllLeft(shift uint64) <Vector>`** shifts all elements left. (Asm: `ZLSL`)
* **`x.ShiftLeft(y <Vector>) <Vector>`** shifts elements left. (Asm: `ZLSL`)
* **`x.ShiftAllRight(shift uint64) <Vector>`** shifts all elements right. (Asm: `ZLSL` / `ZASR`)
* **`x.ShiftRight(y <Vector>) <Vector>`** shifts elements right. (Asm: `ZLSL` / `ZASR`)

#### 4. Type Conversions and Extensions
SVE supports sign/zero extension and truncation:
* **`x.ExtendLo() <WiderVector>`** Widens and sign/zero extends the low half of vector `x` to the next wider vector type. (Asm: `SUNPKLO` / `UUNPKLO`)
* **`x.ExtendHi() <WiderVector>`** Widens and sign/zero extends the high half of vector `x` to the next wider vector type. (Asm: `SUNPKHI` / `UUNPKHI`)
* **`lo.Pack(hi <Vector>) <NarrowerVector>`** Packs two wider vectors `lo` and `hi` into a narrower vector type. (Asm: `UZP1`)

#### 5. Horizontal Reductions
Reductions compute a scalar value across all elements of a scalable vector:
* **`x.Sum() elem`** Computes the sum of all active elements. (Asm: `SADDV` / `UADDV` / `FADDV`)
* **`x.Min() elem`** Finds the minimum value among active elements. (Asm: `SMINV` / `UMINV` / `FMINV`)
* **`x.Max() elem`** Finds the maximum value among active elements. (Asm: `SMAXV` / `UMAXV` / `FMAXV`)

#### 6. Fluent Predication and Peephole Optimization
Nearly all SVE arithmetic, logic, and memory instructions can be governed by a predicate. To provide a clean, idiomatic Go API without doubling the method count (e.g., avoiding `AddMasked`, `SubMasked`, etc.), we propose a fluent masking pattern:
* **`x.Masked(m Mask) Vector`** Returns a vector with elements of `x` where mask `m` is active, and `0` otherwise (Zeroing Predication).
* **`x.IfElse(y Vector, m Mask) Vector`** Returns `x` where mask `m` is active, and elements of `y` where `m` is inactive (Merging Predication).

**Peephole Compiler Lowering**:  
When the Go compiler encounters an unmasked operation followed immediately by a `.Masked(m)` or `.IfElse(y, m)` call, it will optimize it and generate a single predicated instruction on SVE.
* *Example*: `xv.Add(yv).Masked(p)` lowers to a single predicated SVE `ADD` instruction with zeroing predication (`ADD Z0.B, P0/Z, Z0.B, Z1.B`).
* *Example*: `xv.Add(yv).IfElse(zv, p)` lowers to a single SVE `ADD` instruction with merging predication (`ADD Z0.B, P0/M, Z0.B, Z1.B`).

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