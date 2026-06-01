## Proposal Details
This is a proposal to introduce intrinsic support for ARM64 SVE (Scalable Vector Extension) instructions. It is a child proposal of #73787.

SVE is a recent architecture extension introduced to the ARM64 architecture. Its defining feature is a Vector Length Agnostic (VLA) programming model, which allows developers to write SIMD code once and have it scale automatically to the hardware's available vector length, much like standard scalar code. This proposal aims to provide a clean, accessible API that feels idiomatic to Go, mirrors the existing AMD64 `archsimd` API where semantics overlap, and integrates smoothly with [Midway](https://github.com/golang/go/issues/78902).

This proposal only covers SVE and some SVE2. Specifically, loads from and stores to register lists are not supported. Each type supported will map to one `Z` or `P` register and we assume their length to be at most 256 bits and 32 bits. As a result, `PN` registers are also not supported. SVE2.1 and SME are not within the scope of this proposal.

*Question: how many machines support SVE2? Is it worth supporting full-fledged SVE2?*

### Naming alignment

Where an operation has a direct semantic counterpart in `simd/archsimd` (the AMD64 API in #73787), this proposal uses the same method name (`Add`, `Sub`, `Mul`, `Min`, `Max`, `Sqrt`, `Abs`, `Neg`, `And`, `Or`, `Xor`, `AndNot`, `Not`, `ShiftLeft`/`ShiftRight`, `RotateLeft`/`RotateRight`, `AddSaturated`/`SubSaturated`, `MulAdd`, `OnesCount`, `LeadingZeros`, `Equal`/`NotEqual`/`Less`/`LessEqual`/`Greater`/`GreaterEqual`, `Masked`, `Merge`, etc.). The element/vector type names follow the Midway-style plural convention (`Int8s`, `Float32s`, ...) because that matches SVE's length-agnostic nature. Signatures use Midway types throughout so that user code can switch between architectures with minimal surface change.

### Predication

The SVE API all comes without predication, the user can use `.Masked(m)` and `.IfElse(m)` to ask for zero predication and merging predication.

Many instructions take a predicate `P`, while a lot of them also come with an unpredicated form, some does not. For instructions that comes with an unpredicated form, the intrinsic will map to that one; and when combined with `Masked` and `IfElse`, the compiler will try to peephole it to a predicated form if it exists. For instructions that does not come with an unpredicated form, an all-active predicate will be constructed in place by the compiler and provided to the predicated instruction, and the intrisic maps to this two-instruction sequence. The compiler peepholes can strip away this all-true predicate when the user call `Masked` and `IfElse` right after.

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

ARM provides the `RDVL` instruction to read the hardware's actual vector length at runtime. When the `archsimd` package is imported, `RDVL` will be checked during initialization; if the hardware vector length exceeds 32 bytes, the package will panic. We believe 32 bytes (256 bits) covers the vast majority of SVE chips currently on the market (e.g., Neoverse V1). Please let us know if this constraint needs to be expanded. The APIs supported are based on `ISA_A64_xml_A_profile-2025-12`.

### Memory Operations (Loads and Stores)

For names that are already specified in Midway, unless documented in comment, they have the same semantic as [Midway](https://github.com/golang/go/issues/78902). `// Asm` documents their Arm64 instruction title in the spec table.

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
// Asm: Emulated (predicate construction + "LD1B (scalar plus vector)")
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
// Asm: Emulated (predicate construction + "ST1B (scalar plus vector)")
func (x Int8s) ScatterInt8sPart(idx Int8s, base []int8)
// ... analogous for all other vector types
```

*Note: with the proper predicate constructed by the compiler, these gather/scatter loads/stores can be bound-safe for Go.*

#### Mask Loads and Stores
A predicate is loaded from / stored to a `*uint32` bitmask, where bit `i` corresponds to lane `i`'s active state:

```go
// LoadMask8s loads a predicate from a bitmask. The bits are concatenated
// together in little-endian order.
// If the bits slice doesn't have enough elements to fill the full mask, it will panic.
//
// Asm: LDR (predicate)
func LoadMask8s(bits []uint16) Mask8s
// StoreMask8s stores the predicate to a slice of bits. The bits are concatenated
// together in little-endian order.
// If the slice doesn't have enough elements to store the full mask, it will panic.
//
// Asm: STR (predicate)
func (m Mask8s) Store(bits []uint16)
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

// First returns a mask that activates only the first active element of m.
//
// Asm: PFIRST
func (m Mask8s) First() Mask8s
// Next returns a mask that activates the next element after the last active element of m.
// All other elements will be set to inactive.
// If m is all inactive, the returned mask will have its first element activated.
//
// Asm: PNEXT
func (m Mask8s) Next() Mask8s
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
*Note: PTEST could be peepholed with PFIRST and PNEXT.*

*Note: FIRSTP looks like a useful instruction, however it's SVE2, should we support it?*

#### Conversions
SVE predicates are layout-identical bitmasks (1 bit per byte), but are typed by lane width in Go for type safety. Conversions can change a mask type to another.

##### Widen Elements
Unpack and widen (by padding 0 bits) a predicate of narrower lanes into wider lanes.

```go
// UnpackWidenLo unpacks the lanes from the lower half of the source mask m
// and widens them by padding 0s to twice their lane width.
//
// Asm: PUNPKLO
func (m Mask8s) UnpackWidenLo() Mask16s

// UnpackWidenHi unpacks the lanes from the upper half of the source mask m
// and widens them by padding 0s to twice their lane width.
//
// Asm: PUNPKHI
func (m Mask8s) UnpackWidenHi() Mask16s
// ... analogous for wider masks
```

##### Narrow Elements
Pack two wider-lane masks (low/high halves) into a single narrower-lane mask.

```go
// PackNarrow packs two wider-lane masks into a single narrower-lane mask.
// lo will be placed in the lower half of the result, and hi will be placed in the upper half.
//
// Asm: UZP1 (predicates)
func (lo Mask16s) PackNarrow(hi Mask16s) Mask8s
// ... analogous for wider masks
```

*Note:* `UZP1` on predicates can also do interleaved deinterleaving when operand and result types are the same; that is exposed below under `PackEven`/`PackOdd`.

##### Same-width Packing
Pack the even- or odd-indexed lanes of two masks into a single mask.

```go
// PackEven extracts the even-indexed elements from lo and hi and concatenates them.
// lo's even elements are placed in the lower half of the result, and hi's even
// elements are placed in the upper half.
//
// Asm: UZP1 (predicates)
func (lo Mask8s) PackEven(hi Mask8s) Mask8s

// PackOdd extracts the odd-indexed elements from lo and hi and concatenates them.
// lo's odd elements are placed in the lower half of the result, and hi's odd
// elements are placed in the upper half.
//
// Asm: UZP2 (predicates)
func (lo Mask8s) PackOdd(hi Mask8s) Mask8s
// ... analogous for wider masks
```

##### No-op conversions
These conversions reinterpret the bits of a mask. They are no-op operations.
```go
func (m Mask8s) AsMask16s() Mask16s
func (m Mask8s) AsMask32s() Mask32s
func (m Mask8s) AsMask64s() Mask64s
// ... analogous for wider masks
```
These conversions do not exist on amd64 or arm64. However this proposal still includes them because SVE predicates are universally applied on byte lane basis regardless of arrangement. Larger mask types use only the bits whose indices are a multiple of the byte size of the mask lane. So it's possible that the user may want to reinterpret the bits in a predicate to apply on different vector arrangements. Discussions are welcomed!

#### Permutations

```go
// Reverse reverse the order of elements in x.
//	x[i] = x[x.Len() - 1 - i]
// Asm: REV (predicate)
func (m Mask8s) Reverse() Mask8s
// InterleaveLo interleaves the lower half of x with the lower half of y.
//	x = [x0, x1], y = [y0, y1]
//	result = [x0, y0, x1, y1]
//
// Asm: ZIP1 (predicate)
func (x Mask8s) InterleaveLo(y Mask8s) Mask8s
// InterleaveHi interleaves the upper half of x with the upper half of y.
//	
//
// Asm: ZIP2 (predicate)
func (x Mask8s) InterleaveHi(y Mask8s) Mask8s
// InterleaveEven interleaves the even-indexed lanes of x and y,
// x taking the even-indexed lanes and y taking the odd-indexed lanes in the result.
//
// Asm: TRN1 (predicate)
func (x Mask8s) InterleaveEven(y Mask8s) Mask8s
// InterleaveOdd interleaves the odd-indexed lanes of x and y,
// x taking the even-indexed lanes and y taking the odd-indexed lanes in the result.
//
// Asm: TRN2 (predicate)
func (x Mask8s) InterleaveOdd(y Mask8s) Mask8s
// ... analogous for wider masks
```

#### Comparisons Producing Masks
All scalable vector types support element-wise comparisons that yield the corresponding mask type:

```go
func (x Int8s) Equal(y Int8s) Mask8s            // Asm: CMPEQ (vectors)
func (x Int8s) NotEqual(y Int8s) Mask8s         // Asm: CMPNE (vectors)
func (x Int8s) Greater(y Int8s) Mask8s          // Asm: CMPGT (vectors)
func (x Int8s) GreaterEqual(y Int8s) Mask8s     // Asm: CMPGE (vectors)
// ... analogous for all other integer / float vector types
```

Floating-point types additionally provide:

```go
func (x Float32s) IsNaN() Mask32s // Asm: FCMUO (vectors)
func (x Float64s) IsNaN() Mask64s // Asm: FCMUO (vectors)
```

### Vector Operations
All SVE vector operations are presented in their unconditional form. Predication is applied through the fluent `Masked` / `Merge` chaining described at the end of this section, and the compiler folds the chain into a single predicated instruction.

#### Element-wise Arithmetic

```go
func (x Int8s) Add(y Int8s) Int8s        // Asm: ADD (vectors, unpredicated)
func (x Int8s) Sub(y Int8s) Int8s        // Asm: SUB (vectors, unpredicated)
func (x Int8s) Mul(y Int8s) Int8s        // Asm: MUL (vectors, unpredicated)
func (x Int8s) Abs() Int8s               // Asm: ABS (predicated)
func (x Int8s) Min(y Int8s) Int8s        // Asm: SMIN (vectors)
func (x Int8s) Max(y Int8s) Int8s        // Asm: SMAX (vectors)
// ... analogous for all other vector types
```

Division and square root are float-only (integer SDIV/UDIV exist only for 32-/64-bit lanes in SVE and are listed separately):

```go
func (x Float32s) Div(y Float32s) Float32s            // Asm: FDIV
func (x Float32s) Sqrt() Float32s                     // Asm: FSQRT
func (x Float32s) Reciprocal() Float32s               // Asm: FRECPE
func (x Float32s) ReciprocalSqrt() Float32s           // Asm: FRSQRTE
// TODO: should we also support FRECPS, FRECPX, FRSQRTS?
// ... analogous for Float64s
```

Integer division (32- and 64-bit only):

```go
func (x Int32s) Div(y Int32s) Int32s        // Asm: SDIV
func (x Uint32s) Div(y Uint32s) Uint32s     // Asm: UDIV
// ... analogous for 64-bit
```

Multiply-high (upper half of a widening multiply, without widening the result type):

```go
func (x Int8s) MulHigh(y Int8s) Int8s       // Asm: SMULH (unpredicated)
func (x Uint8s) MulHigh(y Uint8s) Uint8s    // Asm: UMULH (unpredicated)
// ... analogous for 16/32/64-bit integer lanes
```

Fused multiply-add (single-rounding):

```go
func (x Float32s) MulAdd(y, z Float32s) Float32s    // Asm: FMLA (vectors)
// MulSub performs a fused (x * y) - z.
//
// Asm: FMLS (vectors)
func (x Float32s) MulSub(y, z Float32s) Float32s
// ... analogous for Float64s
```

#### Saturating Arithmetic (integer types)

```go
func (x Int8s) AddSaturated(y Int8s) Int8s    // Asm: SQADD (vectors, unpredicated)
func (x Uint8s) AddSaturated(y Uint8s) Uint8s // Asm: UQADD (vectors, unpredicated)
func (x Int8s) SubSaturated(y Int8s) Int8s    // Asm: SQSUB (vectors, unpredicated)
func (x Uint8s) SubSaturated(y Uint8s) Uint8s // Asm: UQSUB (vectors, unpredicated)
// ... analogous for 16/32/64-bit integer lanes
```

#### Bitwise Logic (integer types)

```go
func (x Int8s) And(y Int8s) Int8s        // Asm: AND (vectors, unpredicated)
func (x Int8s) Or(y Int8s) Int8s         // Asm: ORR (vectors, unpredicated)
func (x Int8s) Xor(y Int8s) Int8s        // Asm: EOR (vectors, unpredicated)
func (x Int8s) AndNot(y Int8s) Int8s     // Asm: BIC (vectors, unpredicated)
func (x Int8s) Not() Int8s               // Asm: NOT (vectors)
// ... analogous for all other integer vector types
```

#### Shifts and Rotations (integer types)
Vector-by-scalar (single shift amount applied to every lane), and vector-by-vector (per-lane shift amount) forms. Right shift is logical for unsigned lanes and arithmetic for signed lanes.

```go
func (x Int8s) ShiftAllLeft(shift uint64) Int8s     // Asm: LSL (immediate, unpredicated)
func (x Int8s) ShiftLeft(y Uint8s) Int8s            // Asm: LSL (vectors)
func (x Int8s) ShiftAllRight(shift uint64) Int8s    // Asm: ASR (immediate, unpredicated)
func (x Int8s) ShiftRight(y Uint8s) Int8s           // Asm: ASR (vectors)

func (x Uint8s) ShiftAllRight(shift uint64) Uint8s  // Asm: LSR (immediate, unpredicated)
func (x Uint8s) ShiftRight(y Uint8s) Uint8s         // Asm: LSR (vectors)
// ... analogous for 16/32/64-bit integer lanes
```

Rotations are supported in RAX1 and XAR instructions as only **part** of their semantics, how should we support them?

#### Bit Manipulation (integer types)

```go
func (x Uint8s) OnesCount() Uint8s           // Asm: CNT
func (x Uint8s) LeadingZeros() Uint8s        // Asm: CLZ
// ... analogous for signed lanes
```

#### Floating-Point Rounding

```go
func (x Float32s) Round() Float32s        // Asm: FRINTN
func (x Float32s) Ceil() Float32s         // Asm: FRINTP
func (x Float32s) Floor() Float32s        // Asm: FRINTM
func (x Float32s) Trunc() Float32s        // Asm: FRINTZ
// ... analogous for Float64s
```

#### Type Conversions and Lane-Width Changes

##### Reinterpret (no instruction; bit-cast)

```go
func (x Int8s) ToBits() Uint8s
func (x Uint8s) ReshapeToUint16s() Uint16s
func (x Uint8s) ReshapeToUint32s() Uint32s
func (x Uint8s) ReshapeToUint64s() Uint64s
// ... analogous for all same-width integer vectors
```

##### Widen (sign- or zero-extend lanes)

```go
// UnpackWidenLo takes the lower half of the vector and sign-extends it to 16-bit lanes.
//
// Asm: SUNPKLO
func (x Int8s) UnpackWidenLo() Int16s
// UnpackWidenHi takes the higher half of the vector and sign-extends it to 16-bit lanes.
//
// Asm: SUNPKHI
func (x Int8s) UnpackWidenHi() Int16s
// UnpackWidenLo takes the lower half of the vector and zero-extends it to 16-bit lanes.
//
// Asm: UUNPKLO
func (x Uint8s) UnpackWidenLo() Uint16s
// UnpackWidenHi takes the higher half of the vector and zero-extends it to 16-bit lanes.
//
// Asm: UUNPKHI
func (x Uint8s) UnpackWidenHi() Uint16s
// ... analogous for Int16s↔Int32s, Int32s↔Int64s, Uint16s↔Uint32s, Uint32s↔Uint64s
```

##### Narrow (truncate to half-width lanes)

```go
// PackTrunc truncates lo and hi and pack them to the lower and higher halves of
// the result vector.
//
// Asm: UZP1
func (lo Int16s) PackTrunc(hi Int16s) Int8s
// ... analogous for Uint16s, Int32s, Uint32s, Int64s, Uint64s
```

##### Same-width Packing
Pack the even- or odd-indexed lanes of two masks into a single mask.

```go
// PackEven extracts the even-indexed elements from lo and hi and concatenates them.
// lo's even elements are placed in the lower half of the result, and hi's even
// elements are placed in the upper half.
//
// Asm: UZP1 (vectors)
func (lo Int8s) PackEven(hi Int8s) Int8s

// PackOdd extracts the odd-indexed elements from lo and hi and concatenates them.
// lo's odd elements are placed in the lower half of the result, and hi's odd
// elements are placed in the upper half.
//
// Asm: UZP2 (vectors)
func (lo Int8s) PackOdd(hi Int8s) Int8s
// ... analogous for Int16s, Int32s, Int64s, Uint16s, Uint32s, Uint64s
```

##### Integer ↔ Float

```go
func (x Int32s) ConvertToFloat32s() Float32s         // Asm: SCVTF (predicated)
func (x Uint32s) ConvertToFloat32s() Float32s        // Asm: UCVTF (predicated)
func (x Float32s) ConvertToInt32s() Int32s           // Asm: FCVTZS
func (x Float32s) ConvertToUint32s() Uint32s         // Asm: FCVTZU
func (x Int64s) ConvertToFloat64s() Float64s         // Asm: SCVTF (predicated)
func (x Uint64s) ConvertToFloat64s() Float64s        // Asm: UCVTF (predicated)
func (x Float64s) ConvertToInt64s() Int64s           // Asm: FCVTZS
func (x Float64s) ConvertToUint64s() Uint64s         // Asm: FCVTZU
```

Cross-width float conversions:

```go
// UnpackWidenEvenToFloat64s performs the following operation:
//	result[i] = Float32(float64(x[2i]))
//
// Asm: FCVT
func (x Float32s) UnpackWidenEvenToFloat64s() Float64s
// EvenNarrowToFloat32s performs the following operation:
//	result[2i] = Float32((y[i]))
//	result[2i+1] = 0
//
// Asm: FCVT
func (y Float64s) EvenNarrowToFloat32s() Float32s
```

#### Horizontal Reductions
Compute a scalar value across all active lanes of a scalable vector:

```go
// SumReduce reduces x to the sum of all elements in x.
//
// Asm: SADDV
func (x Int8s) SumReduce() int8
// MinReduce reduces x to the minimum value among all active elements.
//
// Asm: SMINV
func (x Int8s) MinReduce() int8
// MaxReduce reduces x to the maximum value among all active elements.
//
// Asm: SMAXV
func (x Int8s) MaxReduce() int8
// AndReduce reduces x to the logical AND of all active elements.
//
// Asm: ANDV
func (x Int8s) AndReduce() int8
// OrReduce reduces x to the logical OR of all active elements.
//
// Asm: ORV
func (x Int8s) OrReduce() int8
// XorReduce reduces x to the logical XOR of all active elements.
//
// Asm: EORV
func (x Int8s) XorReduce() int8
// ... analogous for all other vector types (And/Or/Xor are integer-only)
```

#### Broadcast, Index, and Lane Access

```go
func BroadcastInt8s(v int8) Int8s             // Asm: DUP (scalar)
// .. analogous for all other vector types.

// ArithSeqInt8s creates an arithmetic sequence with the given start and step:
//	result[i] = start + i * step
//
// A non-constant value of step may result in significantly worse performance for this operation.
//
// Asm: INDEX (scalar, immediate)
func IndexInt8s(start, step int8) Int8s
// ... analogous for Int16s, Int32s, Int64s, Uint*s
```

Element Getters:

```go
// GetElemLastActive extract the last active element in x governed by m
// If m is all false, the highest-numbered element is extracted.
//
// Asm: LASTB
func (x Int8s) GetElemLastActive(m Mask8s) int8
// GetElemAfterLastActive extract the element right after the last active element in x governed by m.
// If m is all false, the result is 0. If the last active element in x is the last element in x,
// the result will be x[0].
//
// Asm: LASTA
func (x Int8s) GetElemAfterLastActive(m Mask8s) int8
// ... analogous for all other vector types
```

*Note: These are the intrinsic SVE GetElem operations and they look strange, should we support a clean version of `func (x Int8s) GetElem(i index) int8` that mimics the amd64 API? They will be emulations based on these intrinsics.*

Element Setters:
```go
// SetElem sets the element at index to v.
//
// Asm: Emulated (Predicate construction + "CPY (scalar)")
func (x Int8s) SetElem(index uint8, v uint8) Int8s
// ... analogous for all other vector types
```

#### Permutations

```go
// Reverse reverse the order of elements in x.
//	x[i] = x[x.Len() - 1 - i]
// Asm: REV (vectors)
func (x Int8s) Reverse() Int8s
func (x Int8s) InterleaveLo(y Int8s) Int8s // Asm: ZIP1 (vectors)
func (x Int8s) InterleaveHi(y Int8s) Int8s // Asm: ZIP2 (vectors)
// InterleaveEven interleaves the even-indexed lanes of x and y,
// x taking the even-indexed lanes and y taking the odd-indexed lanes in the result.
//
// Asm: TRN1 (vectors)
func (x Int8s) InterleaveEven(y Int8s) Int8s
// InterleaveOdd interleaves the odd-indexed lanes of x and y,
// x taking the even-indexed lanes and y taking the odd-indexed lanes in the result.
//
// Asm: TRN2 (vectors)
func (x Int8s) InterleaveOdd(y Int8s) Int8s
func (x Int8s) PermuteOrZero(idx Uint8s) Int8s // Asm: TBL
func (x Int8s) Compress(m Mask8s) Int8s        // Asm: COMPACT (32/64-bit only on base SVE)
// Splice splices x and y with m: the a range in x governed by m's first and last active element will
// be copied to the result's lower part, and the remaining high part will be copied from y's low part.
// For example:
// x = [1, 2, 3, 4], m = [T, F, T, F], y = [5, 6, 7, 8]
// result = [1, 2, 3, 5]
//
// Asm: SPLICE
func (x Int8s) Splice(y Int8s, m Mask8s) Int8s
// ... analogous for other element widths
```

#### Fluent Predication and Peephole Lowering
Nearly all SVE arithmetic, logic, and memory instructions can be governed by a predicate. To avoid doubling the method surface (`AddMasked`, `SubMasked`, ...), masking is applied via a chained call. The names match `simd/archsimd`:

```go
func (x Int8s) Masked(m Mask8s) Int8s              // (Zeroing Predication)
func (x Int8s) IfElse(y Int8s, m Mask8s) Int8s     // (Merging Predication)
```

`Masked` returns a vector whose lanes are `x[i]` where `m[i]` is active and `0` otherwise. `IfElse` returns `x[i]` where `m[i]` is active and `y[i]` otherwise.

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
