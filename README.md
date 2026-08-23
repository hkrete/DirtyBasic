# DirtyBasic Compiler and Language Reference

DirtyBasic is a self-hosted native compiler and programming language for Windows x64 and Linux x64. Source files use the `.dbas` extension, and the command-line compiler is `DirtyLPL.exe`.

This document describes the syntax accepted by the current compiler. The executable regression suite in `Samples/syntax_test.dbas` remains the most exact source of truth when the language changes.

## 1. What the compiler produces

DirtyBasic compiles `.dbas` programs directly to native x64 machine code and writes Windows PE executables or DLLs and Linux ELF executables or shared objects.

The compiler pipeline performs these broad phases:

1. Read the `.dbas` source.
2. Resolve and expand imported `.dbas` files.
3. Lower Form and DirtyUI syntax.
4. Normalize the single DirtyBasic block syntax.
5. Lower high-level expressions and derived metadata.
6. Lower classes, methods, properties, and other type features.
7. Emit x64 code and static data.
8. Build native imports, patch references, and optimize.
9. Write a Windows PE executable/DLL or Linux ELF executable/shared object, plus a generated `.dbi` interface for shared libraries.

Supported output targets:

| Target | Status | Output |
| --- | --- | --- |
| `win64-x64` | Supported; default | Windows x64 PE executable or DLL |
| `linux-x64` | Supported | Linux x64 ELF executable or `.so` |
| `macos-arm64` | Planned | Not currently emitted |

## 2. Command-line usage

Compile a Windows console program:

```powershell
.\DirtyLPL.exe Samples\hello.dbas hello.exe
.\hello.exe
```

The explicit form is equivalent:

```powershell
.\DirtyLPL.exe --target win64-x64 --source Samples\hello.dbas --out hello.exe
```

Compile a Windows GUI-subsystem program:

```powershell
.\DirtyLPL.exe --target win64-x64 --gui Samples\app.dbas app.exe
```

Cross-compile a Linux x64 executable:

```powershell
.\DirtyLPL.exe --target linux-x64 Samples\hello.dbas hello
```

Useful diagnostic modes:

```powershell
.\DirtyLPL.exe --check Samples\hello.dbas
.\DirtyLPL.exe --check-include include\string.dbas
.\DirtyLPL.exe --stats Samples\hello.dbas hello.exe
.\DirtyLPL.exe --stats-functions Samples\hello.dbas hello.exe
.\DirtyLPL.exe --emit-asm Samples\hello.dbas hello.asm
.\DirtyLPL.exe --dirtybind native.h --out native.dbi --dll native.dll --so libnative.so
```

### Source-aware diagnostics

Compiler errors retain their original `.dbas` filename, line, and column even when the source came from an imported include or passed through syntax lowering. Diagnostics show the original source line and underline the complete offending token: `end` receives `^^^`, while a typo such as `selact` receives `^^^^^^`. Messages may also include a targeted help suggestion or a secondary source location.

Unknown statement names are reported at the name itself instead of being misreported as an assignment failure on the following line. For unbalanced blocks, DirtyBasic uses indentation only as a diagnostic clue. Indentation still has no effect on parsing. This lets the compiler identify an earlier `end` that probably closed a block prematurely and also show the later unmatched `end` where the structure becomes undeniably invalid.

The compiler defines target symbols automatically:

- `SH_TARGET_WIN64_X64`
- `SH_TARGET_LINUX_X64`
- `SH_TARGET_X64`
- `SH_TARGET_WINDOWS`
- `SH_TARGET_LINUX`

Only the symbols appropriate to the selected target are active.

## 3. Shared libraries and `export`

DirtyBasic can build real native shared libraries. `--shared` selects library output; Windows requires a `.dll` output name and Linux requires `.so`:

```powershell
.\DirtyLPL.exe --shared Samples\shared_library_smoke.dbas Samples\shared_library_smoke.dll
.\DirtyLPL.exe --target linux-x64 --shared Samples\shared_library_smoke.dbas Samples\libshared_library_smoke.so
```

A library must contain at least one exported routine. The compiler emits:

- a PE32+ DLL with a native export table on Windows x64;
- an ELF64 `ET_DYN` shared object with dynamic symbols, relocations, dependencies, `DT_INIT`, W^X load segments, and a non-executable stack on Linux x64;
- a generated DirtyBasic interface file (`.dbi`) beside the library.

Windows DLLs use high-entropy ASLR, `DYNAMIC_BASE`, and NX compatibility. The generated machine code is position-independent and uses RIP-relative code/data/import references.

### Exported declarations

`export` is an explicit library boundary. It is accepted on:

- top-level `function`, `sub`, and `inline` routines, which become native exported symbols;
- `class`, which exports its public methods and properties plus ownership-safe factory/destructor functions;
- `struct`, `union`, `tagged union`, `enum`, `type`, and `const`, which are published in the generated `.dbi` interface for callers.

Example library:

```dbas
export const WIDGET_ABI_VERSION = 1
export type WidgetId = distinct int

export struct Point
    x: float
    y: float
end

private createdWidgets: int = 0

export function Add(a: int, b: int) -> int
    return a + b
end

export class Widget
    location: Point

    sub __init__(initial: Point)
        self.location = initial
        createdWidgets += 1
    end

    property Point Location
        get:
            return self.location
        set:
            self.location = value
    end

    private function InternalId() -> int
        return createdWidgets
    end
end
```

Runtime globals are deliberately not exportable. Keep library state private and expose it through routines or class properties. Native `extern` declarations are imports, not re-exports. The compiler rejects unsupported export targets with `E0503` instead of silently omitting them.

An exported class is opaque to consumers. Its fields are not copied into the `.dbi`; callers use an owned pointer, public methods, and public properties:

```dbas
from "./widget.dbi" import *

function main() -> int
    start: Point = {10.0f, 20.0f}
    widget: Widget* = New Widget(start)
    current: Point = widget.Location
    delete widget
    return 0
end
```

`New Widget(...)` in a consumer calls the library-generated `Widget___new__` factory. `delete widget` calls `Widget___delete__` inside the same library. A private constructor suppresses the public factory; a default public constructor receives a factory automatically. Never use a foreign allocator to free a library-owned object. For returned buffers or other heap memory, export an explicit matching release routine unless the API documents the result as borrowed/static.

### Generated `.dbi` interfaces

For `example.dll` or `libexample.so`, the compiler writes `example.dbi`. It contains:

- shared interface ABI marker `1`;
- all exported/transitively required value types;
- exported constants and type aliases;
- opaque imported class declarations;
- target-selected `unsafe dll` declarations for `example.dll` and `libexample.so`;
- extern signatures for exported functions, subs, public methods, properties, factories, and destructors.

Private routines, private class members, and private constructors are not published. Import the `.dbi` like an ordinary DirtyBasic source file:

```dbas
from "./example.dbi" import *
```

The `.dbi`, DLL, and `.so` are one versioned API unit. Regenerate the interface whenever the library source changes; do not edit it by hand.

### ABI rules

The public ABI is Microsoft x64 on Windows and System V AMD64 on Linux. The compiler bridges DirtyBasic's internal calling convention to the platform ABI, including:

- scalar integer, pointer, string, float, and double arguments/results;
- small and indirect structure/union arguments and results;
- fixed arrays inside exported value types, plus pointer arguments and pointers to indexed elements;
- packed and explicitly aligned value types with their declared layout preserved in the generated `.dbi`;
- mixed integer/SSE aggregates and arguments that overflow into stack slots;
- default and named arguments reconstructed from imported signature metadata;
- inline exported routines as normal native library symbols;
- homogeneous and heterogeneous multiple results, named results, destructuring/multiple assignment, and `or_return`/`or_else`/`or_continue`/`or_break` propagation across the library boundary;
- opaque class pointers and property accessors.

Exported variadic functions are rejected because they do not yet have a stable cross-language wrapper. Keep exceptions inside the library boundary and return explicit result/status values to foreign callers. When a C/C++ caller consumes a value type, mirror field order, signedness, packing, and alignment exactly and verify it with `sizeof`.

Top-level library initialization runs once when the DLL/SO is loaded. Keep loader-time work deterministic and move fallible or heavyweight initialization into an exported routine.

Linux executables terminate through libc `exit()`, so buffered standard I/O is flushed and registered `atexit` processing runs. The compiler does not use the raw `SYS_exit` shortcut for normal program termination.

### Real DirtyLittleUI GUI library

The full GUI library can be built independently:

```powershell
.\build_dirtylittleui_shared.bat
.\DirtyLPL.exe Samples\dirtylittleui_shared_library_demo.dbas Samples\dirtylittleui_shared_library_demo.exe
```

The build writes `Samples\DirtyLittleUI.dll`, generates `Samples\DirtyLittleUI.dbi`, and refreshes the canonical `include\DirtyLittleUI.dbi`. D2D Form and DirtyUI programs import that interface:

```dbas
from "../include/DirtyLittleUI.dbi" import *
```

`DirtyLittleUI.dll` synchronizes every form's `Width` and `Height` with the active native window before each input/layout/draw pass. During a Windows border drag, `DirtyLittleD2D.dll` invokes the form's redraw callback with that window's exact `D2DForm*` context, so the newly exposed area and anchored/fill controls update before the mouse button is released. `D2DFormSyncWindowSize(form)` is also exported for custom loops.

The exported theme API also records the selected theme on each `D2DForm`. Generated Form and DirtyUI initialization calls `D2DFormSetThemeMode`, so single- and multi-form applications preserve their declared `theme = ...` value across the DLL boundary.

The executable contains only its application code and imports `DirtyLittleUI.dll`; the complete GUI implementation remains in the shared library. D2D_Design-generated `.dbas` files use the same interface automatically, and the designer copies both `DirtyLittleUI.dll` and its `DirtyLittleD2D.dll` dependency beside Windows build and preview outputs.

On Linux, build `Samples\libDirtyLittleUI.so` with `--target linux-x64 --shared`. The same `.dbi` selects `libDirtyLittleUI.so` automatically for Linux targets. Place that shared object on the loader path together with the selected graphics backend dependency.

Opaque GUI class pointers cross the shared boundary through exported methods, properties, and stable facade routines. Form/DirtyUI lowering does not read private GUI fields directly; event dispatch, colors, fonts, paths, rotation, title-bar setup, and multi-form execution are routed through the DLL API.

The permanent regression runner covers Windows, WSL/Linux, native MSVC consumers, generated interface contents, class ownership, struct-valued properties, negative diagnostics, loader hardening, and the real DirtyLittleUI DLL:

```powershell
powershell -ExecutionPolicy Bypass -File tests\shared_library_regression.ps1
```

## 4. Minimal program

```dbas
from "../include/stdio.dbas" import *

function main() -> int
    printf("Hello from DirtyBasic!\n")
    return 0
end
```

`main` is the normal executable entry point. Returning zero reports success to the operating system.

## 5. Core block rules

DirtyBasic uses one `end` keyword to close blocks. Indentation is recommended for readability but has no structural meaning.

Canonical block headers:

| Construct | Header ending | Closing form |
| --- | --- | --- |
| `function`, `sub` | no colon | `end` |
| `if`, `elif`, `elseif` | `then` | multiline/chain: one `end`; standalone single-line `if`: no `end` |
| `else` | no colon | closed by the chain's `end` |
| `while`, `for`, `using`, `try` | no colon | `end` |
| `catch` | colon required | closed by the try block's `end` |
| `select` | no colon | `end` |
| `case`, `default` | colon required | closed by the select's `end` |
| `match` expression | colon required | `end` |
| `struct`, `union`, `tagged union`, `enum` | no colon | `end` |
| `interface`, `class`, `property`, `unsafe dll` | no colon | `end` |
| property `get`, `set` | colon required | no separate accessor `end` |
| preprocessor `!if`, `!elif`, `!elseif`, `!else` | no colon | `!endif` |

The forms in this section define DirtyBasic block syntax.

## 6. Comments, continuation, and literals

`#` begins a line comment:

```dbas
count: int = 10 # comment after code
```

An underscore at the end of a line continues an expression or string construction:

```dbas
message: char* = "Hello," _
                 " from" _
                 " DirtyBasic"
```

Common literal forms:

```dbas
enabled: bool = true
letter: char = 'A'
count: int = 42
mask: uint = 0x10
unsignedMask: uint = 0xFFFFu
wideCount: int64 = 100i64
ratio: float = 1.5f
precise: double = 1.5
text: char* = "hello"
nothing: void* = NULL
```

Both `true`/`false` and the commonly used capitalized forms are accepted. `NULL` represents a null pointer.

## 7. Imports and modules

Import all public declarations from another source file:

```dbas
from "../include/stdio.dbas" import *
from "../include/string.dbas" import *
```

Import selected declarations:

```dbas
from "../include/math.dbas" import sqrtf, cosf, sinf
```

Import into a namespace alias:

```dbas
from "../include/math.dbas" import * as math

root: double = math.sqrt(49.0)
```

Relative import paths are resolved from the importing source file. Imports are expanded by the compiler while source mappings are retained for diagnostics.

Each `.dbas` file is expanded at most once per compilation, so direct, transitive, and diamond imports are safe. Import keys normalize `/` versus `\\`, repeated separators, and `.`/`..` path segments; Windows builds also compare paths case-insensitively. Repeating an import with a namespace alias still registers that alias without expanding the file again. The older `!include-once`/`!include_once` directive remains accepted for compatibility but is no longer required.

### Visibility and access control

Declarations are `public` by default, so existing DirtyBasic programs do not need modifiers. Use `private` only where a module or class needs an implementation detail. `public` is optional but useful when documenting an intended API:

```dbas
private cacheHits: int = 0
public const CACHE_LIMIT = 256

private function ReadCache() -> int
    return cacheHits
end

public function CacheHits() -> int
    return ReadCache()
end
```

A top-level private declaration is accessible anywhere in the same `.dbas` file, including from declarations that appear before or after it. Imported files cannot access it. A private class member is accessible only from routines and property accessors owned by that class:

```dbas
public class Counter
    private value: int

    private sub ResetValue()
        self.value = 0
    end

    public property int Value
        get:
            return self.value
        set:
            self.value = value
    end
end
```

The modifiers are accepted on:

- `function`, `sub`, and `inline` routines, including constructors, destructors, and operator overloads;
- `class`, `struct`, `union`, `tagged union`, `enum`, `type`, and `interface` declarations;
- class fields and class `property` declarations;
- top-level typed variables and `const` declarations;
- `implements` declarations;
- `unsafe dll` blocks and individual `extern function`/`extern sub` declarations.

A private `unsafe dll` block makes its extern declarations private unless an individual declaration explicitly says `public`:

```dbas
private unsafe dll "native_helpers.dll"
    extern function InternalHelper(value: int) -> int
    public extern function PublishedHelper(value: int) -> int
end
```

Interface members describe the public contract and therefore cannot be private. `private implements` is accepted and records a private conformance declaration; the current interface implementation is structural and has no runtime conformance-query operation, so this does not change ordinary method lookup. It does not make the implementing class private.

`let`, `var`, and explicitly typed local variables do not accept visibility modifiers because they are already confined to their lexical routine scope. The compiler reports the original file and line when code crosses a private boundary.

Top-level private declarations have true module-local identities. The compiler mangles their internal names with the declaring module while diagnostics and source lookup continue to use the written name. Two imported modules may therefore reuse the same private function, sub, inline routine, global, constant, alias/type, enum, struct, union, tagged union, class, interface, method, property helper, or native import without colliding. Public declarations keep their written compilation-wide identity.

Private native imports also keep two separate identities: a module-local DirtyBasic name and the unchanged native export name. On Windows, final import patching is qualified by both DLL and export name, so two modules can privately import identical export names from different DLLs and each call the intended library. On ELF, the source identities remain isolated while final symbol binding follows the platform dynamic loader's normal global symbol-resolution rules.

`public`/`private` control source access inside one compilation. `export` is different: it publishes a declaration through a DLL/SO and its generated `.dbi` interface. See the shared-library chapter for supported export targets and ownership rules.

## 8. Preprocessor

### Defines and conditional compilation

```dbas
!define ENABLE_LOGGING 1
!define BUFFER_SIZE 4096
!define DEFAULT_SCALE 1.5f

!if defined(SH_TARGET_WINDOWS)
    !define PLATFORM_NAME "Windows"
!elseif defined(SH_TARGET_LINUX)
    !define PLATFORM_NAME "Linux"
!else
    !define PLATFORM_NAME "Unknown"
!endif
```

`!elif` and `!elseif` are equivalent. None of the preprocessor branch headers uses a colon.

### Link pragmas

`!pragma comment(lib, ...)` requests a native library:

```dbas
!pragma comment(lib, "user32.lib")
!pragma comment(lib, "opengl32.lib")
```

### Automatic null-check pragma

Automatic null-pointer checks are disabled by default to keep compilation and generated programs lean. Enable them explicitly for a file or region with:

```dbas
!pragma null_checks(on)
# checked pointer operations
!pragma null_checks(off)
```

The setting is processed in source order and remains active until changed. Put `!pragma null_checks(on)` near the top of a file to protect all following code. With checks off, a null access may become an operating-system access violation.

### Embedded files

Binary or text assets can be embedded into the executable:

```dbas
!embed "assets/logo.png" as logoData
!embed manifest "../include/app.manifest"
```

The named form exposes embedded data through the compiler's generated resource symbols. See `include/resource.dbas` and the samples using `!embed` for runtime helpers.

## 9. Primitive types

The x64 compiler supports these common primitive families:

| Family | Types and aliases |
| --- | --- |
| Boolean | `bool` |
| 8-bit | `char`, `byte`, `ubyte`, `int8`, `uint8`, `u8` |
| 16-bit | `short`, `ushort`, `int16`, `uint16`, `u16` |
| 32-bit | `int`, `uint`, `int32`, `uint32`, `u32`, `unsigned int` |
| 64-bit | `long`, `ulong`, `int64`, `uint64`, `u64`, `unsigned long` |
| Floating point | `float` (32-bit), `double` (64-bit) |
| Strings | `char*`, `str`, `string` |
| Generic pointer | `void*` |
| No result | `void`, normally expressed by a `sub` |

Pointers are 64-bit on both supported targets. Native API aliases such as `DWORD`, `HANDLE`, and `HRESULT` are supplied by platform include files rather than being special language-only types.

### 64-bit integer literals and expressions

Decimal integer literals may range through `18446744073709551615`; hexadecimal literals may contain up to sixteen digits. Values that fit a signed 32-bit `int` use `int`, larger values through `0x7FFFFFFFFFFFFFFF` use `int64`, and values with bit 63 set use `uint64`.

Integer suffixes can select unsigned or 64-bit behavior explicitly. Suffixes are case-insensitive:

| Suffix | Result type |
| --- | --- |
| `u` | `uint` when the value fits 32 bits, otherwise `uint64` |
| `l`, `i64` | `int64` |
| `ul`, `u64`, `ui64` | `uint64` |

```dbas
zero: uint = 0u
mask32: uint = 0xFFFFFFFFu
signedWide: int64 = 100i64
unsignedWide: uint64 = 42ul
maxWide: uint64 = 0xFFFFFFFFFFFFFFFFui64
promoted: uint64 = 4294967296u
```

An `i64` or `l` literal must not exceed `0x7FFFFFFFFFFFFFFF`. Unsigned 64-bit forms may range through `0xFFFFFFFFFFFFFFFF` or decimal `18446744073709551615`. Suffixes are also accepted in compile-time integer positions such as fixed-array lengths and `select` case values. Malformed suffixes are reported at the literal.

`int64`, `long`, `uint64`, `ulong`, `u64`, and `unsigned long` expressions retain all 64 bits across arithmetic, division and remainder, bitwise operations, shifts, unary `-`/`~`, truth tests, and comparisons:

```dbas
signedValue: int64 = -5000000000
scaled: int64 = (signedValue + -7000000000) * -3
arithmeticShift: int64 = -8589934592 >> 3

bits: uint64 = 0x123456789ABCDEF0
rotatedPart: uint64 = (bits << 7) >> 11
maxUnsigned: uint64 = 18446744073709551615
if maxUnsigned > 0x7FFFFFFFFFFFFFFF then
    printf("unsigned comparison keeps bit 63\n")
end
```

Signed division, remainder, comparisons, and right shifts use signed behavior. If either comparison operand is unsigned, the comparison is unsigned. A right shift of `uint64` is logical; a right shift of `int64` is arithmetic.

### Casts and `sizeof`

DirtyBasic uses C-style casts:

```dbas
i: int = 42
f: float = (float)i
p: void* = malloc(64)
numbers: int* = (int*)p
```

`sizeof` accepts primitive and user-defined types:

```dbas
bytes: int = sizeof(int)
nodeBytes: int = sizeof(Node)
```

## 10. Variables, constants, and assignment

Explicitly typed declarations use `name: type`:

```dbas
count: int = 10
position: Vec2
buffer: char[256]
ptr: Vec2* = &position
```

Inferred local declarations use `let` or `var`. `const` declares a value that should not be reassigned:

```dbas
let title = "DirtyBasic"
var total = 0
const maxItems = 64
const screenWidth: int = 1280
```

Top-level typed declarations are global. They may be marked `private` or explicitly `public`; local declarations may not use either modifier:

```dbas
globalScore: int = 0
globalNames: char*[4]
```

### Multiple declarations

Values can be assigned pairwise:

```dbas
a, b: int = 10, 20
let left, right = 30, 40
var x, y = 1.5f, 2.5f
```

A single initializer is broadcast to all declared names. The explicitly typed form accepts the optional equals spelling:

```dbas
a, b: int 10
c, d: int = 20
let first, second = 30
var low, high = 40
```

This works for primitive values, pointers, strings, structs, and fixed arrays.

### Multiple assignment

Existing variables can be updated simultaneously:

```dbas
a, b = b, a
a, b = 7                 # broadcast
a, b = SwapValues(a, b)  # multiple function result

position.x, position.y = 10, 20
position.x, position.y = position.y, position.x
transform.position.x, transform.position.y = nextX, nextY
values[0], values[2] = values[2], values[0]
```

Fixed arrays can also be whole destinations. Their complete byte contents are staged before any destination is written, so array swaps and broadcasts are safe:

```dbas
left, right = right, left
copyA, copyB = left
```

The right side is evaluated before the destinations are overwritten, so swapping does not need a temporary variable.
The destinations do not need to share one type; every staged value is checked against its corresponding destination.

Multi-assignment destinations may be ordinary variables, struct fields, nested struct fields, or indexed l-values. Destinations can be mixed:

```dbas
player.position.x, camera.target.y = nextPlayerX, nextTargetY
```

Each destination is type-checked using the same rules as an ordinary assignment. A multi-assignment supports up to sixteen destinations, and each target expression must fit on the current source line.

Whole-array assignment requires identical element type, structure identity, rank, and dimensions. Indexed elements remain ordinary scalar/structure l-values and can be mixed with other destinations.

### Compound assignment

```dbas
count += 1
count -= 2
scale *= 1.5f
mask |= 0x10
bits <<= 1
bits >>= 1
```

Arithmetic, bitwise, shift, and their compound-assignment forms are available where the operand types support them.

Compound assignment preserves the declared target type and signedness. It supports signed and unsigned integer widths, `float`, and `double`, including wide `int64` and `uint64` arithmetic. Unsigned division and modulo use unsigned semantics.

The target can be a local or global variable, an indexed array element, a structure/class field, a property, an arrow field, or a directly dereferenced scalar pointer:

```dbas
values[index] *= 3
player.score += bonus
settings.Volume /= 2.0f
nodePtr->flags |= READY_FLAG
*counterPtr += 1
particles[index].position.x += particles[index].velocity.x * deltaTime
```

Fields of structures stored in local or global arrays support the same compound
operators. Mixed `float` and `double` operands are converted to the declared
target type before the operation.

Plain and compound pointer stores can directly follow the address declaration; no separator statement is required:

```dbas
value: int64 = 3
valuePtr: int64* = &value
*valuePtr = 4
*valuePtr *= 5
```

For property targets, the getter supplies the left operand and the setter receives the final value. A compound target and its right side are each evaluated once before the result is stored.

## 11. Operators

Arithmetic operators:

```text
+  -  *  /  %
```

Comparison operators:

```text
==  !=  <>  <  <=  >  >=
```

Logical operators can be written as words or symbols:

```dbas
if ready And count > 0 then
if ready and count > 0 then
if ready && count > 0 then
if Not failed Or retry then
if !failed || retry then
```

Bitwise operators include `&`, `|`, `^`, `~`, `<<`, and `>>`. Word forms such as `and`, `or`, `xor`, and `mod` are accepted where appropriate.

The tenary expression uses `? :`:

```dbas
larger: int = (a > b) ? (a) : (b)
label: char* = active ? "active" : "idle"
```

## 12. Arrays, pointers, and memory

### Fixed arrays and aggregate literals

```dbas
values: int[4] = {10, 20, 30, 40}
inferred: int[] = {-5, 0, 3, 42}
nodes: Node[2] = {{1, 2}, {3, 4}}
```

Array elements and struct fields can be combined naturally:

```dbas
nodes[1].x = 20
```

### Pointers

```dbas
value: int = 42
ptr: int* = &value
*ptr = 43

node: Node
nodePtr: Node* = &node
nodePtr->x = 10
```

Use `.` for values and `->` for pointer receivers. Address-of can be written as `&expression`; `addressof(expression)` is also supported. Multiple pointer levels such as `char**` are valid.

When enabled with `!pragma null_checks(on)`, automatic checks cover explicit dereferences, pointer indexing, `->` field access, pointer-backed method/property/string-method calls, and dynamic arguments passed to FFI parameters declared `nonnull`. For example:

```dbas
!pragma null_checks(on)
nodePtr: Node* = NULL
value: int = nodePtr->x
```

Instead of an unexplained access violation, execution stops with exit code `71` and a source-mapped diagnostic such as:

```text
runtime error: null pointer dereference
  --> app.dbas:2:14
      |
    2 | value: int = nodePtr->x
      |              ^^^^^^^ pointer evaluated to NULL
```

The check establishes only that the address is not zero. It cannot prove that a non-null address is still allocated, correctly aligned, large enough for an index, or valid for the requested type.

### Allocation

Raw memory uses the C-style runtime functions from `stdlib.dbas`:

```dbas
memory: void* = malloc(1024)
if memory == NULL then
    return 1
end
free(memory)
```

There is no garbage collector. Raw allocations and native resources must be released explicitly or through `defer`/`using`.

## 13. Structs, layout, and unions

### Structs

```dbas
struct Vec2
    x: float
    y: float
end

struct Particle
    position: Vec2
    velocity: Vec2
    tag: char[32]
end
```

Aggregate initialization follows field order:

```dbas
position: Vec2 = {10.0f, 20.0f}
```

### Packed and aligned structs

```dbas
struct PackedHeader packed
    kind: byte
    size: int
end

struct TwoByteAligned align(2)
    kind: byte
    size: int
end
```

`packed` removes normal padding. `align(n)` changes the field-alignment cap used by the current layout implementation. Always verify extenally shared layouts with `sizeof` and native ABI documentation.

### Untagged unions

```dbas
union NumberBits
    whole: int
    firstByte: char
end
```

All members overlap the same storage. Writing one member changes the bytes observed through the others.

### Tagged unions

```dbas
tagged union Value
    integer: int
    decimal: float
    text: str
end

first: Value = Value(42)
second: Value = Value(2.5f)
third: Value = Value("hello")
```

The constructor chooses a variant by argument type. Variant types must be distinguishable. Use `select type` to inspect a tagged union safely; see the control-flow section.

## 14. Enums, distinct types, and bit sets

### Enums

```dbas
enum Direction
    NORTH = 2
    EAST = 7
    SOUTH = 11
end
```

Unassigned members auto-increment. Explicit expressions, trailing commas, and inline comments are accepted.

When the expected enum type is known, a member can use shorthand notation:

```dbas
direction: Direction = .EAST
```

Iterate over values and zero-based ordinals:

```dbas
for member, ordinal in Direction
    printf("value=%d ordinal=%d\n", member, ordinal)
end
```

An enum can size and index an array:

```dbas
enum Slot
    SLOT_A
    SLOT_B
    SLOT_C
end

values: int[Slot] = {10, 20, 30}
slot: Slot = .SLOT_B
printf("%d\n", values[slot])
```

### Distinct types

Distinct types create nominally different domains even when they share storage:

```dbas
type UserId = distinct int
type WindowId = distinct int

user: UserId = (UserId)501
raw: int = (int)user
```

Explicit casts make domain crossings visible.

### Enum-backed bit sets

```dbas
enum Permission
    READ
    WRITE
    EXECUTE
end

type Permissions = bitset[Permission]

permissions: Permissions = {READ, WRITE}
permissions += EXECUTE
permissions -= WRITE

if READ in permissions And WRITE not_in permissions then
    # ...
end
```

The current bit-set representation supports enum values from 0 through 31.

## 15. Type aliases, spans, and function pointer types

Regular aliases:

```dbas
type IntSpan = span<int>
type Callback = function(value: int, user: void*) -> int
type NotifyCallback = sub(message: char*)
```

Function symbols can be stored and called:

```dbas
function Add(a: int, b: int) -> int
    return a + b
end

let operation = Add
result: int = operation(6, 7)
```

### Spans and slices

A `span<T>` is a non-owning `{data, len}` view:

```dbas
type IntSpan = span<int>

numbers: int[4] = {10, 20, 30, 40}
view: IntSpan
view.data = &numbers[0]
view.len = 4

middle: IntSpan = view[1:3]
prefix: IntSpan = view[:2]
suffix: IntSpan = view[2:]
full: IntSpan = view[:]
```

Spans support indexing and preserve element type, including structs, strings, characters, and floating-point values. They do not own or extend the lifetime of the underlying memory.

## 16. Functions and subs

A `function` returns one or more values. A `sub` has no result:

```dbas
function Add(a: int, b: int) -> int
    return a + b
end

sub PrintMessage(message: char*)
    printf("%s\n", message)
end
```

### Default and named arguments

```dbas
function Mix(a: int, b: int = 20, c: int = 300) -> int
    return a + b + c
end

first: int = Mix(1)
second: int = Mix(c=3, a=1, b=2)
```

Named arguments may be reordered. Required arguments must still be supplied.

### Inline routines

`inline` expands the routine body directly at the call site. Inline functions and subs use ordinary DirtyBasic bodies: locals, assignments, `if`/`elseif`/`else`, `select`, loops, state changes, and early `return` statements are supported.

```dbas
inline function ClampIndex(index: int, count: int) -> int
    if count <= 0 then return 0
    if index < 0 then return 0
    if index >= count then return count - 1
    adjusted: int = index
    return adjusted
end

inline sub Accumulate(value: int)
    if value < 0 then return
    total = total + value
end
```

Methods declared inside a class can also be inline. They retain normal method
syntax at the call site and receive `self` automatically:

```dbas
class Counter
    value: int

    inline function Add(delta: int) -> int
        self.value += delta
        return self.value
    end
end

counter: Counter* = New Counter()
result: int = counter.Add(5)
```

Every function return writes one call-site result slot and jumps to a shared inline exit, so early returns preserve normal function semantics without emitting a hidden call. An inline sub may use bare `return`. Simple one-expression functions retain a smaller fast expansion path. Recursive calls and expansion nesting beyond the compiler limit remain real calls to prevent unbounded code generation.

Inline structure arguments use normal by-value semantics: the expanded routine receives a private copy, so assigning to its fields does not modify the caller. Inline routines can return structures directly and can mix structures with scalar values in multiple-result tuples. Result storage is owned by the caller scope, which keeps nested expressions and tuple destructuring valid without emitting a hidden function call.

```dbas
struct Pair
    left: int
    right: int
end

inline function MakePair(left: int, right: int) -> Pair
    value: Pair = {left, right}
    return value
end

inline function PairAndTotal(left: int, right: int) -> (Pair, int)
    value: Pair = {left, right}
    return value, left + right
end

pair: Pair
total: int
pair, total = PairAndTotal(11, 12)
```

Inline functions also support C-style varargs. Include `stdarg.dbas`, keep at least one fixed argument for `va_start`, and read the promoted arguments with the normal `va_arg_*` helpers. Small integer arguments are promoted to `int`; `float` arguments are promoted to `double`, matching ordinary vararg calls.

```dbas
from "../include/stdarg.dbas" import *

inline function Sum(count: int, ...) -> int
    args: va_list = va_start(&count)
    total: int = 0
    for index in range(count)
        total += va_arg_int(&args)
    end
    va_end(&args)
    return total
end
```

Exported inline routines still receive a normal native symbol for DLL/SO callers.
### Multiple results

Unnamed multiple results can mix types:

```dbas
function BuildConnection() -> (int, bool, char*, float)
    return 8080, true, "tcp", 1.5f
end

let port, valid, protocol, scale = BuildConnection()
```

Homogeneous results can initialize an explicitly typed name list:

```dbas
function Swap(x: int, y: int) -> (int, int)
    return y, x
end

left, right: int = Swap(10, 20)
```

Named results are initialized automatically and can be returned with a bare `return`:

```dbas
function Divide(dividend: int, divisor: int) -> (quotient: int, remainder: int, valid: bool)
    if divisor == 0 then
        return
    end
    quotient = dividend / divisor
    remainder = dividend % divisor
    valid = true
    return
end
```

Falling through the end of a named-result function returns the current named values.

### Result propagation

The propagation operators expect a multiple-result expression whose final result is `bool`:

```dbas
function TryHalf(value: int) -> (result: int, valid: bool)
    if value < 0 then
        return 0, false
    end
    return value / 2, true
end

function ForwardHalf(value: int) -> (result: int, valid: bool)
    result = TryHalf(value) or_return
    result += 1
    valid = true
    return
end

fallback: int = TryHalf(-4) or_else 99

function TryPair(value: int) -> (int, int, bool)
    if value < 0 then
        return 0, 0, false
    end
    return value, value + 10, true
end

function DefaultPair() -> (int, int)
    return 90, 91
end

function ForwardPair(value: int) -> (first: int, second: int, valid: bool)
    first, second = TryPair(value) or_return
    first += 1
    second += 1
    valid = true
    return
end

left, right: int = TryPair(-1) or_else DefaultPair()
```

- `or_return` propagates failure from the enclosing function.
- `or_else value` substitutes a fallback value on failure.
- `or_continue` skips to the next loop iteration on failure.
- `or_break` exits the current loop on failure.

Propagation is not limited to one value plus a status. Any result list whose final value is `bool` can propagate: on success the final status is removed and every preceding value remains available for destructuring; on failure `or_return` returns zero-initialized values plus `false` from either named or unnamed enclosing result lists. A multi-value `or_else` fallback must return the same value layout as the successful prefix.

## 17. Interfaces, classes, and properties

### Interfaces

```dbas
interface IDrawable
    sub Draw(x: int, y: int)
    function GetWidth() -> int
end
```

### Classes and lifecycle

```dbas
class Sprite
    implements IDrawable

    x: int
    y: int
    visible: bool

    sub __init__(startX: int = 0, startY: int = 0)
        self.x = startX
        self.y = startY
        self.visible = true
    end

    sub __del__()
        printf("Sprite destroyed\n")
    end

    sub Draw(x: int, y: int)
        self.x = x
        self.y = y
    end

    function GetWidth() -> int
        return 32
    end
end
```

Heap construction calls `__init__`; `Delete` calls `__del__` and releases the object:

```dbas
sprite: Sprite* = New Sprite(10, 20)
sprite->Draw(30, 40)
Delete sprite
```

Value construction is also supported:

```dbas
spriteValue: Sprite = Sprite(10, 20)
```

### Properties

```dbas
class Sprite
    visible: bool

    property bool Visible
        get:
            return self.visible
        set:
            self.visible = value
    end
end
```

`get:` and `set:` keep their colons and do not have individual `end` markers. `value` is the implicit incoming value in a setter.

### Operator overloading

Classes can overload arithmetic and comparison operators:

```dbas
class Vec2
    x: int
    y: int

    function operator+(rhs: Vec2*) -> Vec2
        result: Vec2
        result.x = self.x + rhs->x
        result.y = self.y + rhs->y
        return result
    end

    function operator==(rhs: Vec2*) -> bool
        return self.x == rhs->x And self.y == rhs->y
    end
end
```

The regression suite covers `+`, `-`, `*`, `/`, `%`, `==`, `!=`, `<`, `<=`, `>`, and `>=`.

## 18. Conditional control flow

### If chains

```dbas
if value < 0 then
    label = "negative"
elif value == 0 then
    label = "zero"
elseif value < 10 then
    label = "small"
else
    label = "large"
end
```

`elif` and `elseif` are equivalent.

### Single-line if

A standalone `if` with its statement after `then` is complete on that line and does not use `end`:

```dbas
if row == 0 then return 7
if alpha < 0 then alpha = 0
if ready then StartWork()
```

Only one simple statement is allowed after `then`. A branch beginning another block must use the multiline form. `elif`/`elseif`/`else` chains are also multiline and retain one final `end`.

If an initializer is useful for only one action, it can also be inline:

```dbas
if let value = ReadValue(); value < 0 then ReportNegative(value)
```

### If initializers

An initializer before the semicolon is scoped to the complete if/elseif/else chain:

```dbas
if let value = ReadValue(); value < 0 then
    printf("negative: %d\n", value)
elseif value == 0 then
    printf("zero\n")
else
    printf("positive: %d\n", value)
end
```

Typed, `let`, `var`, `const`, and multi-variable declarations are accepted in the initializer:

```dbas
if let parsed, valid = ParsePair(); valid then
    Use(parsed)
end
```

The initialized names are not visible after the chain ends.

### Compile-time `when`

`when` uses if-like syntax, but inactive branches are removed before normal parsing and code generation:

```dbas
when FEATURE_LEVEL == 2 then
    value = 42
elseif FEATURE_LEVEL == 3 then
    value = 43
else
    value = 0
end
```

This differs from `!if`: `when` lives in normal source structure and is useful for compile-time selection within routines.

## 19. Loops and iteration

### While

```dbas
while running
    if ShouldStop() then
        break
    end
    Update()
end
```

### Numeric ranges

```dbas
for i in range(5)          # 0 through 4
end

for i in range(3, 8)       # 3 through 7
end

for i in range(0, 20, 3)   # 0, 3, 6, ...
end
```

Interval syntax is also available:

```dbas
for i in 0..<4 # half-open: 0, 1, 2, 3
end

for i in 1..3  # inclusive: 1, 2, 3
end
```

### Foreach, reverse, and reference iteration

```dbas
values: int[4] = {1, 2, 3, 4}

for value in values
    printf("%d\n", value)
end

for value in reverse values
    printf("%d\n", value)
end

for &value in values
    value *= 3
end

for &value in reverse values
    value += 1
end
```

`&value` binds the loop name by reference, allowing mutation of the original element.

### Unrolled loops

```dbas
unroll for i in 0..<6
    ProcessConstantIndex(i)
end
```

The interval bounds must be compile-time integers. The current maximum is 1024 generated iterations. `break` and `continue` are supported in the unrolled body.

### Loop result propagation

```dbas
for i in 0..<count
    value: int = TryRead(i) or_continue
    total += value
end

while true
    value: int = TryNext() or_break
    total += value
end
```

## 20. Select statements

Plain `select` supports values, defaults, interval cases, and comma-separated alternatives that share one body:

```dbas
select value
case 0:
    label = "zero"
case 1, 3, 5:
    label = "odd-small"
case 2, 4, 6..<10:
    label = "small"
case 10..20:
    label = "medium"
default:
    label = "large"
end
```

Cases may include a runtime guard after their value or interval patterns:

```dbas
select status
case 1, 2 when connectionReady And retryCount < 3:
    action = "retry"
case 1, 2:
    action = "wait"
case 3..5 when verbose:
    action = "trace"
default:
    action = "ignore"
end
```

The guard is evaluated only after a case pattern matches. If it is false, matching continues at the next case, so an unguarded case may intentionally repeat a pattern used by an earlier guarded case. The `when` after a case pattern is contextual runtime syntax and does not conflict with the statement-level compile-time `when` block.

An unguarded case makes its patterns final for the remainder of that `select`. A later value or interval that overlaps any part of a previous unguarded case is rejected as unreachable. Alternatives within one comma-separated case must not overlap each other.

Guarded cases do not count toward `exhaustive select` coverage because their condition may be false. Add an unguarded case for every required enum member or provide `default:` when appropriate.

### Exhaustive and partial enum selection

```dbas
function DirectionScore(direction: Direction) -> int
    exhaustive select direction
    case .NORTH:
        return 20
    case .EAST:
        return 70
    case .SOUTH:
        return 110
    end
end
```

`exhaustive select` requires every enum member to be covered. `partial select` allows missing members without requiring a `default:` branch.

### Tagged-union type selection

```dbas
value: Value = Value(42)

exhaustive select type value
case int:
    printf("integer=%d\n", value.integer)
case float:
    printf("decimal=%.2f\n", (double)value.decimal)
case str:
    printf("text=%s\n", value.text)
end
```

Plain and partial type selects are also available:

```dbas
partial select type value
case str:
    PrintText(value.text)
end
```

## 21. Match expressions

`match` produces a value and keeps its colon:

```dbas
function StateName(state: int) -> char*
    return match state:
        0 => "Idle"
        1 => "Running"
        2 | 3 => "Finished"
        4..8 => "Archived"
        _ => "Unknown"
    end
end
```

String altenatives are supported:

```dbas
group: int = match command:
    "open" | "load" => 1
    "save" | "write" => 2
    _ => 0
end
```

`_` is the fallback pattern and `|` separates altenatives.

## 22. Errors, cleanup, and low-level jumps

### Try, catch, and throw

```dbas
try
    if failed then
        throw 42
    end
    printf("success\n")
catch:
    printf("error code: %d\n", Err)
end
```

`throw` transfers control to the nearest `catch:`. The thrown status is available as `Err`. Nested try blocks are supported.

### Defer

`defer` runs at scope exit in last-in, first-out order, including exits caused by `return` or `throw`:

```dbas
file: FILE* = fopen(path, "rb")
if file == NULL then
    return 1
end
defer fclose(file)
```

Repeated wrappers such as `defer defer Flush()` are accepted and replay the enclosed action once at scope exit. They participate in the same LIFO ordering as ordinary deferred actions; this is mainly useful to generated code that composes cleanup wrappers.

### Using

`using` associates a value with explicit deferred cleanup:

```dbas
using outer: int = Acquire(1) defer Release(outer)
    using inner: int = Acquire(2) defer Release(inner)
        Work(outer, inner)
    end
end
```

### Goto and labels

```dbas
goto retry

retry:
attempts += 1
```

Labels retain their colon. `pass` can be used as an explicit empty statement.

## 23. Strings and interpolation

Import `string.dbas` for interpolation and string helpers:

```dbas
from "../include/string.dbas" import *

name: str = "DirtyBasic"
message: str = f"Hello, {name}!"
second: str = $"Welcome, {name}"
```

Common string methods include:

- `Concat`, `Append`
- `ToUpper`, `ToLower`
- `Trim`
- `Contains`, `StartsWith`, `EndsWith`, `IsEmpty`
- `Replace`
- `IndexOf`, `LastIndexOf`
- `Substring`, `Left`, `Right`
- `Reverse`, `Repeat`, `PadLeft`, `PadRight`
- `ToInt`, `ToFloat`

Example:

```dbas
trimmed: str = "  hello  ".Trim()
text: str = trimmed.ToUpper()
part: str = "abcdef".Substring(2, 3)
number: int = "42".ToInt()
```

String methods can be chained directly; each method receives the previous method's result:

```dbas
clean: str = "  dirty".Trim().ToUpper().Append("BASIC")
# clean == "DIRTYBASIC"
```

String helpers return runtime-managed temporary C-compatible strings. Copy data into owned storage when it must survive later helper calls or long-lived API use.

## 24. Caller metadata and derived reflection

Capture the current mapped source location and the exact source text of an expression:

```dbas
value: int = 21
expression: char* = caller_expression(value * (2 + 1))
location: char* = caller_location()
```

`caller_location()` produces a `.dbas` path with line and column information. `caller_expression(...)` returns the argument text as written in source.

Derived metadata can be generated for structs and classes:

```dbas
!reflect
struct Person
    name: char*
    age: int
end
```

`!reflect` and `!derive(reflect)` generate the common reflection helpers:

```text
Person_DebugName()
Person_FieldCount()
Person_FieldName(index)
Person_FieldType(index)
```

Every supported mode keeps those common helpers, so changing an existing
`!derive(reflect)` declaration to a more specialized mode does not remove basic
field enumeration. The specialized modes add distinct helpers:

| Mode | Additional generated helpers | Purpose |
|---|---|---|
| `reflect` | none | Minimal type and field enumeration. |
| `debug` | `Type_DebugField(index)` | Returns a compact `"fieldName: fieldType"` description for logs and debugger views. |
| `inspector` | `Type_InspectorLabel(index)` | Returns a display label derived from the field name. Snake case and camel case names become labels such as `"Display Name"` and `"Max Items"`. |
| `editor` | `Type_EditorKind(index)`, `Type_EditorWritable(index)` | Supplies a default editor category and whether generic editors may safely write the field directly. |

Editor kinds are `"checkbox"` for `bool`, `"integer"` for integral types,
`"number"` for floating-point types, `"text"` for `char*` and `str`,
`"pointer"`, `"array"`, `"callable"`, or `"value"`. Generic pointer, array,
and callable fields are reported as non-writable because an editor cannot infer
their ownership or mutation rules. Text fields remain writable.

Modes may be combined; their generated helpers are the union of the requested
features:

```dbas
!derive(debug, inspector, editor)
struct Settings
    displayName: char*
    enabled: bool
end

printf("%s\n", Settings_DebugField(0))       # displayName: char*
printf("%s\n", Settings_InspectorLabel(0))   # Display Name
printf("%s\n", Settings_EditorKind(1))       # checkbox
```

All index-based helpers return an empty string for an invalid index.
`Type_EditorWritable(index)` returns `false` for an invalid index.
## 25. Native libraries and hardened FFI contracts

Native code is an explicit unsafe boundary. Every import must be declared inside an `unsafe dll` block; raw `dll` blocks and top-level `extern` declarations are rejected. The block has no colon and closes with `end`:

```dbas
unsafe dll "kernel32.dll"
    extern function WINAPI GetTickCount() -> uint
    extern function WINAPI GetStdHandle(index: int) -> void* returns_borrowed last_error
    extern sub WINAPI Sleep(milliseconds: uint)
end
```

`public` or `private` may prefix an individual extern declaration. A `private unsafe dll` block also supplies a private default for its externs; an explicit `public extern` overrides that block default.

Linux shared objects use the same construct. Pointer arguments and results can carry contracts:

```dbas
unsafe dll "libc.so.6"
    extern function strlen(text: char* in nonnull borrowed) -> int
    extern function memcpy(destination: void* out nonnull borrowed len(count), source: void* in nonnull borrowed len(count), count: int) -> void* returns_borrowed
    extern function malloc(size: int) -> void* returns_owned
    extern sub free(memory: void* in nullable owned)
end
```

Argument contracts:

- `in`, `out`, and `inout` document the native direction of a pointer.
- `nonnull` always rejects explicit null constants at compile time and dynamically checks pointer expressions when `null_checks(on)` is active; `nullable` records that null is accepted.
- `owned` means ownership is transferred to the callee; `borrowed` means it is not transferred.
- `len(count)` links a pointer to an integer length argument. The referenced argument may appear later in the parameter list.

Result contracts:

- `returns_owned` and `returns_borrowed` record pointer ownership.
- `must_close` requires `returns_owned` and marks a resource result that needs an explicit release operation.
- `last_error` records that a non-void result participates in the platform error convention.

The compiler rejects contracts on non-pointer values, contradictory nullability or ownership, invalid `len(...)` targets, invalid result ownership, `must_close` without `returns_owned`, explicit null passed to `nonnull`, conflicting duplicate declarations, raw `dll`, and top-level `extern`. String and raw pointer spellings are treated as ABI-equivalent when checking duplicate declarations.

Supported calling-convention markers include `WINAPI`, `cdecl`, `stdcall`, `__cdecl`, and `__stdcall`. Contracts make the boundary visible and catch declaration/call mistakes, but native code can still violate a contract or corrupt memory. Keep declarations ABI-accurate, use distinct handle types in wrappers, release owned resources deterministically, and regression-test each native interface.

### DirtyBind: turn a C or C++ header into a DirtyBasic interface

DirtyBind lets a DirtyBasic program call functions contained in a native Windows `.dll` or Linux `.so` without manually rewriting the library's C declarations in DirtyBasic.

The most important point is that three different files have three different jobs:

| File | What it contains | Who creates it |
|---|---|---|
| `sdk.h` | C declarations describing functions, structs, enums, and callbacks | The library author |
| `sdk.dll` or `libsdk.so` | The compiled native machine code that performs the work | The library author or you |
| `sdk.dbi` | The same public interface written in syntax the DirtyBasic compiler understands | DirtyBind |

DirtyBind creates the `.dbi`. It does **not** create the original DLL/SO from only a header, and it does not copy native code into the `.dbi`.

The normal C workflow is:

```text
sdk.h ---------------------> DirtyBind ---------------------> sdk.dbi
sdk.dll / libsdk.so ----------------------------------------------+
                                                                  |
program.dbas imports sdk.dbi -> DirtyBasic executable -> loads ---+
```

At compile time, DirtyBasic reads `sdk.dbi` to learn the native function names, argument types, return types, structures, and calling conventions. At run time, the compiled program loads `sdk.dll` or `libsdk.so` and calls the real native functions.

DirtyBind is integrated into `DirtyLPL.exe`; there is no separate DirtyBind executable. The implementation lives in `include/selfhost/selfhost_dirtybind.dbas`, with C++ bridge generation in `include/selfhost/selfhost_dirtybind_cpp.dbas`.

#### When DirtyBind is useful

Use DirtyBind when all of the following are true:

1. You have a native C or C++ library, such as a physics engine, codec, database, GUI toolkit, or small custom DLL.
2. You have one or more header files describing its public API.
3. You want DirtyBasic declarations without manually translating hundreds of functions and structures.

You do not need DirtyBind for a library that already supplies a correct `.dbi`.

#### Requirements

- A 64-bit native library matching the platform you will run on.
- Its C or C++ header files.
- Clang installed and available as `clang`, or supplied with `--clang`.
- For C++, a C++ compiler is also needed to compile the generated bridge.

DirtyBasic, the header, and the library must agree on architecture, packing, calling convention, and compiler ABI. A 32-bit DLL cannot be loaded by a 64-bit DirtyBasic program.

#### Small C example from start to finish

Suppose `mathsdk.h` contains:

```c
#include <stdint.h>

typedef struct MathPoint {
    int32_t x;
    int32_t y;
} MathPoint;

int32_t math_add(int32_t a, int32_t b);
MathPoint math_make_point(int32_t x, int32_t y);
```

The native implementation must already be built as:

- `mathsdk.dll` on Windows; and/or
- `libmathsdk.so` on Linux.

Generate the DirtyBasic interface:

```powershell
.\DirtyLPL.exe --dirtybind mathsdk.h --out mathsdk.dbi --dll mathsdk.dll --so libmathsdk.so
```

The command is one line. PowerShell backticks may be used only to split it visually:

```powershell
.\DirtyLPL.exe --dirtybind mathsdk.h `
    --out mathsdk.dbi `
    --dll mathsdk.dll `
    --so libmathsdk.so
```

DirtyBind produces a `mathsdk.dbi` resembling:

```dbas
struct MathPoint
    x: int
    y: int
end

!if defined(SH_TARGET_LINUX_X64)
unsafe dll "libmathsdk.so"
!else
unsafe dll "mathsdk.dll"
!endif
    extern function math_add(a: int, b: int) -> int
    extern function math_make_point(x: int, y: int) -> MathPoint
end
```

Use it from DirtyBasic:

```dbas
from "mathsdk.dbi" import *
from "include/stdio.dbas" import *

function main() -> int
    total: int = math_add(20, 22)
    point: MathPoint = math_make_point(10, 15)
    printf("total=%d point=(%d,%d)\n", total, point.x, point.y)
    return 0
end
```

Compile normally:

```powershell
.\DirtyLPL.exe program.dbas program.exe
```

For the program to start, put `mathsdk.dll` next to `program.exe`, or in another directory searched by Windows. On Linux, make `libmathsdk.so` available through the executable's loader path, rpath, or `LD_LIBRARY_PATH`.

The `.dbi` is needed while compiling. The `.dll`/`.so` is needed while running.

#### Files generated by DirtyBind

For:

```powershell
.\DirtyLPL.exe --dirtybind sdk.h --out sdk.dbi --dll sdk.dll --so libsdk.so
```

DirtyBind writes:

| Generated file | Purpose |
|---|---|
| `sdk.dbi` | Import this from a DirtyBasic program |
| `sdk.dbi.abi.c` | Native C program that prints C structure sizes and field offsets |
| `sdk.dbi.abi.dbas` | DirtyBasic program that prints the same sizes and offsets |

The `.dbi` may contain:

- enums;
- normal, packed, and aligned structs;
- unions;
- aliases;
- callback function types;
- native functions and subs;
- Windows/Linux library selection;
- Windows LLP64 versus Linux LP64 mappings for C `long`, `unsigned long`, and `wchar_t`;
- pointer direction, nullability, ownership, and length contracts supplied by a manifest.

Do not edit a generated `.dbi` to fix the generator output. Prefer fixing the input wrapper header, contract manifest, or DirtyBind itself, then regenerate it.

#### Why the ABI probe files matter

C and DirtyBasic must calculate identical sizes and field offsets. If C thinks a struct is 16 bytes while DirtyBasic thinks it is 12 bytes, passing that struct can corrupt arguments or memory.

Compile both generated probes:

```powershell
gcc sdk.dbi.abi.c -o sdk_abi_c.exe
.\DirtyLPL.exe sdk.dbi.abi.dbas sdk_abi_dbas.exe
```

Run and compare them:

```powershell
.\sdk_abi_c.exe > c-layout.txt
.\sdk_abi_dbas.exe > db-layout.txt
Compare-Object (Get-Content c-layout.txt) (Get-Content db-layout.txt)
```

`Compare-Object` should print nothing. Compile the C probe with the same target, packing defines, include paths, and ABI-related compiler flags as the real native library.

#### Pointer contracts: information missing from ordinary C headers

A declaration such as:

```c
size_t sdk_copy(void *destination, const void *source, size_t count);
```

does not tell a tool whether either pointer may be null, which pointer is written, or whether the function takes ownership. DirtyBind therefore uses conservative defaults unless an optional JSON contract manifest supplies that information.

Example `sdk.contracts.json`:

```json
{
  "library": {
    "windows": "sdk.dll",
    "linux": "libsdk.so"
  },
  "functions": {
    "sdk_copy": {
      "params": {
        "destination": {
          "direction": "out",
          "nullability": "nonnull",
          "ownership": "borrowed",
          "len": "count"
        },
        "source": {
          "direction": "in",
          "nullability": "nonnull",
          "ownership": "borrowed",
          "len": "count"
        }
      }
    },
    "sdk_create": {
      "returns": ["owned", "must_close"]
    }
  }
}
```

Pass it while generating:

```powershell
.\DirtyLPL.exe --dirtybind sdk.h --out sdk.dbi --dll sdk.dll --so libsdk.so --contracts sdk.contracts.json
```

The resulting declarations can include:

```dbas
extern function sdk_copy(destination: void* out nonnull borrowed len(count), source: void* in nonnull borrowed len(count), count: uint64) -> uint64
extern function sdk_create() -> void* returns_owned must_close
```

The words mean:

- `in`: native code reads through the pointer.
- `out`: native code writes through the pointer.
- `inout`: native code may read and write.
- `nonnull`: null is not accepted.
- `nullable`: null is accepted.
- `borrowed`: ownership remains with the caller.
- `owned`: ownership is transferred.
- `len(count)`: the pointer describes `count` elements or bytes, according to the API.
- `returns_owned`: the caller owns the returned pointer or handle.
- `returns_borrowed`: the library retains ownership.
- `must_close`: the returned resource requires the library's matching destroy/close call.
- `last_error`: the result participates in the platform's last-error convention.

Contracts describe and validate the boundary; they do not make an incorrect native library safe. DirtyBind does not automatically free `returns_owned` results. Your program must call the library's matching destroy, free, release, or close function.

An explicit null passed to `nonnull` is rejected at compile time. Dynamic pointer expressions are checked when `!pragma null_checks(on)` is active.

#### C++ classes need a bridge

DirtyBasic cannot directly call a normal C++ class ABI reliably. C++ exports use name mangling, compiler-specific object layouts, constructors, destructors, overloads, and exception rules.

For:

```cpp
class Counter {
public:
    explicit Counter(int initial);
    ~Counter();
    int add(int amount);
    int value() const;
    static int twice(int value);
};
```

request a bridge:

```powershell
.\DirtyLPL.exe --dirtybind counter.hpp `
    --out counter.dbi `
    --dll counter.dll `
    --so libcounter.so `
    --language c++ `
    --cpp-bridge generated/counter_bridge `
    --contracts counter.contracts.json
```

This additionally creates:

- `generated/counter_bridge.h`;
- `generated/counter_bridge.cpp`;
- `counter.dbi`;
- the two ABI probe files.

The generated bridge turns the class into ordinary C functions:

```c
void *Counter_create(int initial);
void Counter_destroy(void *self);
int Counter_add(void *self, int amount);
int Counter_value(void *self);
int Counter_twice(int value);
```

You must compile `counter_bridge.cpp` as C++ and link it with the original class implementation or original C++ library. For a small MSVC example:

```powershell
cl /std:c++17 /EHsc /LD counter.cpp generated\counter_bridge.cpp /Fe:counter.dll
```

Then DirtyBasic uses the class through an opaque pointer:

```dbas
from "include/stdio.dbas" import *
from "counter.dbi" import *

function main() -> int
    counter: void* = Counter_create(10)
    if counter == NULL then
        return 1
    end

    value: int = Counter_add(counter, 7)
    printf("value=%d\n", value)

    Counter_destroy(counter)
    return 0
end
```

The bridge maps:

- public constructors to `_create` functions;
- destructors to `_destroy`;
- instance methods to functions whose first argument is `void* self`;
- static methods to normal C functions.

Generated bridge functions catch C++ exceptions so exceptions do not unwind through DirtyBasic. Templates, operators, overloaded/reference-heavy APIs, and signatures that cannot be represented safely are skipped rather than guessed. For a complicated C++ library, write a small hand-designed C wrapper header and use DirtyBind on that wrapper.

#### Common mistakes

- **Only supplying a header:** a header contains declarations, not executable code. The DLL/SO must also exist.
- **Not compiling the C++ bridge:** generation alone does not add the bridge functions to a DLL.
- **Wrong library filename:** `--dll` and `--so` must match the files available at run time.
- **DLL not found:** on Windows, put it next to the executable while testing.
- **32-bit/64-bit mismatch:** DirtyBasic targets x64; the native library must also be x64.
- **ABI probe mismatch:** do not ignore it. Fix packing, types, defines, or calling conventions first.
- **Clang cannot parse the vendor header:** create a small wrapper header with the needed includes/defines, or pass a suitable Clang executable with `--clang`.
- **Assuming ownership:** add and review a contract manifest for pointer-heavy APIs.
- **Editing generated output:** regeneration will overwrite it; fix the header, manifest, or generator instead.

#### Command options

- `--dirtybind header.h` or `--bind header.h`: run DirtyBind for that header.
- `--out file.dbi`: output interface filename.
- `--dll name.dll`: native filename embedded for Windows.
- `--so libname.so`: native filename embedded for Linux.
- `--contracts file.json`: pointer/result contract manifest.
- `--clang path`: select the Clang executable.
- `--language c` or `--language c++`: select parsing mode.
- `--cpp-bridge path/prefix`: generate a C ABI bridge for supported C++ classes.
- `--keep-temp`: preserve Clang AST and error files for generator diagnostics.

#### Working examples in this repository

- `tests/dirtybind_fixture.h` and `tests/dirtybind_fixture.generated.dbi`: C functions, callbacks, enum, normal struct, packed struct, union, and contracts.
- `tests/dirtybind_cpp_fixture.hpp` and `tests/dirtybind_cpp_bridge.cpp`: a C++ class and its generated C bridge.
- `tests/dirtybind_cpp_runtime.dbas`: DirtyBasic creating, calling, and destroying the bridged C++ object.
- `tests/dirtybind_regression.ps1`: regenerates everything, compares ABI layouts, builds the libraries, and runs the bindings on Windows and WSL/Linux.

Run the complete tested example with:

```powershell
.\tests\dirtybind_regression.ps1
```

## 26. Form syntax

The Form sugar provides declarative controls over either the D2D GUI or native GUI runtime. It is part of the compiler front end and lowers to normal DirtyBasic runtime calls. D2D Form programs use `include/DirtyLittleUI.dbi`, so generated executables call the shared GUI library instead of embedding its implementation.

### D2D form

```dbas
from "../include/DirtyLittleUI.dbi" import *

form MainWindow as D2D
    title = "DirtyBasic D2D Form"
    width = 640
    height = 420
    theme = "frosted-pearl"
    style = "3d"
    titlebar = true
    visible = true

    label lblStatus
        x = 24
        y = 24
        width = 300
        height = 28
        text = "Ready"
    end

    button btnHello
        x = 24
        y = 72
        width = 130
        height = 34
        text = "Say hello"

        on click
            lblStatus.Text = "Hello!"
        end
    end
end

function main() -> int
    D2D.Run()
    return 0
end
```

D2D controls have two rendering styles:

- `style = "flat"` preserves the original flat renderer.
- `style = "3d"` uses raised, pressed, recessed, highlighted, and shadowed surfaces.
- `style = "theme"` lets the selected theme choose its intended style.

Frosted Pearl (`"frosted-pearl"`) and Midnight Vibrancy (`"midnight-vibrancy"`, with `"midnight"` as an alias) default to 3D. Existing themes default to Flat. Code that does not use Form sugar can call `D2DGUI_SetControlStyle(D2DGUI_STYLE_FLAT)` or `D2DGUI_SetControlStyle(D2DGUI_STYLE_3D)` before `NewD2DForm()`. Use `D2DGUI_UseThemeControlStyle()` to return to the theme default, or `D2DFormSetControlStyle(form, style)` to change one form.

Normal Windows palettes are available as `theme = "windows-light"` and `theme = "windows-dark"` (short aliases: `"win-light"` and `"win-dark"`). Both default to Flat and support either `style = "flat"` or `style = "3d"`; the 3D variant uses restrained native-desktop bevels while retaining the same neutral palette.

The style affects GUI surfaces such as panels, buttons, text inputs, lists, combo boxes, check/radio controls, progress bars, knobs, sliders, and toggles. Shape controls remain literal drawing primitives and are not automatically beveled.

### Native GUI form

Use `as GUI` with `include/gui.dbas`:

```dbas
from "../include/gui.dbas" import *

form MainWindow as GUI
    title = "Native GUI"
    width = 520
    height = 300
    visible = true

    button btnClose
        text = "Close"
        x = 380
        y = 220
        width = 100
        height = 32

        on click
            MainWindow.CloseForm()
        end
    end
end

function main() -> int
    GUI.Run()
    return 0
end
```

Form controls are named objects. Their property block and every `on ...` event block use explicit `end` markers.

## 27. DirtyUI syntax

DirtyUI is the higher-level declarative D2D syntax. It is intentionally distinct from ordinary Form syntax while still using DirtyBasic `end`-based structure. DirtyUI uses the same `DirtyLittleUI.dbi` shared-library boundary as D2D Form syntax.

```dbas
from "../include/stdio.dbas" import *
from "../include/DirtyLittleUI_LITE.dbas" import *

DirtyUI SettingsWindow
    title = "Settings"
    width = 560
    height = 420
    theme = "modern-light"
    visible = true

    @State busy: int = 0
    @State status: char* = "Ready"

    style accentButton
        BackColor = "#2C7BE5"
        ForeColor = "#FFFFFF"
        BorderColor = "#2C7BE5"
        FontSize = 14
    end

    VStack(x=32, y=32, width=480, spacing=12)
        Text("DirtyUI settings", width=420, height=30, FontSize=22)
        Text($status, name=statusText, width=420, height=24)

        Button("Build", name=btnBuild, style=accentButton, width=120, height=34, disabled=$busy)
            on click
                busy = 1
                status = "Building"
            end
        end
    end
end

function main() -> int
    D2D.Run()
    return 0
end
```

### The one-line control rule

A DirtyUI control with no modifiers, event blocks, or child controls is complete on its own line and must not be followed by `end`:

```dbas
Text("One-line text", width=300, height=24)
Button("One-line button", width=120, height=34)
```

If a control contains modifiers or events, it becomes a block and requires `end`:

```dbas
Text("Bound visibility", width=300, height=24)
    .visible($showText)
end

Button("Save", width=120, height=34)
    .conerRadius(6)
    on click
        SaveDocument()
    end
end
```

Containers such as stacks, frames, lists, sections, sheets, and scroll views contain children and therefore require `end`.

### State and bindings

```dbas
@State showAdvanced: int = 1
@State volume: float = 0.68f
@State caption: char* = "Build"

Text($caption, width=240, height=24)
Checkbox("Advanced", checked=$showAdvanced, width=180, height=28)
HSlider("Volume", value=$volume, width=260, height=28)
```

`$name` binds a control property to DirtyUI state. Event bodies can update the underlying state name directly.

### Styles and inheritance

```dbas
style baseButton
    ForeColor = "#FFFFFF"
    FontSize = 13
    HoverScale = 1.03f
end

style dangerButton extends baseButton
    BackColor = "#D73A49"
    BorderColor = "#D73A49"
end

Button("Delete", style=dangerButton, width=120, height=34)
```

### Layout containers

Common containers and layout helpers:

- `VStack`, `HStack`, `ZStack`
- `Frame`, `Padding`, `Spacer`, `Divider`
- `ScrollView`, `List`, `Section`
- `Sheet`

DirtyUI also supports repeated UI construction:

```dbas
for each(["One", "Two", "Three"], as=title, indexVar=i)
    Button(title, width=118, height=34)
end
```

### Modifiers

Common modifiers include:

```dbas
.frame(width=220, height=28)
.background("#171C24")
.foregroundColor("#FFFFFF")
.borderColor("#30343C")
.fontSize(14)
.fontName("SomeFont.ttf")
.opacity(0.8f)
.rotation(8.0f)
.conerRadius(6)
.checked($enabled)
.visible($showControl)
.disabled($busy)
.hoverScale(1.04f)
```

### Events

Common event names include `click`, `change`, `scroll`, and menu events:

```dbas
TextField("name", name=nameBox)
    on change
        printf("name changed\n")
    end
end

MenuBar(name=mainMenu, menus=["File", "Help"], menu0=["Open", "Exit"], menu1=["About"])
    on menu "File", "Open"
        OpenDocument()
    end
end
```

### Common controls

The current DirtyUI/D2D layer includes, among others:

- `Text`, `TextShape`, `Button`
- `TextField`, `TextArea`, `SyntaxBox`
- `Checkbox`, `Radio`, `Toggle`
- `Slider`, `HSlider`, `VSlider`, `RangeSlider`
- `Knob`, `XYPad`, `Meter`, `Progress`, `SpinBox`, `FractionBox`
- `Picker`, `Segmented`, `Expander`
- `ListBox`, `ListView`, `PropertyGrid`, `Tabs`, `MenuBar`
- `HScrollBar`, `VScrollBar`, `Panel`, `DropZone`, `Image`
- shapes such as `Rect`, `RRect`, `Circle`, `Star`, and `Line`

See the `Samples/DirtyUI_*.dbas` files for full control and layout examples.

## 28. XML UI event decorators

The XML D2D layer can embed a layout and bind handlers with decorators:

```dbas
!embed "layout.xml" as xmlLayout

@onpress(btnRun, OnRun)
@onchange(cmbMode, OnModeChanged)

function main() -> int
    xml: D2DXmlUI*
    # Load the embedded XML through d2d_xml_gui.dbas.
    @bindevents(xml)
    D2D.Run()
    return 0
end
```

Supported decorator families include press/click, change/input, selection/value, checked/toggle, slide/drag/drop, menu, hover/enter/leave, mouse down/up/double-click, focus/blur, and `@bindevents`.

## 29. D2D Form Designer

`Samples/D2D_Design.dbas` is a visual designer and code editor built in DirtyBasic. It can:

- Create D2D and console projects.
- Save editable `.d2d` project files.
- Persist the active designer theme with the project.
- Generate `.dbas` Form code.
- Preview and compile generated applications.
- Edit control properties, events, pages, shapes, and visual scripts.
- Highlight current DirtyBasic syntax in its built-in code editor.

Generated preview files such as `_d2d_form_preview.dbas` and `_d2d_console_preview.dbas` are ordinary DirtyBasic sources and can be compiled independently for diagnostics.

## 30. Standard library overview

Important include files:

| Area | Includes |
| --- | --- |
| Core convenience | `std.dbas`, `stdio.dbas`, `stdlib.dbas`, `string.dbas`, `math.dbas`, `fmt.dbas` |
| Files and data | `path.dbas`, `serialize.dbas`, `resource.dbas`, `config.dbas`, `settings.dbas` |
| Time and arguments | `datetime.dbas`, `args.dbas`, `log.dbas` |
| Collections/results | `collection.dbas`, `result.dbas`, `reflect.dbas` |
| Native GUI | `gui.dbas`, `gtk_gui.dbas`, `winui3.dbas` |
| D2D GUI | `DirtyLittleD2D.dbas`, `DirtyLittleUI.dbas`, `DirtyLittleUI_LITE.dbas`, `d2d_xml_gui.dbas` |
| Graphics and games | `Direct2D.dbas`, `DLGL.dbas`, `raylib.dbas`, `raymath.dbas`, `retro2d.dbas` |
| Database | `database.dbas`, `sqlite.dbas` |
| Networking and tasks | `network.dbas`, `thread.dbas`, `task.dbas`, `task_http.dbas` |
| Media | `stb_image.dbas`, `stb_truetype.dbas`, `miniaudio.dbas`, `synth.dbas` |
| Platform APIs | `windows.dbas`, `commdlg.dbas`, `process.dbas`, `webview.dbas` |

Import only what an application needs, or begin with `std.dbas` for the common core.

## 31. Current boundaries

- DirtyBasic source files use the `.dbas` extension.
- Indentation is not a block delimiter; missing or extra `end` markers are syntax errors.
- Generics are not implemented.
- macOS ARM64 output is planned, not supported.
- Shared libraries are supported on Windows x64 and Linux x64; exported varargs, runtime-global symbols, and cross-boundary exception unwinding are intentionally not part of the stable ABI yet.
- Memory management is manual; there is no garbage collector.
- Opt-in automatic null checks prevent checked operations from dereferencing address zero and provide source-mapped diagnostics. They do not prevent dangling pointers, use-after-free, invalid non-null addresses, buffer overruns through raw pointers, or ABI/type mistakes.
- Native FFI is isolated behind mandatory `unsafe dll` blocks and supports compile-time contracts plus opt-in dynamic `nonnull` checks; contracts cannot protect against a native library that lies or corrupts memory.
- DirtyUI and Form are compiler-supported DSLs, but they ultimately use the included native GUI runtimes.
- Features and ABI details are still evolving, so native binary interfaces should be regression-tested when the compiler changes.

## 32. Recommended tests and examples

Start with these files:

- `Samples/syntax_test.dbas` — comprehensive executable language regression suite, including export declaration syntax.
- `Samples/shared_library_smoke.dbas` and `Samples/shared_library_smoke_consumer.dbas` — DLL/SO ABI, generated interface, class ownership, properties, aggregates, aliases, enums, unions, and constants.
- `Samples/shared_library_language_features.dbas` and `Samples/shared_library_language_features_consumer.dbas` — cross-boundary multiple assignment, array aggregates, pointers, packed/aligned layouts, default/named arguments, inline exports, multiple results, and result propagation on Windows and Linux.
- `Samples/dirtylittleui_shared_library_demo.dbas` — compact consumer of the full `DirtyLittleUI.dll`.
- `Samples/form_gui_sugar_demo.dbas` — native GUI Form syntax.
- `Samples/DirtyUI_state_demo.dbas` — state, bindings, styles, conditional visibility, and sheets.
- `Samples/DirtyUI_stack_layout_demo.dbas` — stacks, layout helpers, and repeated controls.
- `Samples/DirtyUI_control_showcase.dbas` — broad DirtyUI control showcase.
- `Samples/fluid_simulator.dbas` — interactive Stable Fluids simulation using Raylib, a dynamic texture, pointer-backed grids, and real-time FFI. It also owns the renderer-neutral solver shared by the other fluid frontends.
- `Samples/fluid_simulator_d2d.dbas` — the same simulation rendered through DirtyLittleD2D using an updateable BGRA bitmap. Mouse wheel changes the brush size.
- `Samples/fluid_simulator_dlgl.dbas` — the same simulation rendered through DLGL as a single scaled ARGB software sprite. Up/Down changes the brush size because DLGL currently exposes no mouse-wheel query.
- `Samples/xml_gui_complete_demo.dbas` — XML UI and event decorators.
- `Samples/D2D_Design.dbas` — the visual Form designer and integrated code editor.
- `tests/null_checks_regression.ps1` — positive and negative automatic-null-check regression matrix.
- `tests/visibility_regression.ps1` — public/default access plus private module, type, member, routine, constant, and native-import rejection cases.
- `tests/private_module_isolation_regression.ps1` - duplicate private identities across modules, access diagnostics, and DLL-qualified same-name private imports on Windows plus module-isolation coverage on WSL/Linux.
- `tests/dirtybind_regression.ps1` - C/C++ binding generation, contract emission, ABI probes, generated bridge compilation, and DirtyBasic runtime calls on Windows and WSL/Linux.
- `tests/compiler_performance_regression.ps1` - guards against quadratic symbol-resolution regressions by timing both a full `DirtyLittleUI.dbas` import and `Samples/D2D_Design.dbas`.
- `tests/shared_library_regression.ps1` — Windows DLL, Linux SO/WSL, the complete language-feature ABI matrix, native C ABI, generated DBI, DirtyLittleUI, compact Form/DirtyUI/designer outputs, GUI startup, hardening, and negative export regressions.
- `tests/linux_runtime_exit_regression.ps1` — Linux libc termination, buffered output flushing, and loop-propagation runtime coverage.

Run the comprehensive Windows test:

```powershell
.\DirtyLPL.exe Samples\syntax_test.dbas Samples\syntax_test.exe
.\Samples\syntax_test.exe
```

The successful run ends with:

```text
=== All tests complete ===
```

For Linux x64:

```powershell
.\DirtyLPL.exe --target linux-x64 Samples\syntax_test.dbas Samples\syntax_test
```

## 33. Quick syntax index

```text
Imports:          from "file.dbas" import * as alias
Preprocessor:     !define, !if, !elif, !elseif, !else, !endif, !pragma, !embed
Pragmas:          comment(lib, ...), null_checks(on/off)
Visibility:       public (optional/default), private (module or class scope)
Library API:      export, --shared, generated .dbi, .dll, .so
Bindings:         --dirtybind, --contracts, --cpp-bridge, generated ABI probes
Routines:         function, sub, inline, return
Declarations:     let, var, const, name: type, multi-name declarations
Types:            type, distinct, bitset, span, struct, union, tagged union
Object model:     interface, class, implements, property, get:, set:, self
Branches:         if ... then, elif, elseif, else, when ... then
Loops:            while, for ... in, reverse, &reference, unroll for
Selection:        select, exhaustive select, partial select, select type
Cases:            case:, default:, match:, =>
Flow:             break, continue, goto, pass
Errors/cleanup:   try, catch:, throw, defer, using
Results:          multiple returns, named returns, or_return, or_else,
                  or_continue, or_break
Memory:           &, *, ->, New, Delete, sizeof, addressof, opt-in null checks
Native calls:     unsafe dll, extern, in/out/inout, nonnull/nullable,
                  owned/borrowed, len, returns_owned, returns_borrowed,
                  must_close, last_error, WINAPI, cdecl, stdcall
Metadata:         caller_location, caller_expression, !reflect, !derive
UI:               form ... as GUI/D2D, DirtyUI, @State, style, on event
Block terminator: end
```
