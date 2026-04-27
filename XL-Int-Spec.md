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
- [ ] **Phase 8 — Architectural Polish & Internal Diagnostics**
  - [X] Audite and refine variable nomenclature
  - [ ] Audit and refine doc-strings and in-line comments.
  - [ ] Align public and private docstrings to the specification.
  - [ ] Implement `core_BigInt.DumpState` and other debugging tools to aid troubleshooting.
- [ ] **Phase 9 — Targeted Hardening & Boundary Stress Testing**
  - [ ] Develop automated fuzz-testing for the absolute 32,766-character string ceiling.
  - [ ] Validate garbage collection stability during the hybrid `DivRouter` crossover under massive load.
  - [ ] Hard-test the 14-digit `SafeK` boundary against silent IEEE-754 precision bleeding.
- [ ] **Phase 10 — The 48-Bit Binary Engine & Base-N I/O**
  - [ ] Implement `core_BigInt.ToBase2Array`: Converts base-10 strings to little-endian arrays of 48-bit integers via a Hybrid Radix Router.
  - [ ] Implement `core_BigInt.FromBase2Array`: Merges 48-bit integer arrays back to base-10 strings.
  - [ ] Implement `BigInt.ToHex` and `BigInt.FromHex`.
  - [ ] Implement `BigInt.ToBinary` and `BigInt.FromBinary` (Public text boundary wrappers).
- [ ] **Phase 11 — Native Bitwise Operations**
  - [ ] Implement `BigInt.BitAnd`, `BigInt.BitOr`, and `BigInt.BitXor` (Vectorized natively across 48-bit arrays to prevent boundary shear).
  - [ ] Implement `BigInt.ShiftLeft` and `BigInt.ShiftRight`.
  - [ ] Implement `BigInt.TestBit` and `BigInt.BitLength`.
- [ ] **Phase 12 — Binary-Optimized Algorithms**
  - [ ] Implement `BigInt.GCD` utilizing Stein’s Algorithm (Binary GCD) via the new bitwise primitives, bypassing the division router.
  - [ ] Implement `BigInt.LCM` natively via `(A * B) / GCD(A, B)`.
  - [ ] Implement `BigInt.ModPow` (Modular Exponentiation) utilizing bitwise shifts for the exponent and continuous modulo wrapping.
  - [ ] Implement `BigInt.ModInverse` via the Extended Euclidean algorithm.
- [ ] **Phase 13 — Combinatorics, Probability & Cryptography**
  - [ ] Implement `BigInt.RandBetween(min, max)` ensuring uniform mathematical distribution without modulo bias.
  - [ ] Implement `BigInt.Combinations` and `BigInt.Permutations` mapped through the Prime Swing factorial kernel.
  - [ ] Implement `BigInt.IsPrime` using a deterministic Miller-Rabin test powered by the `ModPow` kernel.
- [ ] **Phase 14 — Serialization & Formatted Output**
  - [ ] Implement `BigInt.Format` to cleanly inject thousands separators or custom delimiters into output strings.
  - [ ] Implement `BigInt.ChunkExport` to automatically slice outputs exceeding 32,767 characters across multiple contiguous cells.

## Implementation
The library is implemented across three modules. First, `BigInt`, the public API. Second, `core_BigInt`, the private API. Third, `test_BigInt`, a private benchmarking suite. Excel modules automatically prepend the module name and a period onto its functions. For example, `UAdd`is part of the private API, which can call it without the prefix. The Excel grid and other modules must use `core_BigInt.UAdd` instead.

```excel
////////////////////////////////////
// Big Integer Arithmetic Library //
//        BigInt Module           //
//          Public API            //
////////////////////////////////////

/**
 * Normalises a BigInt string. Strips leading zeros, resolves "-0" to "0", and throws #VALUE! on invalid characters.
 * @param input The BigInt string.
 */
Norm = LAMBDA(input,
    LET(
        str_input, IF(OR(ISOMITTED(input), input = ""), "0", input & ""),
        is_value_format, REGEXTEST(str_input, "^-?\d+$"),
        str_norm, REGEXREPLACE(str_input, "^(-?)0+(?=\d)", "$1"),

        IF(NOT(is_value_format), VALUE("#VALUE!"), IF(str_norm = "-0", "0", str_norm))
    )
);

/**
 * Returns the sign of a BigInt: 1 (positive), -1 (negative), or 0 (zero).
 * @param big_int The BigInt string.
 */
Sign = LAMBDA(input,
    LET(
        str_norm, Norm(input),
        SWITCH(LEFT(str_norm),
            "0", 0,
            "-", -1,
            1
        )
    )
);

// --- Predicates --- \\

/**
 * Returns TRUE if the BigInt is exactly zero.
 * @param input The BigInt string.
 */
IsZero = LAMBDA(input, Norm(input) = "0");

/**
 * Returns TRUE if the BigInt is negative.
 * @param input The BigInt string.
 */
IsNeg = LAMBDA(input, LEFT(Norm(input)) = "-");

/**
 * Returns TRUE if the BigInt is even.
 * @param input The BigInt string.
 */
IsEven = LAMBDA(input, ISEVEN(VALUE(RIGHT(Norm(input)))));

/**
 * Returns TRUE if the BigInt is odd.
 * @param input The BigInt string.
 */
IsOdd = LAMBDA(input, ISODD(VALUE(RIGHT(Norm(input)))));

/**
 * Returns the unsigned magnitude of a BigInt string (absolute value).
 * @param input The BigInt string.
 */
Abs = LAMBDA(input,
    LET(
        str_norm, Norm(input),
        IF(LEFT(str_norm) = "-", MID(str_norm, 2, LEN(str_norm) - 1), str_norm)
    )
);

/**
 * Compares two BigInt strings. Returns 1 (a > b), -1 (a < b), or 0 (a = b).
 * Evaluates signs and lengths. If identical, delegates to native limb array comparison.
 * @param input_a The first BigInt string.
 * @param input_b The second BigInt string.
 */
Compare = LAMBDA(input_a, input_b,
    LET(
        str_norm_a, Norm(input_a),
        str_norm_b, Norm(input_b),
        sign_a, BigInt.Sign(str_norm_a),
        sign_b, Bigint.Sign(str_norm_b),

        IF(sign_a <> sign_b,
            SIGN(sign_a - sign_b),
            IF(sign_a = 0,
                0,
                LET(
                    len_a, LEN(str_norm_a),
                    len_b, LEN(str_norm_b),

                    IF(len_a <> len_b,
                        SIGN(len_a - len_b) * sign_a,
                        LET(
                            k, core_BigInt.SafeK("ADD"),
                            limbs_a, core_BigInt.Split(BigInt.Abs(str_norm_a), k),
                            limbs_b, core_BigInt.Split(BigInt.Abs(str_norm_b), k),
                            magnitude_comparison, core_BigInt.UCompare(limbs_a, limbs_b),

                            magnitude_comparison * sign_a
                        )
                    )
                )
            )
        )
    )
);

//          - Addition & Subtraction -       \\

/**
 * Adds two BigInt strings.
 * @param input_a The first BigInt string.
 * @param input_b The second BigInt string.
 */
Add = LAMBDA(input_a, input_b,
    LET(
        str_norm_a, Norm(input_a),
        str_norm_b, Norm(input_b),
        sign_a, Bigint.Sign(str_norm_a),
        sign_b, BigInt.Sign(str_norm_b),

        k, core_BigInt.SafeK("ADD"),
        limbs_a, core_BigInt.Split(BigInt.Abs(str_norm_a), k),
        limbs_b, core_BigInt.Split(BigInt.Abs(str_norm_b), k),

        packed_result, core_BigInt.AddSubRouter(limbs_a, limbs_b, sign_a, sign_b, k),
        final_sign, CHOOSEROWS(packed_result, -1),
        final_limbs, DROP(packed_result, -1),

        str_merged, core_BigInt.Merge(final_limbs, k),

        IF(final_sign = -1, "-" & str_merged, str_merged)
    )
);

/**
 * Subtracts the second BigInt string from the first (A - B).
 * @param input_a The minuend BigInt string.
 * @param input_b The subtrahend BigInt string.
 */
Sub = LAMBDA(input_a, input_b,
    LET(
        str_norm_b, Norm(input_b),

        // Invert the sign of B at the text level
        str_inverse_b, IF(str_norm_b = "0", "0",
            IF(LEFT(str_norm_b) = "-",
                MID(str_norm_b, 2, LEN(str_norm_b)),
                "-" & str_norm_b
            )
        ),

        // Pass to Add
        Add(input_a, str_inverse_b)
    )
);

/**
 * Sums an array or range of BigInt strings via vectorized group matrices.
 * @param input_range A contiguous range or array of BigInt strings.
 */
Sum = LAMBDA(input_range,
    LET(
        // Setup for little-endian alignment in matrix sum
        flattened_input, TOROW(input_range, 1),

        IF(OR(ISERROR(flattened_input), COLUMNS(flattened_input) = 0), "0",
            LET(
                str_norms, MAP(flattened_input, LAMBDA(x, Norm(x))),
                arr_signs, MAP(LEFT(str_norms), LAMBDA(x, IF(x = "0", 0, if(x = "-", -1, 1)))),
                str_abs, MAP(str_norms, arr_signs, LAMBDA(x, sign, IF(sign = -1, MID(x, 2, LEN(x)-1), x))),

                str_pos, FILTER(str_abs, arr_signs = 1, {"0"}),
                str_negs, FILTER(str_abs, arr_signs = -1, {"0"}),

                unified_k, core_BigInt.SafeK("ADD", COLUMNS(flattened_input)),

                limbs_pos_sum, core_BigInt.UMatrixSum(str_pos, unified_k),
                limbs_neg_sum, core_BigInt.UMatrixSum(str_negs, unified_k),

                packed_result, core_BigInt.AddSubRouter(limbs_pos_sum, limbs_neg_sum, 1, -1, unified_k),
                final_sign, CHOOSEROWS(packed_result, -1),
                final_limbs, DROP(packed_result, -1),

                str_merged, core_BigInt.Merge(final_limbs, unified_k),

                IF(final_sign = -1, "-" & str_merged, str_merged)
            )
        )
    )
);

/**
 * Multiplies two BigInt strings.
 * @param input_a The first BigInt string.
 * @param input_b The second BigInt string.
 */
Mul = LAMBDA(input_a, input_b,
    LET(
        str_norm_a, Norm(input_a),
        str_norm_b, Norm(input_b),
        sign_a, BigInt.Sign(str_norm_a),
        sign_b, BigInt.Sign(str_norm_b),
        len_a, LEN(str_norm_a) - IF(sign_a = -1, 1, 0),
        len_b, LEN(str_norm_b) - IF(sign_b = -1, 1, 0),

        // Product of with 0 is 0
        IF(OR(sign_a = 0, sign_b = 0), "0",
        // The result will exceed char limit
        IF(len_a + len_b > 32766, #NUM!,
            LET(
                k, core_BigInt.SafeK("MUL", MIN(len_a, len_b)),
                limbs_a, core_BigInt.Split(BigInt.Abs(str_norm_a), k),
                limbs_b, core_BigInt.Split(BigInt.Abs(str_norm_b), k),

                final_limbs, core_BigInt.UMul(limbs_a, limbs_b, k),
                final_sign, sign_a * sign_b,

                str_merged, core_BigInt.Merge(final_limbs, k),

                IF(final_sign = -1, "-" & str_merged, str_merged)
            )
        ))
    )
);

/**
 * [VOLATILE] Generates a dynamic array of integers as text.
 * @param [num_rows] array height, default 1.
 * @param [num_cols] array width, default 1.
 * @param [min_digits] minimum number of digits, default 16 (min 1 digit)
 * @param [max_digits] maximum number of digits, default 50 (max 32,767 chars, including sign)
 */
RandArray = LAMBDA([num_rows], [num_cols], [min_digits], [max_digits],
    LET(
        n_rows, IF(ISOMITTED(num_rows), 1, num_rows),
        n_cols, IF(ISOMITTED(num_cols), 1, num_cols),
        min_len, IF(ISOMITTED(min_digits), 16, MIN(32767, MAX(1, min_digits))),
        max_len, IF(ISOMITTED(max_digits), 50, MAX(1, MIN(32766, max_digits))),

        MAP(RANDARRAY(n_rows, n_cols, min_len, max_len, TRUE), LAMBDA(length,
            LET(
                sign_prefix, CHOOSE(RANDBETWEEN(1, 2), "", "-"),
                rand_digits, CONCAT(RANDARRAY(1, length, 0, 9, TRUE)),
                sign_prefix & REGEXREPLACE(rand_digits, "^0*", "")
            )
        ))
    )
);

/**
 * Divides the first BigInt by the second (A / B).
 * Returns the quotient and modulus in rows 1 and 2, respectively.
 * @param input_a The dividend BigInt string.
 * @param input_b The divisor BigInt string.
 * @param [use_floor] Optional boolean. TRUE for Python-style Floor division, FALSE for Truncated. Defaults to FALSE.
 */
DivMod = LAMBDA(input_a, input_b, [use_floor],
    LET(
        str_norm_a, Norm(input_a),
        str_norm_b, Norm(input_b),
        sign_a, BigInt.Sign(str_norm_a),
        sign_b, BigInt.Sign(str_norm_b),
        is_floor_div, IF(ISOMITTED(use_floor), FALSE, use_floor),

        IF(sign_b = 0, #DIV/0!,
        IF(sign_a = 0, {"0";"0"},
            LET(
                k, core_BigInt.SafeK("DIV"),
                limbs_a, core_BigInt.Split(BigInt.Abs(str_norm_a), k),
                limbs_b, core_BigInt.Split(BigInt.Abs(str_norm_b), k),

                packed_result, core_BigInt.DivRouter(limbs_a, limbs_b, k),

                len_remainder, INDEX(packed_result, 1, 1),
                limbs_remainder, CHOOSEROWS(packed_result, SEQUENCE(len_remainder, 1, 2)),
                limbs_quotient, DROP(packed_result, len_remainder + 1),

                final_sign, sign_a * sign_b,
                str_merged_q, core_BigInt.Merge(limbs_quotient, k),
                str_merged_r, core_BigInt.Merge(limbs_remainder, k),

                str_trunc_q, IF(AND(final_sign = -1, str_merged_q <> "0") , "-" & str_merged_q, str_merged_q),
                str_trunc_r, IF(and(sign_a = -1, str_merged_r <> "0"), "-" & str_merged_r, str_merged_r),

                // Floor Correction if floor is requested, signs differ, and remainder is non-zero
                needs_floor_correction, AND(is_floor_div, final_sign = -1, str_merged_r <> "0"),

                str_final_q, IF(needs_floor_correction, Sub(str_trunc_q, "1"), str_trunc_q),
                str_final_r, IF(needs_floor_correction, Add(str_trunc_r, str_norm_b), str_trunc_r),

                VSTACK(str_final_q, str_final_r)
            )
        ))
    )
);

/**
 * Divides the first BigInt by the second (A / B).
 * @param input_a The dividend BigInt string.
 * @param input_b The divisor BigInt string.
 * @param [use_floor] Optional boolean. TRUE for Python-style Floor division. Defaults to FALSE.
 */
Div = LAMBDA(input_a, input_b, [use_floor],
    INDEX(DivMod(input_a, input_b, use_floor), 1, 1)
);

/**
 * Computes the remainder of A / B.
 * @param input_a The dividend BigInt string.
 * @param input_b The divisor BigInt string.
 * @param [use_floor] Optional boolean. TRUE for Python-style Modulo. Defaults to FALSE.
 */
Mod = LAMBDA(input_a, input_b, [use_floor],
    INDEX(DivMod(input_a, input_b, use_floor), 2, 1)
);

/**
 * Raises a BigInt base to the power of a BigInt exponent.
 * @param base The base BigInt string.
 * @param exponent The exponent BigInt string.
 */
Pow = LAMBDA(base, exponent,
    LET(
        str_norm_base, Norm(base),
        str_norm_exp, Norm(exponent),
        sign_base, BigInt.Sign(str_norm_base),
        sign_exp, BigInt.Sign(str_norm_exp),

        str_abs_base, BigInt.Abs(str_norm_base),
        str_abs_exp, BigInt.Abs(str_norm_exp),

        is_exp_even, ISEVEN(VALUE(RIGHT(str_norm_exp, 1))),
        final_sign, IF(sign_base = -1, IF(is_exp_even, 1, -1), sign_base),

        // Handle mathematical boundaries and base limits
        IF(str_abs_base = "0", IF(str_norm_exp = "0", #NUM!, "0"), // 0^0 is undefined
        IF(str_abs_base = "1", IF(final_sign = -1, "-1", "1"),
        IF(sign_exp = -1, "0", // Integer truncation: negative exponents yield 0 for bases >= 2
        IF(str_norm_exp = "0", "1",

        // Results from exponents greater than 6 digits cannot be displayed.
        IF(LEN(str_norm_exp) > 6, #NUM!,
            LET(
                num_exp, VALUE(str_norm_exp),

                // Approximate the final character count: B * Log10(A)
                base_log10, LOG10(VALUE(LEFT(str_abs_base, 14))) + MAX(0, LEN(str_abs_base) - 14),
                approx_digits, num_exp * base_log10,

                IF(approx_digits > 32766, #NUM!,
                    LET(
                        k, core_BigInt.SafeK("MUL", approx_digits / 2),
                        limbs_base, core_BigInt.Split(str_abs_base, k),

                        limbs_final, core_BigInt.UPow(limbs_base, str_norm_exp, k),
                        str_merged, core_BigInt.Merge(limbs_final, k),

                        IF(final_sign = -1, "-" & str_merged, str_merged)
                    )
                )
            )
        )))))
    )
);
//123456789012345678901234567890123456789012345678901234567890123456789-1234567-901234567890123456789012345678901234567-
/**
 * Calculates the integer square root (floor) of a BigInt.
 * @param input The radicand BigInt string.
 */
Sqrt = LAMBDA(input,
    LET(
        str_norm, Norm(input),
        sign_n, BigInt.Sign(str_norm),

        IF(sign_n = -1, #NUM!,
        IF(str_norm = "0", "0",
        IF(str_norm = "1", "1",
            LET(
                len_norm, LEN(str_norm),
                num_seed_precision, IF(len_norm <= 14, len_norm, IF(ISEVEN(len_norm), 14, 13)),
                str_native_precision, LEFT(str_norm, num_seed_precision),
                num_native_precision, VALUE(str_native_precision),
                num_magnitude_zeroes, (len_norm - num_seed_precision) / 2,

                str_prefix, TEXT(ROUNDUP(SQRT(num_native_precision), 0) + 1, "0"),
                str_seed, str_prefix & REPT("0", num_magnitude_zeroes),

                // Scales k by the maximum overlap
                k, core_BigInt.SafeK("MUL", len_norm),

                limbs_radicand, core_BigInt.Split(str_norm, k),
                limbs_seed, core_BigInt.Split(str_seed, k),

                limbs_final, core_BigInt.USqrt(limbs_radicand, limbs_seed, k),

                core_BigInt.Merge(limbs_final, k)
            )
        )))
    )
);

/**
 * Computes the factorial of a BigInt (n!) using Peter Luschny's Prime Swing algorithm.
 * @param input The BigInt string. Maximum supported input is 9273.
 */
Fact = LAMBDA(input,
    LET(
        str_norm, Norm(input),
        sign_n, BigInt.Sign(str_norm),

        IF(sign_n = -1, #NUM!,
        IF(str_norm = "0", "1",
        IF(str_norm = "1", "1",
        // Firewall 1: Trap massive strings before they hit VALUE()
        IF(LEN(str_norm) > 4, #NUM!,
        // Firewall 2: Trap mathematical ceiling
        IF(VALUE(str_norm) > 9273, #NUM!,
            LET(
                num_n, VALUE(str_norm),
                global_primes, core_BigInt.NativePrimes(num_n),

                core_BigInt.PrimeSwingKernel(num_n, global_primes)
            )
        )))))
    )
);

/**
 * Returns the minimum BigInt from an array or range.
 * @param range A contiguous range or array of BigInt strings.
 */
Min = LAMBDA(range,
    LET(
        flattened_col, TOCOL(range, 1),

        IF(OR(ISERROR(flattened_col), ROWS(flattened_col) = 0), "0",
            core_BigInt.TreeCompare(MAP(flattened_col, BigInt.Norm), -1)
        )
    )
);

/**
 * Returns the maximum BigInt from an array or range.
 * @param range A contiguous range or array of BigInt strings.
 */
Max = LAMBDA(range,
    LET(
        flattened_col, TOCOL(range, 1),

        IF(OR(ISERROR(flattened_col), ROWS(flattened_col) = 0), "0",
            core_BigInt.TreeCompare(MAP(flattened_col, BigInt.Norm), 1)
        )
    )
);
```

```excel
////////////////////////////////////
// Big Integer Arithmetic Library //
//      core_BigInt Module        //
//          Private API           //
////////////////////////////////////

/**
 * [Internal] Computes the maximum safe base-10 limb size (k) to avoid IEEE-754 precision loss.
 * @param operation: "ADD", "MUL", or "DIV".
 * @param [max_items] ADD: number of operands. MUL: shortest string digits count.
 */
SafeK = LAMBDA(operation, [operand_or_digits],
    LET(
        resolved_count, IF(OR(operand_or_digits = "", ISOMITTED(operand_or_digits)), 2, MAX(2, operand_or_digits)),

        SWITCH(operation,
            "ADD", 15 - LEN(resolved_count),

            "MUL", LET(
                test_k, {7; 6; 5; 4; 3; 2; 1},

                max_overlap, ROUNDUP(resolved_count / test_k, 0),

                // Calculate the absolute peak value inside the engine (Convolution + Carry)
                uncarried_max, max_overlap * (10^test_k - 1)^2,
                max_carry, INT(uncarried_max / 10^test_k),
                peak_val, uncarried_max + max_carry,

                // Return the max k <= Excel's 15-digit ceiling
                MAX(FILTER(test_k, peak_val <= (10^15 - 1)))
            ),

            "DIV", 7,
            #NAME?
        )
    )
);

/**
 * [Internal] Unified Splitter. Splits both scalars and arrays into Little-Endian limbs.
 * Uses MAX() to guarantee scalar inputs to the SEQUENCE engine.
 * @param big_ints The string or array of strings.
 * @param k The dynamic chunk size context.
 */
Split = LAMBDA(str_norms, k,
    LET(
        flattened_norms, TOROW(str_norms),
        lengths, LEN(flattened_norms),
        max_len, MAX(lengths),
        pad_len, ROUNDUP(max_len / k, 0) * k,

        padded_norms, REPT("0", pad_len - lengths) & flattened_norms,
        limb_start_indices, SEQUENCE(pad_len / k, 1, pad_len - k + 1, -k),

        --MID(padded_norms, limb_start_indices, k)
    )
);

/**
 * [Internal] Concatenates an array of numeric limbs into a BigInt string, zero-padding all but the first limb.
 * @param limbs The vertical array of numeric limbs.
 * @param k The limb size used during calculation.
 */
Merge = LAMBDA(limbs, k,
    LET(
        num_limbs, ROWS(limbs),
        rev_limbs, CHOOSEROWS(limbs, SEQUENCE(num_limbs, 1, num_limbs, -1)),

        IF(num_limbs = 1,
            rev_limbs & "",
            CONCAT(
                VSTACK(
                    TAKE(rev_limbs, 1) & "",
                    TEXT(DROP(rev_limbs, 1), REPT("0", k))
                )
            )
        )
    )
);

/**
 * [Internal] Resolves carries right-to-left across a vertical array of base-10^k limbs.
 * @param limbs The array of numeric limbs.
 * @param k The limb size.
 */
Carry = LAMBDA(limbs, k,
    LET(
        num_limbs, ROWS(limbs),
        radix_base, 10^k,
        arr_carries, SCAN(0, limbs, LAMBDA(acc, val, val + INT(acc / radix_base))),
        remainders, MOD(arr_carries, radix_base),
        final_carry, INT(INDEX(arr_carries, num_limbs, 1) / radix_base),

        IF(final_carry > 0, VSTACK(remainders, final_carry), remainders)
    )
);

// --- Unsigned Kernels --- \\

/**
 * [Internal] Unsigned comparison of two Little-Endian limb arrays.
 * Returns 1 (a > b), -1 (a < b), or 0 (a = b).
 * @param limbs_a The first limb array.
 * @param limbs_b The second limb array.
 */
UCompare = LAMBDA(limbs_a, limbs_b,
    LET(
        max_rows, MAX(ROWS(limbs_a), ROWS(limbs_b)),
        limbs_a_aligned, EXPAND(limbs_a, max_rows, 1, 0),
        limbs_b_aligned, EXPAND(limbs_b, max_rows, 1, 0),

        differences, limbs_a_aligned - limbs_b_aligned,
        highest_non_zero_index, XMATCH(TRUE, differences <> 0, 0, -1),

        IF(ISNA(highest_non_zero_index), 0, SIGN(INDEX(differences, highest_non_zero_index, 1)))
    )
);

/**
 * [Internal] Unsigned addition of two Little-Endian limb arrays.
 * @param limbs_a The first limb array.
 * @param limbs_b The second limb array.
 * @param k The dynamic chunk size context.
 */
UAdd = LAMBDA(a, b, k,
    LET(
        max_rows, MAX(ROWS(a), ROWS(b)),
        limbs_a_aligned, EXPAND(a, max_rows, 1, 0),
        limbs_b_aligned, EXPAND(b, max_rows, 1, 0),

        Carry(limbs_a_aligned + limbs_b_aligned, k)
    )
);

/**
 * [Internal] Unsigned subtraction of two Little-Endian limb arrays.
 * Assumes magnitudes A >= B. Resolves borrows top-to-bottom and trims trailing zeros.
 * @param limbs_a The minuend limb array (must be >= limbs_b).
 * @param limbs_b The subtrahend limb array.
 * @param k The dynamic chunk size context.
 */
USub = LAMBDA(limbs_a, limbs_b, k,
    LET(
        max_rows, MAX(ROWS(limbs_a), ROWS(limbs_b)),
        limbs_a_aligned, EXPAND(limbs_a, max_rows, 1, 0),
        limbs_b_aligned, EXPAND(limbs_b, max_rows, 1, 0),

        differences, limbs_a_aligned - limbs_b_aligned,
        carried_differences, Carry(differences, k),

        highest_non_zero_index, XMATCH(TRUE, carried_differences <> 0, 0, -1),

        IF(ISNA(highest_non_zero_index), {0}, TAKE(carried_differences, highest_non_zero_index))
    )
);

/**
 * [Internal] Vectorized matrix sum of unsigned BigInt strings.
 * @param str_norms Horizontal array (1xN) of normalized magnitude strings.
 * @param k The unified dynamic chunk size context.
 */
UMatrixSum = LAMBDA(str_norms, k,
    LET(
        grid, Split(str_norms, k),
        raw_limbs, BYROW(grid, SUM),

        Carry(raw_limbs, k)
    )
);

/**
 * [Internal] Routes addition/subtraction based on signs and magnitudes.
 * Returns a tuple array: VSTACK(limbs, final_sign) where the final row is the scalar sign.
 * @param limbs_a The first little-endian limb array.
 * @param limbs_b The second little-endian limb array.
 * @param sign_a The sign of the first array (1, -1, 0).
 * @param sign_b The sign of the second array (1, -1, 0).
 * @param k The dynamic chunk size context.
 */
AddSubRouter = LAMBDA(limbs_a, limbs_b, sign_a, sign_b, k,
    LET(
        is_addition, sign_a = sign_b,

        IF(is_addition,
            VSTACK(UAdd(limbs_a, limbs_b, k), sign_a),

            LET(
                magnitude_comparison, UCompare(limbs_a, limbs_b),

                IF(magnitude_comparison = 0, // Total cancellation:
                    {0;0},

                    IF(magnitude_comparison > 0,
                        // A is larger: sign belongs to A.
                        VSTACK(USub(limbs_a, limbs_b, k), sign_a),

                        // B is larger: route B first, sign belongs to B.
                        VSTACK(USub(limbs_b, limbs_a, k), sign_b)
                    )
                )
            )
        )
    )
);

/**
 * [Internal] Unsigned multiplication of two Little-Endian limb arrays.
 * Utilizes a 1D Discrete Convolution (Cauchy Product) array-slicing strategy.
 * Zero dynamic memory reallocation. $O(N+M)$ memory footprint.
 * @param limbs_a The multiplicand limb array.
 * @param limbs_b The multiplier limb array.
 * @param k The dynamic chunk size context.
 */
UMul = LAMBDA(limbs_a, limbs_b, k,
    LET(
        num_limbs_a, ROWS(limbs_a),
        num_limbs_b, ROWS(limbs_b),
        convolution_length, num_limbs_a + num_limbs_b - 1,

        raw_convolution, MAKEARRAY(convolution_length, 1, LAMBDA(r, c,
            LET(
                window_start_i, MAX(1, r - num_limbs_b + 1),
                window_end_i, MIN(r, num_limbs_a),

                overlap_rows, window_end_i - window_start_i + 1,

                seq_i, SEQUENCE(overlap_rows, 1, window_start_i, 1),
                seq_j, SEQUENCE(overlap_rows, 1, r - window_start_i + 1, -1),

                SUM(INDEX(limbs_a, seq_i, 1) * INDEX(limbs_b, seq_j, 1))
            )
        )),

        Carry(raw_convolution, k)
    )
);

/**
 * [Internal] Binary search to find the highest scalar quotient limb q (0 <= q < 10^k)
 * such that B * q <= limbs_r. Eliminates the need for Knuth D float-estimation.
 * @param limbs_r The current remainder array.
 * @param b The divisor array.
 * @param k The dynamic chunk size context.
 */
FindQuotientLimb = LAMBDA(limbs_r, limbs_b, k,
    LET(
        required_bits, CEILING.MATH(LOG(10^k, 2)),
        powers_of_two, 2 ^ SEQUENCE(required_bits, 1, required_bits - 1, -1),

        REDUCE(0, powers_of_two, LAMBDA(current_q, current_power,
            LET(
                test_q, current_q + current_power,

                IF(test_q >= 10^k,
                    current_q,
                    LET(
                        limbs_bq, Carry(limbs_b * test_q, k),
                        magnitude_comparison, UCompare(limbs_r, limbs_bq),

                        IF(magnitude_comparison >= 0, test_q, current_q)
                    )
                )
            )
        ))
    )
);

/**
 * [Internal] Basecase Division via sequential shift-and-subtract.
 * Returns a packed 1D array: VSTACK(Length_R, R_limbs, Q_limbs).
 * Eliminates 2D array padding to strictly preserve memory efficiency.
 * @param limbs_a The dividend array.
 * @param limbs_b The divisor array (assumed non-zero).
 * @param k The dynamic chunk size context.
 */
KnuthDiv = LAMBDA(limbs_a, limbs_b, k,
    LET(
        magnitude_comparison, UCompare(limbs_a, limbs_b),

        IF(magnitude_comparison < 0, VSTACK(ROWS(limbs_a), limbs_a, 0),
        IF(magnitude_comparison = 0, VSTACK(1, 0, 1),
            LET(
                num_limbs_a, ROWS(limbs_a),
                big_endian_a, CHOOSEROWS(limbs_a, SEQUENCE(num_limbs_a, 1, num_limbs_a, -1)),

                final_packed_state, REDUCE(VSTACK(1, 0, 0), big_endian_a, LAMBDA(packed_state, val,
                    LET(
                        num_limbs_r, INDEX(packed_state, 1, 1),
                        limbs_r, CHOOSEROWS(packed_state, SEQUENCE(num_limbs_r, 1, 2)),
                        limbs_q, DROP(packed_state, num_limbs_r + 1),

                        limbs_shifted_r, IF(AND(ROWS(limbs_r) = 1, INDEX(limbs_r, 1, 1) = 0),
                            val,
                            VSTACK(val, limbs_r)
                        ),

                        q_limb, FindQuotientLimb(limbs_shifted_r, limbs_b, k),
                        limbs_bq, Carry(limbs_b * q_limb, k),
                        limbs_next_r, USub(limbs_shifted_r, limbs_bq, k),
                        has_q, OR(ROWS(limbs_q) > 1, INDEX(limbs_q,1,1) > 0),

                        limbs_next_q, IF(has_q, VSTACK(limbs_q, q_limb), q_limb),

                        VSTACK(ROWS(limbs_next_r), limbs_next_r, limbs_next_q)
                    )
                )),

                num_limbs_r, INDEX(final_packed_state, 1, 1),
                limbs_r, CHOOSEROWS(final_packed_state, SEQUENCE(num_limbs_r, 1, 2)),
                big_endian_q, DROP(final_packed_state, num_limbs_r + 1),

                len_q, ROWS(big_endian_q),
                limbs_q, CHOOSEROWS(big_endian_q, SEQUENCE(len_q, 1, len_q, -1)),

                VSTACK(num_limbs_r, limbs_r, limbs_q)
            )
        ))
    )
);

/**
 * [Internal] Hybrid Division Router.
 * Directs division to the optimal kernel based on Dividend length.
 * @param limbs_a The dividend array.
 * @param limbs_b The divisor array.
 * @param k The dynamic chunk size context.
 */
DivRouter = LAMBDA(limbs_a, limbs_b, k,
    LET(
        approx_digits_a, ROWS(limbs_a) * k,
        approx_digits_b, ROWS(limbs_b) * k,

        // Logistic Regression coefficients
        regression_slope_m, 0.107821194271297,
        regression_intercept_c, -221.902189462068,
        threshold_digits, (regression_slope_m * approx_digits_a) + regression_intercept_c,

        IF(approx_digits_b <= threshold_digits,
            KnuthDiv(limbs_a, limbs_b, k),
            NewtonRaphsonDiv(limbs_a, limbs_b, k)
        )
    )
);

/**
 * [Internal] Generates the exact reciprocal seed for Newton-Raphson Division using Knuth Basecase.
 * Extracts up to the top 3 limbs of B and calculates floor(Beta^(2*len) / B_top).
 * @param limbs_b The divisor limb array.
 * @param k The dynamic chunk size context.
 */
NewtonRaphsonSeed = LAMBDA(limbs_b, k,
    LET(
        num_limbs_b, ROWS(limbs_b),
        num_seed_limbs, MIN(3, num_limbs_b),

        // Extract top 'seed_len' limbs (Little-Endian: take from the bottom)
        limbs_b_top, TAKE(limbs_b, -num_seed_limbs),

        // Construct Beta^(2 * seed_len) natively
        // e.g., if seed_len = 1, then 1 followed by two zero limbs: VSTACK(0, 0, 1)
        limbs_beta_2m, VSTACK(EXPAND(0, 2 * num_seed_limbs, , 0), 1),

        // Knuth Division yields exact initial reciprocal
        packed_basecase_result, KnuthDiv(limbs_beta_2m, limbs_b_top, k),

        // Extract Quotient array from the packed VSTACK
        num_limbs_r, INDEX(packed_basecase_result, 1, 1),
        DROP(packed_basecase_result, num_limbs_r + 1)
    )
);

/**
 * [Internal] Computes the Reciprocal progressively scaling up to the Dividend's precision.
 * @param limbs_b The divisor limb array.
 * @param target_len The limb-length of Dividend A.
 * @param k The dynamic chunk size context.
 */
NewtonRaphsonReciprocal = LAMBDA(limbs_b, target_len, k,
    LET(
        num_limbs_b, ROWS(limbs_b),
        num_seed_limbs, MIN(3, num_limbs_b),
        limbs_r_seed, core_BigInt.NewtonRaphsonSeed(limbs_b, k),

        required_iterations, MAX(0, CEILING.MATH(LN(target_len / num_seed_limbs) / LN(2))) + 1,

        IF(required_iterations = 0, limbs_r_seed,
            LET(
                iterations, SEQUENCE(required_iterations),
                final_packed_state, REDUCE(VSTACK(num_seed_limbs, limbs_r_seed), iterations, LAMBDA(packed_state, i,
                    LET(
                        current_len, INDEX(packed_state, 1, 1),
                        limbs_current_r, DROP(packed_state, 1),

                        next_len, MIN(current_len * 2, target_len),

                        // Dynamic Scaling Shift:
                        // If B has plateaued, R must scale by 2*(n-m). If B is growing, R scales by (n-m).
                        active_b_limbs_current, MIN(current_len, num_limbs_b),
                        active_b_limbs_next, MIN(next_len, num_limbs_b),
                        required_shift_limbs, 2 * (next_len - current_len) - (active_b_limbs_next - active_b_limbs_current),

                        limbs_r_prime, IF(required_shift_limbs > 0,
                            VSTACK(EXPAND(0, required_shift_limbs, , 0), limbs_current_r),
                            limbs_current_r
                        ),

                        limbs_b_active, TAKE(limbs_b, -active_b_limbs_next),

                        // P is natively aligned to 2*next_len because of the corrected shift_limbs
                        limbs_p, core_BigInt.UMul(limbs_b_active, limbs_r_prime, k),

                        limbs_two_beta, VSTACK(EXPAND(0, 2 * next_len, , 0), 2),
                        limbs_error_term, core_BigInt.USub(limbs_two_beta, limbs_p, k),

                        limbs_next_r_unscaled, core_BigInt.UMul(limbs_r_prime, limbs_error_term, k),
                        limbs_next_r, IF(ROWS(limbs_next_r_unscaled) <= 2 * next_len,
                            {0},
                            DROP(limbs_next_r_unscaled, 2 * next_len)
                        ),

                        VSTACK(next_len, limbs_next_r)
                    )
                )),
                DROP(final_packed_state, 1)
            )
        )
    )
);

/**
 * [Internal] Heavyweight Newton-Raphson Division.
 * Implements progressive scaling bounded to MAX(len_a, len_b).
 * @param limbs_a The dividend array.
 * @param limbs_b The divisor array.
 * @param k The dynamic chunk size context.
 */
NewtonRaphsonDiv = LAMBDA(limbs_a, limbs_b, k,
    LET(
        num_limbs_a, ROWS(limbs_a),
        num_limbs_b, ROWS(limbs_b),
        num_limbs_target, MAX(num_limbs_a, num_limbs_b),

        // 1. Calculate reciprocal R scaled dynamically to 2 * target_len
        limbs_reciprocal, core_BigInt.NewtonRaphsonReciprocal(limbs_b, num_limbs_target, k),

        // 2. Approximate Quotient: Q_approx = A * R / Beta^(2 * target_len)
        right_shift_limbs, 2 * num_limbs_target,
        limbs_q_unscaled, core_BigInt.UMul(limbs_a, limbs_reciprocal, k),
        limbs_q_approx, IF(ROWS(limbs_q_unscaled) <= right_shift_limbs, {0}, DROP(limbs_q_unscaled, right_shift_limbs)),

        // Calculate the massive baseline product exactly ONCE
        limbs_p_baseline, core_BigInt.UMul(limbs_b, limbs_q_approx, k),

        // 3. Bounded Error Correction Sweep (Optimized O(N) Stepping)
        // State packing: VSTACK(Length_Q, Q_limbs, P_limbs)
        initial_packed_state, VSTACK(ROWS(limbs_q_approx), limbs_q_approx, limbs_p_baseline),

        // Sweep Down: If P > A, Q is too big. Step Q down by 1, and P down by B.
        packed_state_swept_down, REDUCE(initial_packed_state, {1; 2}, LAMBDA(packed_state, i,
            LET(
                len_q, INDEX(packed_state, 1, 1),
                limbs_current_q, CHOOSEROWS(packed_state, SEQUENCE(len_q, 1, 2)),
                limbs_current_p, DROP(packed_state, len_q + 1),

                IF(core_BigInt.UCompare(limbs_current_p, limbs_a) <= 0, packed_state,
                    LET(
                        limbs_next_q, core_BigInt.USub(limbs_current_q, {1}, k),
                        limbs_next_p, core_BigInt.USub(limbs_current_p, limbs_b, k),
                        VSTACK(ROWS(limbs_next_q), limbs_next_q, limbs_next_p)
                    )
                )
            )
        )),

        // Sweep Up: If P + B <= A, Q is too small. Step Q up by 1, and P up by B.
        packed_state_swept_up, REDUCE(packed_state_swept_down, {1; 2}, LAMBDA(packed_state, i,
            LET(
                len_q, INDEX(packed_state, 1, 1),
                limbs_current_q, CHOOSEROWS(packed_state, SEQUENCE(len_q, 1, 2)),
                limbs_current_p, DROP(packed_state, len_q + 1),

                limbs_next_p_test, core_BigInt.UAdd(limbs_current_p, limbs_b, k),

                IF(core_BigInt.UCompare(limbs_next_p_test, limbs_a) > 0, packed_state,
                    LET(
                        limbs_next_q, core_BigInt.UAdd(limbs_current_q, {1}, k),
                        VSTACK(ROWS(limbs_next_q), limbs_next_q, limbs_next_p_test)
                    )
                )
            )
        )),

        // Extract Final Q and P
        len_q, INDEX(packed_state_swept_up, 1, 1),
        limbs_q, CHOOSEROWS(packed_state_swept_up, SEQUENCE(len_q, 1, 2)),
        limbs_p, DROP(packed_state_swept_up, len_q + 1),

        // 4. Extract True Remainder Safely (A - Final_P)
        limbs_r, core_BigInt.USub(limbs_a, limbs_p, k),

        VSTACK(
            ROWS(limbs_r),
            limbs_r,
            limbs_q
        )
    )
);

/**
 * [Internal] Unsigned Base-10 Left-to-Right Exponentiation.
 * Evaluates the exponent iteratively using a pre-computed 1-9 cache.
 * @param limbs_base The little-endian limb array of the base.
 * @param exp_string The exact text string of the exponent.
 * @param k The dynamic chunk size context.
 */
UPow = LAMBDA(limbs_base, exponent_string, k,
    LET(
        // Pre-compute powers 1 through 9.
        limbs_base1, limbs_base,
        limbs_base2, core_BigInt.UMul(limbs_base1, limbs_base1, k),
        limbs_base3, core_BigInt.UMul(limbs_base2, limbs_base1, k),
        limbs_base4, core_BigInt.UMul(limbs_base2, limbs_base2, k),
        limbs_base5, core_BigInt.UMul(limbs_base4, limbs_base1, k),
        limbs_base6, core_BigInt.UMul(limbs_base3, limbs_base3, k),
        limbs_base7, core_BigInt.UMul(limbs_base6, limbs_base1, k),
        limbs_base8, core_BigInt.UMul(limbs_base4, limbs_base4, k),
        limbs_base9, core_BigInt.UMul(limbs_base8, limbs_base1, k),

        exponent_digits, --MID(exponent_string, SEQUENCE(LEN(exponent_string)), 1),

        REDUCE(1, exponent_digits, LAMBDA(limbs_accumulator, current_digit,
            LET(
                // Short-circuit: Skip calculating 1^10 during leading iterations
                is_one, AND(ROWS(limbs_accumulator) = 1, INDEX(limbs_accumulator, 1, 1) = 1),

                // 1. Raise Acc to the 10th power
                limbs_acc_10, IF(is_one, limbs_accumulator,
                    LET(
                        limbs_acc_2, core_BigInt.UMul(limbs_accumulator, limbs_accumulator, k),
                        limbs_acc_4, core_BigInt.UMul(limbs_acc_2, limbs_acc_2, k),
                        limbs_acc_8, core_BigInt.UMul(limbs_acc_4, limbs_acc_4, k),
                        core_BigInt.UMul(limbs_acc_8, limbs_acc_2, k)
                    )
                ),

                // 2. Multiply by the corresponding cached value
                IF(current_digit = 0, limbs_acc_10,
                    LET(
                        limbs_cache_multiplier, CHOOSE(current_digit, limbs_base1, limbs_base2, limbs_base3, limbs_base4, limbs_base5, limbs_base6, limbs_base7, limbs_base8, limbs_base9),
                        IF(is_one, limbs_cache_multiplier, core_BigInt.UMul(limbs_acc_10, limbs_cache_multiplier, k))
                    )
                )
            )
        ))
    )
);

/**
 * [Internal] Unsigned Integer Square Root using Newton's Method.
 * @param limbs_radicand.
 * @param limbs_initial_x Over-estimated seed array.
 * @param k The dynamic chunk size context.
 */
USqrt = LAMBDA(limbs_radicand, limbs_initial_x, k,
    LET(
        maximum_iterations, 35,

        final_packed_state, REDUCE(VSTACK(0, limbs_initial_x), SEQUENCE(maximum_iterations), LAMBDA(packed_state, i,
            LET(
                has_converged, INDEX(packed_state, 1, 1),
                limbs_current_x, DROP(packed_state, 1),

                IF(has_converged, packed_state,
                    LET(
                        // 1. Division: n / x
                        packed_division_result, core_BigInt.DivRouter(limbs_radicand, limbs_current_x, k),
                        num_limbs_r, INDEX(packed_division_result, 1, 1),
                        limbs_q, DROP(packed_division_result, num_limbs_r + 1),

                        // 2. Addition: x + (n / x)
                        limbs_sum_xq, core_BigInt.UAdd(limbs_current_x, limbs_q, k),

                        // 3. Halving: (x + (n / x)) / 2
                        // KnuthDiv inherently supports scalar divisors with zero overhead
                        packed_half_result, core_BigInt.KnuthDiv(limbs_sum_xq, {2}, k),
                        num_limbs_halfed, INDEX(packed_half_result, 1, 1),
                        limbs_next_x, DROP(packed_half_result, num_limbs_halfed + 1),

                        // 4. Convergence Check
                        convergence_comparison, core_BigInt.UCompare(limbs_next_x, limbs_current_x),

                        // Newton's method converges on the floor root, and stabilizes or oscillates up by 1.
                        // Therefore, if x_next >= curr_x, the algorithm has converged.
                        IF(convergence_comparison >= 0,
                            VSTACK(1, limbs_current_x),
                            VSTACK(0, limbs_next_x)
                        )
                    )
                )
            )
        )),

        DROP(final_packed_state, 1)
    )
);

/**
 * [Internal] Generates a 1D vertical array of all prime numbers up to n.
 * Evaluates purely in native C++ using a dynamic sieve.
 * @param n The upper bound integer (assumed <= 9273).
 */
NativePrimes = LAMBDA(n,
    IF(n < 2, #NUM!,
        LET(
            candidate_sequence, SEQUENCE(n - 1, 1, 2),
            FILTER(candidate_sequence, MAP(candidate_sequence, LAMBDA(current_candidate,
                LET(
                    limit, INT(SQRT(current_candidate)),
                    IF(limit < 2, TRUE,
                        MIN(MOD(current_candidate, SEQUENCE(limit - 1, 1, 2))) > 0
                    )
                )
            )))
        )
    )
);

/**
 * [Internal] Unsafe, high-performance text multiplication for normalised, positive BigInt strings.
 * Bypasses Layer 0 validation. Implements strict short-circuits for padding optimization.
 * @param norm_a
 * @param norm_b
 */
UTextMul = LAMBDA(norm_a, norm_b,
    IF(OR(norm_a = "0", norm_b = "0"), "0",
    IF(norm_a = "1", norm_b,
    IF(norm_b = "1", norm_a,
        LET(
            len_a, LEN(norm_a),
            len_b, LEN(norm_b),
            k, core_BigInt.SafeK("MUL", MIN(len_a, len_b)),

            limbs_a, core_BigInt.Split(norm_a, k),
            limbs_b, core_BigInt.Split(norm_b, k),

            final_limbs, core_BigInt.UMul(limbs_a, limbs_b, k),

            core_BigInt.Merge(final_limbs, k)
        )
    )))
);

/**
 * [Internal] Vectorized logarithmic product of an array of BigInt strings.
 * Recursively cuts the array in half to bypass REDUCE sequential memory allocations.
 * @param arr_strings A 1D vertical array of BigInt strings.
 */
TextTreeProd = LAMBDA(arr_strings,
    LET(
        num_elements, ROWS(arr_strings),

        IF(num_elements = 1, INDEX(arr_strings, 1, 1),
            LET(
                // Reshape array into pairs. Odd-length arrays are safely padded with the multiplicative identity "1"
                paired_strings, WRAPROWS(arr_strings, 2, "1"),

                // Vectorized multiplication across all pairs simultaneously
                reduced_array, BYROW(paired_strings, LAMBDA(row, UTextMul(INDEX(row, 1, 1), INDEX(row, 1, 2)))),

                // Shallow recursion
                core_BigInt.TextTreeProd(reduced_array)
            )
        )
    )
);

/**
 * [Internal] Calculates the exponent for an array of primes in the Swing term of a factorial.
 * @param n The current factorial magnitude (<= 9273).
 * @param primes A vertical array of prime numbers (<= n).
 */
NativeSwingExponents = LAMBDA(n, primes,
    MAP(primes, LAMBDA(p,
        LET(
            max_power_k, INT(LOG(n, p)),
            prime_powers, p ^ SEQUENCE(max_power_k),
            SUM(MOD(INT(n / prime_powers), 2))
        )
    ))
);

/**
 * [Internal] Calculates the massive BigInt string for the Swing term of a factorial.
 * @param n The current factorial magnitude.
 * @param tier_primes A vertical array of prime numbers (<= n).
 */
SwingTerm = LAMBDA(n, tier_primes,
    LET(
        swing_exponents, core_BigInt.NativeSwingExponents(n, tier_primes),
        has_native_exponent, swing_exponents > 0,

        // If all exponents are 0, the swing term is 1.
        IF(AND(NOT(has_native_exponent)), "1",
            LET(
                active_primes, FILTER(tier_primes, has_native_exponent),
                active_exponents, FILTER(swing_exponents, has_native_exponent),

                evalutated_prime_powers, MAP(active_primes, active_exponents, LAMBDA(p, e,
                    IF(e = 1, p & "", BigInt.Pow(p & "", e & ""))
                )),

                core_BigInt.TextTreeProd(evalutated_prime_powers)
            )
        )
    )
);

/**
 * [Internal] Recursive top-down LAMBDA calculating the Prime Swing factorial.
 * @param n The current factorial magnitude.
 * @param global_primes The unified vertical array of all primes up to the target factorial.
 */
PrimeSwingKernel = LAMBDA(n, global_primes,
    IF(n < 2, "1",
        LET(
            floor_half_n, INT(n / 2),
            factorial_half_n, core_BigInt.PrimeSwingKernel(floor_half_n, global_primes),

            // Square the halfway point using the unsafe kernel
            squared_half_factorial, core_BigInt.UTextMul(factorial_half_n, factorial_half_n),

            // Isolate the primes needed for this tier and calculate the swing term
            applicable_tier_primes, FILTER(global_primes, global_primes <= n),
            str_swing_term, core_BigInt.SwingTerm(n, applicable_tier_primes),

            core_BigInt.UTextMul(squared_half_factorial, str_swing_term)
        )
    )
);

/**
 * [Internal] Recursive tournament tree for Min/Max evaluations.
 * @param col A 1D vertical array of normalized BigInt strings.
 * @param mode 1 for Maximum, -1 for Minimum.
 */
TreeCompare = LAMBDA(str_norms, comparison_mode,
    LET(
        num_rows, ROWS(str_norms),

        IF(num_rows = 1, INDEX(str_norms, 1, 1),
            LET(
                tournament_pairs, WRAPROWS(str_norms, 2),
                competitors_a, TAKE(tournament_pairs, , 1),
                competitors_b, TAKE(tournament_pairs, , -1),

                round_winners, MAP(competitors_a, competitors_b, LAMBDA(a, b,
                    IF(ISNA(b), a,
                        LET(
                            comparison_result, BigInt.Compare(a, b),
                            IF(comparison_result * comparison_mode >= 0, a, b)
                        )
                    )
                )),

                core_BigInt.TreeCompare(round_winners, comparison_mode)
            )
        )
    )
);
```

## Current Priority
Continue with the roadmap and:
1. Review the documentation.
2. Enforce a strict 2-line limit on doc strings to support tooltips.
3. Ensure in-line comments are provided where necessary.
4. Ensure in-line comments document "why" the code is there, not "what" it does.