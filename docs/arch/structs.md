# Struct Representation and ABI

Structures can be declared directly in oz-lang or imported as named module
types, such as `компл` from the complex-number module. For construction, field
access, nesting, and reference parameters, start with the
[language guide](core-lang.md#named-types-and-structures). At AST level a struct is
`TStructType`, optionally wrapped in `TNamedType` so user-facing diagnostics and
printers can keep the domain name:

```text
<named компл <struct (re f64) (im f64)>>
```

## IR Type

`FromAstType(TStructType)` maps to an IR `Struct` type whose fields are the
lowered field types in declaration order. `TTypeTable` interns struct layouts
by the vector of field type ids:

```text
struct { f64; f64 }          ; complex number payload
```

Struct layout includes alignment padding. `FieldOffset` aligns each field's
start using `AlignInBytes`; `SizeInBytes` follows the same placement rules and
rounds the final size to the structure's alignment. For example, an `i8`
followed by an `i64` has field offsets 0 and 8 and occupies 16 bytes, rather
than 9. A 128-bit integer field requires 16-byte alignment. Pointer size and
alignment follow the module's target width, so a pointer-containing structure
can differ between native 64-bit code and wasm32. There is no separate runtime
header or field-name table in each value.

## In-Memory Layout

When lowering needs an addressable struct value, it emits `salloc(size)` and
writes fields into that temporary buffer. Field access computes:

```text
fieldPtr = objectAddress + fieldByteOffset
```

For scalar fields, reads use `lde` and writes use `ste`. For nested struct
fields, field access returns the field pointer and assignments use `copy`
because the value may be address-backed rather than an SSA scalar.

Struct values inside arrays are stored inline in the flat array buffer. An
array access to a struct element returns a pointer to the element so callers can
copy from or write into that storage.

## LLVM ABI

The LLVM backend models struct arguments and return values as ordinary LLVM
struct values, not hidden out-parameters:

```llvm
define { double, double } @make_complex(...)
define void @use_complex({ double, double } %z)
```

This is the external ABI seen by LLVM functions. The IR lowering may still
represent an expression as a pointer to a temporary struct buffer. Codegen
bridges this mismatch at call and return boundaries:

| Boundary | Codegen behavior |
|----------|------------------|
| struct argument | if the IR operand is a pointer, load the LLVM struct value before `call` |
| struct return | if `ret` receives a pointer but LLVM expects a struct, load before `ret` |
| store into struct local | if destination is a struct alloca and source is a pointer, emit memcpy |
| `copy` into pointer | if source is already a struct LLVM value, store it; otherwise memcpy |

So the high-level convention is value ABI for LLVM, with address-backed
temporaries as an implementation detail inside lowering.

## Function Parameters and Reference Parameters

Plain struct parameters are passed by value in LLVM. Mutating such a parameter
inside the callee mutates the callee's local copy, not the caller's object.

Reference parameters (`рез` / `арг рез`) are lowered as pointer/reference
types. For a struct reference assignment, lowering copies the full struct into
the referenced destination:

```text
addr = load reference parameter
copy addr, rhsStruct, sizeof(struct)
```

This is the mechanism used for `рез компл`-style APIs: the ABI carries an
address, and assignment writes the whole struct payload into the caller-owned
storage.

## VM Notes

The VM has extra support for materializing struct temporaries and copying
struct payloads. `StructStore` copies a struct value into a local frame slot,
and call/return handling materializes struct temporaries when a struct value is
represented by an address in IR. This keeps VM behavior aligned with the LLVM
value ABI at the language level.
