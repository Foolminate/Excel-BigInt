# XL-Int — The BigInt Library for Excel
**Specification & Development Roadmap**

---

## Document Purpose
This document specifies the scope, design principles, public API, internal architecture, and phased development roadmap for a pure–Excel Big Integer (“BigInt”) arithmetic library.

The goal is to push Excel’s formula engine to its functional limits. The library intentionally treats Excel not as a spreadsheet, but as a deterministic, pure-functional calculation engine. Slow but correct is considered a success.

## Project Goals & Design Priorities

### The Hierarchy of Execution (Strict Order)
If a design choice forces a trade-off, the following hierarchy applies absolutely:

1. **Correctness:** Identical inputs must always produce deterministic, identical outputs. No reliance on volatile functions. No floating-point approximations.
2. **Completeness:** If an operation is mathematically definable and can be expressed with Excel formulas, it is in scope. No feature is excluded solely on the basis of calculation time.
3. **Performance:** Best-effort optimizations are applied (favoring vectorized operations like `REDUCE`, `MAP`, `BYROW`, and native array expansions over recursion and reallocation), *provided they do not compromise correctness or completeness*.
4. **Clarity:** Excel LAMBDAs are inherently dense. Algorithmic accuracy and single-responsibility composition outrank short-term readability.

### Explicit Non-Goals
- Real-time or sub-second performance for massive calculations.
- Competing with native C/Python bignum libraries.
- Avoiding Excel recalculation stress.
- Support for floating-point large numbers, fractions, or non-integer arithmetic.

## Aggregation Strategys

### Orthogonal Broadcasting
When summing arrays of BigInts, traversing the text-boundary inside an $O(N)$ `REDUCE` loop causes severe performance degradation. Instead, `XL-Int` utilizes **Native Matrix Aggregation**:
1. **Sign Grouping:** The 1D input array is split into positive and negative groups.
2. **Unified Radix:** A single, unified $k$ (chunk size) is calculated based on the total number of items to ensure identical base-10 Radix contexts across both groups.
3. **Orthogonal Broadcast:** The grouped strings are padded and array-intersected horizontally and vertically simultaneously using `--MID(...)`. This generates a massive 2D grid in a single calculation step.
4. **Vertical Collapse:** `BYROW` collapses the grid into a standard 1D Little-Endian array, executing thousands of string additions natively before resolving carries exactly once.

### Array Multiplication (1D Discrete Convolution)
For $A \times B$ multiplication (`core_BigInt.UMul`), `XL-Int` strictly avoids both $O(M)$ sequential `REDUCE` loops and $O(N \times M)$ 2D matrix generation. Instead, it utilizes a **1D Discrete Convolution (Cauchy Product)**:
1. **Target Array Mapping:** A single 1D `MAKEARRAY` maps the exact size of the final uncarried limb array ($Length_A + Length_B - 1$).
2. **Dynamic Pointer Slicing:** Inside the loop, `INDEX(..., SEQUENCE(...))` is used to dynamically slice the overlapping Little-Endian limbs of array $A$ and array $B$ for the current mathematical magnitude.
3. **Engine Hyper-Optimization:** Excel's calculation engine evaluates array slicing via `INDEX` as lightweight memory pointers rather than instantiating new COM objects. This completely bypasses the massive administrative overhead and garbage collection of 2D grids, executing pure vector math natively.
4. **Absolute Memory Safety:** The memory footprint remains strictly $O(N+M)$. This guarantees mathematical completeness and immunity from `#CALC!` memory errors all the way to the absolute string limit.

1. **The Global $k$ Baseline:** To avoid the $O(N)$ penalty of repeatedly converting limb arrays back to text to recalculate dynamic chunk sizes, `BigInt.Product` calculates a single safe $k$ based on the maximum theoretical length of the final product (`SUM(LEN(all_strings)) / 2`). All inputs are `Split` into Little-Endian numeric arrays exactly *once* at Tier 0.
2. **Vectorized Pairing:** The 1D array of numeric limb arrays is reshaped into a 2-column matrix using `WRAPROWS(..., 2, {1})` (padding odd-length arrays with the multiplicative identity).
3. **Horizontal Broadcast:** `BYROW` executes the `UMul` kernel across all pairs simultaneously, cutting the array length in half in a single parallel calculation step.
4. **Shallow Recursion:** The resulting halved array is passed back into the pairing function. This architecture resolves $N$ operands in exactly $\log_2(N)$ recursive steps, perfectly balancing intermediate matrix sizes, preventing integer overflow, and safely skirting Excel's `#NUM!` LAMBDA stack limits. String `Merge` occurs exactly once at the absolute final boundary.

### Division & Modulus (Hybrid Routing Architecture)
Division in Excel presents a unique conflict between asymptotic algorithmic complexity and the physical realities of the calculation engine's memory management. To preserve the library's performance across both small and massive arrays without triggering sequential `#CALC!` limits, `XL-Int` rejects a single-algorithm approach.

Instead, division is executed via a **Threshold Router** that dynamically selects between two disparate internal kernels based on the length of the dividend.

#### 1. The Basecase Kernel (`core_BigInt.BasecaseDiv`)
For smaller numbers, the library utilizes **Knuth's Algorithm D** (standard sequential long division).
* **Mechanics:** It iterates Left-to-Right over the dividend using a standard `REDUCE` loop, calculating the quotient and remainder naturally.
* **The Engine Advantage:** Despite being mathematically $O(N^2)$, its constant-time overhead is effectively zero. For small arrays, it executes in milliseconds.
* **The Limitation:** Because Excel cannot vectorize a mutating `REDUCE` accumulator, large dividends force thousands of sequential array reallocations, eventually freezing the engine or triggering stack limits.

#### 2. The Heavyweight Kernel (`core_BigInt.NewtonDiv`)
To safely push to the 32,766-character ceiling, the library abandons sequential loops and utilizes **Integer-Scaled Newton-Raphson Division**. It calculates $A \times (1 \div B)$ logarithmically, routing the heavy lifting through the existing $O(N+M)$ Cauchy product (`UMul`).
* **Native Float Ignition:** The top limbs of divisor $B$ (up to 14 decimal digits) are evaluated via Excel's native IEEE-754 engine to provide a highly accurate starting seed, bypassing the first several iterations.
* **The Logarithmic Loop:** The core reciprocal $R$ is refined using $R_{k+1} = \lfloor (R_k(2\beta^{2n} - B \cdot R_k)) / \beta^{2n} \rfloor$. A 32k-digit division resolves in approximately 15 `REDUCE` iterations.
* **Memory-Bounded Bit-Shifting:** To prevent exponential memory blowouts during intermediate $B \cdot R_k$ multiplications, precision is strictly managed without arithmetic. `TAKE(..., -n)` slices only the required significant limbs, and division by $\beta^{2n}$ is executed using `DROP(..., 2n)` as a hyper-fast right bit-shift.

#### 3. Unified Error Correction (`core_BigInt.UDivMod`)
Because Newton-Raphson is an approximation, it carries a strict $\pm 1$ truncation risk. Absolute mathematical correctness requires an error-correction sweep.
* Both `BasecaseDiv` and `NewtonDiv` must ultimately return a unified 2-element array: `[Quotient, Remainder]`.
* For `NewtonDiv`, the remainder is calculated explicitly via $Remainder = A - (B \times Q_{approx})$. The result of this orthogonal subtraction identifies and resolves the off-by-one quotient error.
* The public `BigInt.Div` and `BigInt.Mod` functions operate strictly as Layer 0 wrappers, executing the master `UDivMod` router and extracting their respective outputs.

#### 4. The `DivRouter` & Empirical Threshold
The active crossover threshold ($T$) between these algorithms is strictly empirical, determined by stress-testing Excel's garbage collection limits, not mathematical theory.
* `core_BigInt.DivRouter` evaluates the limb length of Dividend $A$.
* If $Length_A \le T$, route to `BasecaseDiv`.
* If $Length_A > T$, route to `NewtonDiv`.
*(Note: The exact integer value of $T$ is to be established during the testing phase by plotting calculation times of both kernels across incrementally larger arrays until the `REDUCE` reallocation penalty forces Algorithm D to cross above Newton-Raphson's constant-time overhead.)*

## Design Constraints & Axioms

**Axiom 1 — Text is the Public Data Type**
- All public inputs AND outputs are a TEXT only boundary.
- Numeric coercion is permitted ONLY inside bounded, limb-safe internal steps.
- ALL limb-safe internal steps accept and return numeric limbs, facilitating high-performance vector operations.

**Axiom 2 — The String Ceiling**
- Excel cells have a hard limit of 32,767 characters. Therefore, XL-Int has an absolute theoretical ceiling of a $\pm$32,766-digit number.

**Axiom 3 — Strict Mathematical Integers ($\mathbb{Z}$)**
- The library models strict integers. Floating-point concepts like positive/negative infinity are not supported. Canonical form forbids `-0` (which strictly normalizes to `0`).

**Axiom 4 — Dynamic IEEE-754 Chunking & Little-Endian Arrays**
- Excel stores numbers with a maximum of 15 significant digits of precision. To guarantee safe mathematical carries and borrows during 53-bit floating-point division, all numeric intermediates enforce a strict **14-digit absolute ceiling**.
- Limb sizes ($k$) are dynamically constrained by input size and the specific mathematical context (e.g., multiplication requires a smaller $k$ than addition).
- Limb arrays are strictly **Little-Endian**, simplifying logic and completely removing $O(N)$ memory reallocations (array reversals) inside the calculation engine. Top-to-bottom array iteration perfectly mirrors right-to-left mathematical evaluation.
  Given the raw input string `1234567` and `k = 3`, the array indices have the following meaning:
  1. Row 1: `567` (units - least significant)
  2. Row 2: `234` (thousands)
  3. Row 3: `1` (millions - most significant)

**Axiom 5 — Atomic Composition**
- Each named LAMBDA has one responsibility. Public-facing functions compose simpler internal primitives.

## Core Data Model & Conventions

Excel’s native formula bar and the Advanced Formula Environment (AFE) present unique constraints. The following standards ensure the library remains maintainable internally while providing a pristine, readable UX for the end-user.

### 1. Naming Conventions

**Public API (Layer 0): Terse & Standardized**
Public functions are used to compose complex mathematical formulas. Verbose names create unreadable, screen-breaking syntax.
- **Namespaces:** Uses the `BigInt.` prefix and PascalCase (e.g., `BigInt.Sum`, `BigInt.Compare`) to cleanly differentiate from native uppercase Excel functions and prevent shadowing
- **Internal Kernels:** Private helper functions are prefixed with `core_.` (e.g., `core_BigInt.UAdd`, `core_BigInt.SafeK`).
- **Math Primitives:** Use standard C/Python abbreviations (`BigInt.Add`, `BigInt.Mul`, `BigInt.Pow`, `BigInt.Fact`).
- **Clarity over Extreme Brevity:** Expand ambiguous mathematical notation where abbreviations cause confusion. Use `BigInt.Combinations` instead of `BigInt.nCr`.
- **Explicit Conversions:** Use fully qualified type names for I/O boundaries (`BigInt.ToHex`, `BigInt.FromBinary`).

**Internal Variables: The Dual-Context Rule**
Variable verbosity depends strictly on the architectural layer.
* **Layer 2 (Mathematical Kernels):** Stick to standard academic nomenclature. Use `a`, `b`, `q` (quotient), `r` (remainder), `n` (limit), and `k` (radix chunk). This minimizes cognitive friction when mapping the code to reference algorithms like Newton-Raphson or Prime Swing.
* **Layer 1 & 0 (Routers/Orchestrators):** Use explicit, highly descriptive names for state and shape management. Use `limbs_a` (denoting a little-endian array), `initial_state`, `is_same_sign`, or `needs_correction`.

### 2. Documentation & Commenting

**Docstrings (The Tooltip Contract)**
Excel's native formula tooltip truncates text aggressively. Docstrings must be ruthlessly concise. JSDoc `@param` tags are strictly banned, as Excel renders them as raw, unformatted text in the grid, destroying the UX.
* **Line 1:** State exactly what the function returns.
* **Line 2 (Optional):** Define parameter types and context using terse bracket notation `[var_name: Type]`.
* *Example:* ```excel
  /**
   * Returns packed 1D array: [Len_R, Remainder_Limbs, Quotient_Limbs].
   * limbs_a: Array, limbs_b: Array, k: int.
   */
   ```
### Inline Comments: *Why*—Not *What*
Do not narrate syntax or explicitly list sequential steps if the code already speaks for itself. Comments exist solely to explain invisible constraints.

- Explain Excel-specific workarounds (e.g., vertical state packing to bypass REDUCE accumulator limits).
- Highlight mathematical boundary risks, chunking alignments, or orientation flips (e.g., reversing an array to Big-Endian for a top-down iteration).

### Visual Organization
Use standard ASCII dividers to break internal modules into distinct functional regions, creating visual bulkheads for easier navigation within the AFE.

```excel
// ========================================== //
//              Division Kernels              //
// ========================================== //
```

### Canonical Form Rules
- **Input tolerance:** Input strings may contain leading zeros (common in fixed-width data).
- **Strict Normalization:** `BigInt.Norm` must instantly strip all leading zeros to produce the canonical internal state. `"000123"` becomes `"123"`.
- Input strings must consist ONLY of digits `0-9`, with an optional single `-` as the first character.
- Empty, null, and omitted input strings normalize to `"0"`.

### Validation & Error Handling (Split Architecture)
To avoid performance-destroying validation spiderwebs, the library uses a two-tier error system:

**Internal Engine (Fail-Fast):** Functions like `BigInt.Norm` strictly validate input. Any invalid character instantly throws a native Excel `#VALUE!` error. Division by zero throws `#DIV/0!`. Exceeding the string limit throws `#NUM!`. These errors naturally bubble up and short-circuit `REDUCE` loops with zero performance overhead. Silently ignoring dirty data is strictly prohibited.

### Boundary Contracts (Endianness)
To prevent endian-mismatch bugs across the hot loops, the library strictly enforces where data orientation flips:
- **Layer 0 (Text):** All text strings are Big-Endian (left-to-right).
- **Layer 1-3 (Internal Engine):** All internal array mathematics are strictly Little-Endian (bottom-up).
- **The Boundary:** `core_BigInt.Split` is solely responsible for generating Little-Endian arrays from text. `core_BigInt.Merge` is solely responsible for the single `CHOOSEROWS` reversal to translate the final array back to Big-Endian text. Every internal function within this boundary must accept and return Little-Endian numeric arrays.

### Alignment & Padding Strategy
Array dimensions are matched using Excel's native `EXPAND` function. Because internal arrays are Little-Endian, expanding an array downwards with `0`s mathematically equates to adding empty, higher-magnitude limbs. This avoids $O(N)$ string-padding or $O(N^2)$ `VSTACK` loop operations.

## Documentation & Testing Standards

**Docstrings:**
Every public and internal LAMBDA must include a concise, clear docstring. Because these appear in Excel's native formula tooltips, they must be brief but accurately describe the inputs and expected return type (Text vs. Limb Array).

**Testing Protocol:**
*Every* function *must* be validated against a standardized Markdown test table before development proceeds to the next function. All required arguments are listed under `input` as comma separated KVPs. Descriptions have a category and ultra concise explanation of what is being tested.

The standard format is:

| input | expected | output | passing | test |
| --- | --- | --- | --- | --- |
| KVPs | Target result | | | Brief description of edge case |

## Layered Architecture

**Layer 0 — The Public Text Boundary**
The outer shell. Functions here are public-facing, safe, and heavily validated.
- **Contract:** Accepts ONLY text (or native Excel ranges). Returns ONLY text.
- **Role:** Normalizes inputs (`BigInt.Norm`), evaluates signs, and acts as the grand orchestrator. It uses `Split` to convert normalized text into little-endian arrays, passes those arrays down to Layer 1, and uses `Merge` to format the final array back into text.
- **Examples:** `BigInt.Add`, `BigInt.Sub`, `BigInt.Div`, `BigInt.Sum`, `BigInt.Fact`, `BigInt.Norm`.

**Layer 1 — Internal Orchestration & Sign Logic**
The routing layer. Operations here deal with mathematical concepts (like positive/negative interactions) but don't perform the raw arithmetic.
- **Contract:** Accepts ONLY numeric little-endian arrays and boolean/scalar sign flags. Returns ONLY numeric little-endian arrays.
- **Role:** Composes core functions. For example, an internal signed addition function evaluates the magnitudes (via Layer 3 `Compare`) and signs of two arrays, determining whether to route the actual math to `UAdd` or `USub` (Layer 2), and tracks the sign of the final result.
- **Examples:** `core_BigInt.AddSubRouter`, `core_BigInt.DivSearch`.

**Layer 2 — The Unsigned Kernels (The Core)**
The engine room. This is where the heavy, dangerous vector math happens.
- **Contract:** Accepts ONLY unsigned, normalized, little-endian numeric arrays. Assumes all validation and sign logic is already resolved.
- **Role:** Executes the core mathematical operations using Excel's dynamic array functions (`MAP`, `REDUCE`, `EXPAND`).
- **Examples:** `core_BigInt.UAdd`, `core_BigInt.USub` (assumes $A \ge B$), `core_BigInt.UMul` (1D Convolution).

**Layer 3 — Atomic Primitives**
The foundational building blocks used by Layers 0, 1, and 2.
- **Contract:** Single-responsibility utilities. Array operations here operate strictly top-to-bottom on little-endian data.
- **Role:** Memory alignment, carry resolution, array comparison, and crossing the text/array boundary. Crucially, `core_BigInt.Split` acts as a unified firewall, utilizing `MAX()` to crush input arrays into pure scalars to prevent downstream array-poisoning inside `SEQUENCE`.
- **Examples:** `core_BigInt.Carry` (via `SCAN`), `core_BigInt.Compare`, `core_BigInt.Split`, `core_BigInt.Merge`, `core_BigInt.SafeK`.

## Development Workflow Rules

1. **Single-Formula Focus:** Development occurs strictly one function at a time.
2. **Explicit Confirmation:** No progression to the next formula or phase occurs without explicit sign-off on the current function's test table.
3. **Concise Explanations:** Deep, granular explanations of Excel basics are omitted. Summaries are provided only for non-obvious mathematical logic or complex vector manipulations (e.g., array alignment tricks using `EXPAND`, or `SCAN`-based carry propagation).
4. **Pre-Emptive Clarification:** If boundary conditions or input formats for a specific function are ambiguous, development pauses for clarification before drafting the formula.

## Development Roadmap

- [X] **Phase 1 — Core Primitives & Limbs (Layer 3)**
  - [X] Implement `BigInt.Norm` (with strict regex/char validation mapping to `#VALUE!`).
  - [X] Implement `core_BigInt.SafeK`.
  - [X] Implement `core_BigInt.Split` and `core_BigInt.Merge`.
  - [X] Validate limb safety and string construction rigorously.
- [X] **Phase 2 — Comparisons & Logic (Layer 3 & 0)**
  - [X] Implement `BigInt.Sign`, `BigInt.Abs`, `BigInt.IsZero`, `BigInt.IsNeg`.
  - [X] Implement `BigInt.Compare` (comparing limb arrays natively).
- [X] **Phase 3 — Unsigned Kernels (Layer 2)**
  - [X] Implement `core_BigInt.UAdd` (Little-Endian `EXPAND` alignment).
  - [X] Implement `core_BigInt.USub` (with borrow handling).
- [X] **Phase 4 — Public Routers & Sum (Layer 1 & 0)**
  - [X] Implement `BigInt.Add` and `BigInt.Sub` (Sign wrappers).
  - [X] Implement `core_BigInt.UMatrixSum` (Orthogonal Broadcast).
  - [X] Implement `BigInt.Sum` (Sign-grouping, pure scalar mapping, and AddSubRouter reconciliation).
  - [X] Stress test Matrix broadcast engine limits (Thousands of rows).
- [X] **Phase 5 — Multiplication**
  - [X] Implement `core_BigInt.UMul` (Calculate context-safe $k$).
  - [X] Implement `BigInt.Mul` wrapper.
  - [X] Large-value stress tests ($10^{2k}$ boundary checks).
- [X] **Phase 6 — Division & Modulo**
  - [X] Implement `BigInt.Div`.
  - [X] Implement `BigInt.Mod`.
  - [X] `#DIV/0!` edge case hardening.
  - [X] Implement internal Newton-Raphson division and logic to choose when it is used.
- [X] **Phase 7 — Higher-Order & Polish**
  - [X] Implement `BigInt.Pow`.
  - [X] Implement `BigInt.Sqrt`.
  - [X] Implement `BigInt.Fact`.
  - [X] Range helpers (`BigInt.Min`, `BigInt.Max`).
  - [X] Abuse-testing Excel engine (Tens of thousands of digits).
- [X] **Phase 8 — Architectural Polish**
  - [X] Audite and refine variable nomenclature
  - [X] Audit and refine doc-strings and in-line comments.
- [ ] **Phase 9 — Adverse Input & Boundary Hardening (The Forgiving Sanitizer)**
  - [ ] Refactor `BigInt.Norm` to intercept and reject raw numeric types > 15 digits to prevent silent IEEE-754 truncation.
  - [ ] Implement text Scientific Notation parsing (safely expanding positive exponents like `"1.2E5"` into `"120000"`, while rejecting fractional evaluations like `"1.2E-2"` or `"1.234E1"`).
  - [ ] Implement regex sanitization pipeline to safely strip whitespace, commas, and underscores.
  - [ ] Implement accounting format resolution (`(123)` -> `-123`).
  - [ ] Implement safe decimal truncation (allow harmless `.000` tails, instantly reject fractional `.45` tails).
  - [ ] Implement Layer 0 predictive limit checks for computationally heavy functions (`Mul`, `Pow`, `Fact`) to intercept 32k+ character blowouts before engaging the internal engine.
- [ ] **Phase 10 — Layer 0 Dynamic Broadcasting (The Matrix Router)**
  - [ ] Implement `core_BigInt.Broadcast(a, b, core_lambda)` to intercept inputs before the core engine.
  - [ ] Program the router to natively evaluate dimensions and apply `MAP` for arrays of the exact same size.
  - [ ] Program the router to utilize `MAKEARRAY` and `INDEX` to seamlessly generate 2D outer-product grids when fed orthogonal 1D arrays (Row $\times$ Column).
  - [ ] Refactor Layer 0 wrappers (`BigInt.Add`, `BigInt.Sub`, `BigInt.Mul`, `BigInt.Div`, `BigInt.Pow`) to route through `core_BigInt.Broadcast`, ensuring strict element-wise operations.
  - [ ] Establish architecture for future `BigInt.MMult` to isolate true linear algebra from standard element-wise multiplication.
- [ ] **Phase 11 — The 48-Bit Binary Engine & Base-N I/O**
  - [ ] Implement `core_BigInt.ToBase2Array`: Converts base-10 strings to little-endian arrays of 48-bit integers via a Hybrid Radix Router.
  - [ ] Implement `core_BigInt.FromBase2Array`: Merges 48-bit integer arrays back to base-10 strings.
  - [ ] Implement `BigInt.ToHex` and `BigInt.FromHex`.
  - [ ] Implement `BigInt.ToBinary` and `BigInt.FromBinary` (Public text boundary wrappers).
- [ ] **Phase 12 — Native Bitwise Operations**
  - [ ] Implement `BigInt.BitAnd`, `BigInt.BitOr`, and `BigInt.BitXor` (Vectorized natively across 48-bit arrays to prevent boundary shear).
  - [ ] Implement `BigInt.ShiftLeft` and `BigInt.ShiftRight`.
  - [ ] Implement `BigInt.TestBit` and `BigInt.BitLength`.
- [ ] **Phase 13 — Binary-Optimized Algorithms**
  - [ ] Implement `BigInt.GCD` utilizing Stein’s Algorithm (Binary GCD) via the new bitwise primitives, bypassing the division router.
  - [ ] Implement `BigInt.LCM` natively via `(A * B) / GCD(A, B)`.
  - [ ] Implement `BigInt.ModPow` (Modular Exponentiation) utilizing bitwise shifts for the exponent and continuous modulo wrapping.
  - [ ] Implement `BigInt.ModInverse` via the Extended Euclidean algorithm.
- [ ] **Phase 14 — Combinatorics, Probability & Cryptography**
  - [ ] Implement `BigInt.RandBetween(min, max)` ensuring uniform mathematical distribution without modulo bias.
  - [ ] Implement `BigInt.Combinations` and `BigInt.Permutations` mapped through the Prime Swing factorial kernel.
  - [ ] Implement `BigInt.IsPrime` using a deterministic Miller-Rabin test powered by the `ModPow` kernel.
- [ ] **Phase 15 — Serialization & Formatted Output**
  - [ ] Implement `BigInt.Format` to cleanly inject thousands separators or custom delimiters into output strings.
  - [ ] Implement `BigInt.ChunkExport` to automatically slice outputs exceeding 32,767 characters across multiple contiguous cells.

## Implementation
The library is implemented across three modules. First, `BigInt`, the public API. Second, `core_BigInt`, the private API. Third, `test_BigInt`, a private benchmarking suite. Excel modules automatically prepend the module name and a period onto its functions. For example, `UAdd`is part of the private API, which can call it without the prefix. The Excel grid and other modules must use `core_BigInt.UAdd` instead.

```excel

```

```excel

```

## Current Priority
Continue with the roadmap, let's explore layer 0 weaknesses and strategies to harden against them. Do not write any code yet. Let's discuss the design, and once aligned, write the code to achieve it.