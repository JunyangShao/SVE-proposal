## Proposal Details
This is a proposal to introduce intrinsic support for ARM64 SVE (Scalable Vector Extension) instructions. It is a child proposal of #73787.

SVE is a recent architecture extension introduced to the ARM64 architecture. Its defining feature is a Vector Length Agnostic (VLA) programming model, which allows developers to write SIMD code once and have it scale automatically to the hardware's available vector length, much like standard scalar code. This proposal aims to provide a clean, accessible API that feels idiomatic to Go, mirrors the existing AMD64 `archsimd` API where semantics overlap, and integrates smoothly with [Midway](https://github.com/golang/go/issues/78902).

This proposal only covers SVE and some SVE2. Specifically, loads from and stores to register lists are not supported. Each type supported will map to one `Z` or `P` register and we assume their length to be at most 256 bits and 32 bits. As a result, `PN` registers are also not supported. SVE2.1 and SME are not within the scope of this proposal.

### Naming alignment

Where an operation has a direct semantic counterpart in `simd/archsimd` (the AMD64 API in #73787), this proposal uses the same method name (`Add`, `Sub`, `Mul`, `Min`, `Max`, `Sqrt`, `Abs`, `Neg`, `And`, `Or`, `Xor`, `AndNot`, `Not`, `ShiftLeft`/`ShiftRight`, `RotateLeft`/`RotateRight`, `AddSaturated`/`SubSaturated`, `MulAdd`, `OnesCount`, `LeadingZeros`, `Equal`/`NotEqual`/`Less`/`LessEqual`/`Greater`/`GreaterEqual`, `Masked`, `Merge`, etc.). The element/vector type names follow the Midway-style plural convention (`Int8s`, `Float32s`, ...) because that matches SVE's length-agnostic nature. Signatures use Midway types throughout so that user code can switch between architectures with minimal surface change.

## API Overview

### Types
Scalable vector types use `ElementType + "s"`. For example, a scalable vector of `int8` elements is typed as `Int8s`. For scalable predicates (masks), the naming convention is `"Mask" + LaneBitWidth + "s"`, such as `Mask8s`.

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

**No 16-bit floats (yet).** SVE hardware supports half-precision (`fp16`) and brain-float (`bf16`) lanes, but Go has no `float16` / `bfloat16` primitive scalar type, and we want the SVE vector element types in `archsimd` to remain a one-to-one mapping to Go's primitive scalar types. If/when Go gains those scalar types, `Float16s` and `BFloat16s` (with mask `Mask16s`) can be added without disturbing the rest of this surface.

Because Go does not currently support dynamic stack allocations, scalable vectors and predicates are assumed to fit within a predefined maximum bound (currently set to 32 bytes for vector types and 8 bytes for predicate types).

All types come with these utility methods:

```go
func (x <Vector>) Len() int        // number of elements (vector types)
func (x <Vector>) String() string  // human-readable form (vector and mask types)
```

ARM provides the `RDVL` instruction to read the hardware's actual vector length at runtime. When the `archsimd` package is imported, `RDVL` will be checked during initialization; if the hardware vector length exceeds 32 bytes, the package will panic. We believe 32 bytes (256 bits) covers the vast majority of SVE chips currently on the market (e.g., Neoverse V1). Please let us know if this constraint needs to be expanded.

### Memory Operations (Loads and Stores)

For names that are already specified in Midway, unless documented in comment, they have the same semantic as [Midway](https://github.com/golang/go/issues/78902). `// Asm` documents their Arm64 instruction title from the spec table.

#### Vector Loads

Consecutive loads:

```go
func LoadInt8s(s []int8) Int8s // Asm: LD1B (scalar plus immediate, single register)
func LoadInt8sPart(s []int8) Int8s // Asm: LD1B (scalar plus immediate, single register)
// ... analogous for all other vector types
```

Gather loads (from a slice plus a vector of indices):

```go
// GatherInt8sPart gathers value into the result vector.
// result[i] = base[idx[i]].
// Out of bound elements will be zeroed.
//
// Asm: LD1B (scalar plus vector)
func (idx Int8s) GatherInt8sPart(base []int8) Int8s
// ... analogous for all other vector types
```

#### Vector Stores

Consecutive stores:

```go
func (x Int8s) Store(s []int8) // Asm: ST1B (scalar plus immediate, single register)
func (x Int8s) StorePart(s []int8) // Asm: ST1B (scalar plus immediate, single register)
// ... analogous for all other vector types
```

Scatter stores:

```go
// ScatterInt8sPart stores value into a slice.
// base[idx[i]] = [x[i]].
// Out of bound elements will be skipped
//
// Asm: ST1B (scalar plus vector)
func (x Int8s) ScatterInt8sPart(idx Int8s, base []int8)
// ... analogous for all other vector types
```

#### Mask Loads and Stores
A predicate is loaded from / stored to a `*uint32` bitmask, where bit `i` corresponds to lane `i`'s active state:

```go
func LoadMask8s(bits *uint32) Mask8s // Asm: LDR (predicate)
func (m Mask8s) Store(bits *uint32) // Asm: STR (predicate)
// ... analogous for all other mask type
```

### Mask Operations

#### Generation

```go
// Mask8sFromCount returns a mask that activates the first count elements.
//
// Asm: WHILELO (predicate)
func Mask8sFromCount(count int) Mask8s
// Mask8sAllTrue returns a mask that has all its 
//
// Asm: PTRUE (predicate)
func Mask8sAllTrue() Mask8s

// We don't need a Mask8sAllFalse, which is PFALSE (predicate), that would be the zero value of P.

// First returns a mask that activate only the first active element of m.
//
// Asm: PFIRST
func (m Mask8s) First() Mask8s
// Next returns a mask that activate the next element // TODO: finish this
func (m Mask8s) Next() Mask8s // next-active under governing predicate. Asm: PNEXT
// ... analogous for all other mask type

```

*Note:* `PTRUE` has more variants (e.g., a fixed power-of-two count). They can be exposed later if useful.

#### Logic and Bitwise Operations
Element-wise logical operations between masks of the same lane width.

```go
func (m Mask8s) And(n Mask8s) Mask8s        // Asm: AND (predicate)
func (m Mask8s) Or(n Mask8s) Mask8s         // Asm: ORR (predicate)
func (m Mask8s) Xor(n Mask8s) Mask8s        // Asm: EOR (predicate)
func (m Mask8s) AndNot(n Mask8s) Mask8s     // Asm: BIC (predicate)
func (m Mask8s) Not() Mask8s                // Asm: NOT (predicate)
// ... analogous for all other mask type
```

#### Tests and Reductions

```go
// CountActive returns the number of active elements of m
//
// Asm: CNTP (predicate)
func (m Mask8s) CountActive() int
// FirstIsActive returns true if the first elements in m is active.
//
// Asm: PTEST
func (m Mask8s) FirstIsActive() bool
// LastIsActive returns true if the last elements in m is active.
//
// Asm: PTEST
func (m Mask8s) LastIsActive() bool
// ... analogous for wider masks
```

*Note: FIRSTP looks like a useful instruction, however it's SVE2, should we support it?*

#### Conversions
SVE predicates are layout-identical bitmasks (1 bit per byte), but are typed by lane width in Go for type safety. Conversions reshape the lane width without changing the underlying bits.

##### Widen Elements
Unpack and widen (by padding 0 bits) a predicate of narrower lanes into wider lanes.

```go
func (m Mask8s) UnpackWidenLo() Mask16s     // Asm: PUNPKLO
func (m Mask8s) UnpackWidenHi() Mask16s     // Asm: PUNPKHI
func (m Mask16s) UnpackWidenLo() Mask32s
func (m Mask16s) UnpackWidenHi() Mask32s
func (m Mask32s) UnpackWidenLo() Mask64s
func (m Mask32s) UnpackWidenHi() Mask64s
```

##### Narrow Elements
Pack two wider-lane masks (low/high halves) into a single narrower-lane mask.

```go
func (lo Mask16s) PackNarrow(hi Mask16s) Mask8s     // Asm: UZP1 (predicate form)
func (lo Mask32s) PackNarrow(hi Mask32s) Mask16s
func (lo Mask64s) PackNarrow(hi Mask64s) Mask32s
```

*Note:* `UZP1` on predicates can also do interleaved deinterleaving when operand and result types are the same; that is exposed below under `PackEven`/`PackOdd`.

##### Same-width Packing
Pack the even- or odd-indexed lanes of two masks into a single mask.

```go
func (lo Mask8s) PackEven(hi Mask8s) Mask8s         // Asm: UZP1 (predicate form)
func (lo Mask8s) PackOdd(hi Mask8s) Mask8s          // Asm: UZP2 (predicate form)
// ... analogous for Mask16s, Mask32s, Mask64s
```

#### Comparisons Producing Masks
All scalable vector types support element-wise comparisons that yield the corresponding mask type:

```go
func (x Int8s) Equal(y Int8s) Mask8s            // Asm: CMPEQ
func (x Int8s) NotEqual(y Int8s) Mask8s         // Asm: CMPNE
func (x Int8s) Greater(y Int8s) Mask8s          // Asm: CMPGT
func (x Int8s) GreaterEqual(y Int8s) Mask8s     // Asm: CMPGE
func (x Int8s) Less(y Int8s) Mask8s             // Asm: CMPLT
func (x Int8s) LessEqual(y Int8s) Mask8s        // Asm: CMPLE
// ... analogous for all other integer / float vector types
```

Floating-point types additionally provide:

```go
func (x Float32s) IsNaN() Mask32s               // Asm: FCMUO  Zd, Pg/Z, Zn, Zn
func (x Float64s) IsNaN() Mask64s
```

### Vector Operations
All SVE vector operations are presented in their unconditional form. Predication is applied through the fluent `Masked` / `Merge` chaining described at the end of this section, and the compiler folds the chain into a single predicated instruction.

#### 1. Element-wise Arithmetic

```go
func (x Int8s) Add(y Int8s) Int8s        // Asm: ADD            (FADD for floats)
func (x Int8s) Sub(y Int8s) Int8s        // Asm: SUB            (FSUB for floats)
func (x Int8s) Mul(y Int8s) Int8s        // Asm: MUL            (FMUL for floats)
func (x Int8s) Neg() Int8s               // Asm: NEG            (FNEG for floats)
func (x Int8s) Abs() Int8s               // Asm: ABS            (FABS for floats)
func (x Int8s) Min(y Int8s) Int8s        // Asm: SMIN/UMIN/FMIN
func (x Int8s) Max(y Int8s) Int8s        // Asm: SMAX/UMAX/FMAX
// ... analogous for all other vector types
```

Division and square root are float-only (integer SDIV/UDIV exist only for 32-/64-bit lanes in SVE and are listed separately):

```go
func (x Float32s) Div(y Float32s) Float32s            // Asm: FDIV
func (x Float32s) Sqrt() Float32s                     // Asm: FSQRT
func (x Float32s) Reciprocal() Float32s               // Asm: FRECPE + FRECPS refinement
func (x Float32s) ReciprocalSqrt() Float32s           // Asm: FRSQRTE + FRSQRTS refinement
// ... analogous for Float64s
```

Integer division (32- and 64-bit only):

```go
func (x Int32s) Div(y Int32s) Int32s        // Asm: SDIV
func (x Uint32s) Div(y Uint32s) Uint32s     // Asm: UDIV
func (x Int64s) Div(y Int64s) Int64s        // Asm: SDIV
func (x Uint64s) Div(y Uint64s) Uint64s     // Asm: UDIV
```

Multiply-high (upper half of a widening multiply, without widening the result type):

```go
func (x Int8s) MulHigh(y Int8s) Int8s       // Asm: SMULH
func (x Uint8s) MulHigh(y Uint8s) Uint8s    // Asm: UMULH
// ... analogous for 16/32/64-bit integer lanes
```

Fused multiply-add (single-rounding):

```go
func (x Float32s) MulAdd(y, z Float32s) Float32s    // x*y + z. Asm: FMLA
func (x Float32s) MulSub(y, z Float32s) Float32s    // x*y - z. Asm: FMLS
// ... analogous for Float64s
```

#### 2. Saturating Arithmetic (integer types)

```go
func (x Int8s) AddSaturated(y Int8s) Int8s    // Asm: SQADD
func (x Uint8s) AddSaturated(y Uint8s) Uint8s // Asm: UQADD
func (x Int8s) SubSaturated(y Int8s) Int8s    // Asm: SQSUB
func (x Uint8s) SubSaturated(y Uint8s) Uint8s // Asm: UQSUB
// ... analogous for 16/32/64-bit integer lanes
```

#### 3. Bitwise Logic (integer types)

```go
func (x Int8s) And(y Int8s) Int8s        // Asm: AND
func (x Int8s) Or(y Int8s) Int8s         // Asm: ORR
func (x Int8s) Xor(y Int8s) Int8s        // Asm: EOR
func (x Int8s) AndNot(y Int8s) Int8s     // x AND NOT y. Asm: BIC
func (x Int8s) Not() Int8s               // Asm: NOT
// ... analogous for all other integer vector types
```

#### 4. Shifts and Rotations (integer types)
Vector-by-scalar (single shift amount applied to every lane), and vector-by-vector (per-lane shift amount) forms. Right shift is logical for unsigned lanes and arithmetic for signed lanes.

```go
func (x Int8s) ShiftAllLeft(shift uint64) Int8s     // Asm: LSL (by scalar)
func (x Int8s) ShiftLeft(y Uint8s) Int8s            // Asm: LSL (by vector)
func (x Int8s) ShiftAllRight(shift uint64) Int8s    // Asm: ASR (by scalar)
func (x Int8s) ShiftRight(y Uint8s) Int8s           // Asm: ASR (by vector)

func (x Uint8s) ShiftAllRight(shift uint64) Uint8s  // Asm: LSR (by scalar)
func (x Uint8s) ShiftRight(y Uint8s) Uint8s         // Asm: LSR (by vector)
// ... analogous for 16/32/64-bit integer lanes
```

Rotations (SVE2):

```go
func (x Int8s) RotateAllRight(shift uint64) Int8s   // Asm: XAR (#imm form, SVE2)
func (x Int8s) RotateRight(y Uint8s) Int8s          // (composed with LSR/LSL+ORR if no single XAR encoding)
// ... analogous for 16/32/64-bit integer lanes
```

#### 5. Bit Manipulation (integer types)

```go
func (x Uint8s) OnesCount() Uint8s           // popcount per lane. Asm: CNT
func (x Uint8s) LeadingZeros() Uint8s        // count leading zeros. Asm: CLZ
func (x Uint8s) ReverseBits() Uint8s         // reverse bits within each lane. Asm: RBIT
func (x Uint16s) ReverseBytes() Uint16s      // reverse bytes within each lane. Asm: REVB
func (x Uint32s) ReverseBytes() Uint32s      // Asm: REVB
func (x Uint64s) ReverseBytes() Uint64s      // Asm: REVB
// ... analogous for signed lanes
```

#### 6. Floating-Point Rounding

```go
func (x Float32s) RoundToEven() Float32s   // banker's rounding. Asm: FRINTN
func (x Float32s) Ceil() Float32s          // Asm: FRINTP
func (x Float32s) Floor() Float32s         // Asm: FRINTM
func (x Float32s) Trunc() Float32s         // Asm: FRINTZ
// ... analogous for Float64s
```

#### 7. Type Conversions and Lane-Width Changes

##### Reinterpret (no instruction; bit-cast)

```go
func (x Int8s) AsUint8s() Uint8s
func (x Uint8s) AsInt8s() Int8s
func (x Int32s) AsFloat32s() Float32s
func (x Float32s) AsInt32s() Int32s
// ... analogous for all same-width pairs
```

##### Widen (sign- or zero-extend lanes)

```go
func (x Int8s) UnpackWidenLo() Int16s      // Asm: SUNPKLO
func (x Int8s) UnpackWidenHi() Int16s      // Asm: SUNPKHI
func (x Uint8s) UnpackWidenLo() Uint16s    // Asm: UUNPKLO
func (x Uint8s) UnpackWidenHi() Uint16s    // Asm: UUNPKHI
// ... analogous for Int16s↔Int32s, Int32s↔Int64s, Uint16s↔Uint32s, Uint32s↔Uint64s
```

##### Narrow (truncate to half-width lanes)

```go
func (lo Int16s) PackNarrow(hi Int16s) Int8s       // Asm: UZP1
func (lo Uint16s) PackNarrow(hi Uint16s) Uint8s    // Asm: UZP1
// ... analogous for 32→16 and 64→32 lanes
```

Saturating narrowing (SVE2):

```go
func (lo Int16s) PackNarrowSaturated(hi Int16s) Int8s          // Asm: SQXTNB+SQXTNT
func (lo Uint16s) PackNarrowSaturated(hi Uint16s) Uint8s       // Asm: UQXTNB+UQXTNT
func (lo Int16s) PackNarrowSaturatedUnsigned(hi Int16s) Uint8s // Asm: SQXTUNB+SQXTUNT
// ... analogous for 32→16 and 64→32 lanes
```

##### Integer ↔ Float

```go
func (x Int32s) ConvertToFloat32s() Float32s         // Asm: SCVTF
func (x Uint32s) ConvertToFloat32s() Float32s        // Asm: UCVTF
func (x Float32s) ConvertToInt32s() Int32s           // round-to-zero. Asm: FCVTZS
func (x Float32s) ConvertToUint32s() Uint32s         // round-to-zero. Asm: FCVTZU
func (x Int64s) ConvertToFloat64s() Float64s         // Asm: SCVTF
func (x Uint64s) ConvertToFloat64s() Float64s        // Asm: UCVTF
func (x Float64s) ConvertToInt64s() Int64s           // Asm: FCVTZS
func (x Float64s) ConvertToUint64s() Uint64s         // Asm: FCVTZU
```

Cross-width float conversions:

```go
func (x Float32s) UnpackWidenLoToFloat64s() Float64s   // Asm: FCVT (widen)
func (x Float32s) UnpackWidenHiToFloat64s() Float64s
func (lo Float64s) PackNarrowToFloat32s(hi Float64s) Float32s // Asm: FCVT (narrow)
```

#### 8. Horizontal Reductions
Compute a scalar value across all active lanes of a scalable vector:

```go
func (x Int8s) Sum() int8         // Asm: SADDV (UADDV / FADDV)
func (x Int8s) Min() int8         // Asm: SMINV (UMINV / FMINV)
func (x Int8s) Max() int8         // Asm: SMAXV (UMAXV / FMAXV)
func (x Int8s) And() int8         // Asm: ANDV
func (x Int8s) Or() int8          // Asm: ORV
func (x Int8s) Xor() int8         // Asm: EORV
// ... analogous for all other vector types (And/Or/Xor are integer-only)
```

#### 9. Broadcast, Index, and Lane Access

```go
func BroadcastInt8s(v int8) Int8s             // Asm: DUP
// ... analogous for all other element types

func IndexInt8s(start, step int8) Int8s       // 0..n-1 sequence scaled to lanes. Asm: INDEX
// ... analogous for Int16s, Int32s, Int64s, Uint*s
```

Conditional extract (last active lane):

```go
func (x Int8s) LastActive(m Mask8s) int8      // Asm: LASTB
func (x Int8s) FirstActive(m Mask8s) int8     // Asm: LASTA (the lane *before* the first active is returned by LASTA on SVE)
```

#### 10. Permutations and Interleaves

```go
func (x Int8s) Reverse() Int8s                       // reverse all lanes. Asm: REV
func (lo Int8s) InterleaveLo(hi Int8s) Int8s         // Asm: ZIP1
func (lo Int8s) InterleaveHi(hi Int8s) Int8s         // Asm: ZIP2
func (lo Int8s) PackEven(hi Int8s) Int8s             // even-indexed lanes. Asm: UZP1
func (lo Int8s) PackOdd(hi Int8s) Int8s              // odd-indexed lanes. Asm: UZP2
func (lo Int8s) TransposeEven(hi Int8s) Int8s        // Asm: TRN1
func (lo Int8s) TransposeOdd(hi Int8s) Int8s         // Asm: TRN2
func (x Int8s) Permute(idx Uint8s) Int8s             // table lookup. Asm: TBL
func (x Int8s) Compress(m Mask8s) Int8s              // pack active lanes into low end. Asm: COMPACT (32/64-bit only on base SVE)
func (x Int8s) Splice(y Int8s, m Mask8s) Int8s       // splice active suffix of x with prefix of y. Asm: SPLICE
// ... analogous for other element widths (Permute index lane width matches the data lane width)
```

#### 11. Fluent Predication and Peephole Lowering
Nearly all SVE arithmetic, logic, and memory instructions can be governed by a predicate. To avoid doubling the method surface (`AddMasked`, `SubMasked`, ...), masking is applied via a chained call. The names match `simd/archsimd`:

```go
func (x Int8s) Masked(m Mask8s) Int8s              // zero-on-false (Zeroing Predication)
func (x Int8s) Merge(y Int8s, m Mask8s) Int8s      // select(m, x, y) (Merging Predication)
```

`Masked` returns a vector whose lanes are `x[i]` where `m[i]` is active and `0` otherwise. `Merge` returns `x[i]` where `m[i]` is active and `y[i]` otherwise.

**Peephole Compiler Lowering.** When the Go compiler sees an unmasked operation followed immediately by `.Masked(m)` or `.Merge(y, m)`, it lowers the pair into a single predicated SVE instruction.

* `xv.Add(yv).Masked(p)` lowers to `ADD Z0.B, P0/Z, Z0.B, Z1.B` (zeroing).
* `xv.Add(yv).Merge(zv, p)` lowers to `ADD Z0.B, P0/M, Z0.B, Z1.B` (merging into `zv`).

## Example
Below is an example test demonstrating a Vector Length Agnostic (VLA) loop that adds two slices of `int8`s together. SVE allows the loop stride to safely scale to the hardware's vector length while automatically masking the tail end of the slice.

```go
func AddSlice(x, y []int8) []int8 {
	// Any stride <= the hardware VL works. We use 5 here as an example.
	stride := min(5, len(x), len(y))
	commonLen := min(len(x), len(y))
	res := make([]int8, commonLen)
	for i := 0; i < commonLen; i += stride {
		xv := archsimd.LoadInt8sSlicePart(x[i:])
		yv := archsimd.LoadInt8sSlicePart(y[i:])
		zv := xv.Add(yv)
		zv.StoreSlicePart(res[i:])
	}
	return res
}
```
