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
3. **C++ Orthogonal Broadcast:** The grouped strings are padded and array-intersected horizontally and vertically simultaneously using `--MID(...)`. This generates a massive 2D C++ grid in a single calculation step.
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

### Naming & Namespaces
- **Public API:** Uses the `BigInt.` prefix and PascalCase (e.g., `BigInt.Sum`, `BigInt.Compare`) to cleanly differentiate from native uppercase Excel functions and prevent shadowing.
- **Internal Kernels:** Private helper functions are prefixed with `internal.` (e.g., `core_BigInt.UAdd`, `core_BigInt.SafeK`).

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
*Every* function *must* be validated against a standardized Markdown test table before development proceeds to the next function. The standard format is:

| input | expected | output | passing | test |
| --- | --- | --- | --- | --- |
| Values | Target result | | | Brief description of edge case |

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
- **Role:** Memory alignment, carry resolution, array comparison, and crossing the text/array boundary. Crucially, `core_BigInt.Split` acts as a unified C++ firewall, utilizing `MAX()` to crush input arrays into pure scalars to prevent downstream array-poisoning inside `SEQUENCE`.
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
  - [X] Implement `core_BigInt.UMatrixSum` (C++ Orthogonal Broadcast).
  - [X] Implement `BigInt.Sum` (Sign-grouping, pure scalar mapping, and AddSubRouter reconciliation).
  - [X] Stress test Matrix C++ broadcast engine limits (Thousands of rows).
- [X] **Phase 5 — Multiplication**
  - [X] Implement `core_BigInt.UMul` (Calculate context-safe $k$).
  - [X] Implement `BigInt.Mul` wrapper.
  - [X] Large-value stress tests ($10^{2k}$ boundary checks).
- [X] **Phase 6 — Division & Modulo**
  - [X] Implement `BigInt.Div`.
  - [X] Implement `BigInt.Mod`.
  - [X] `#DIV/0!` edge case hardening.
  - [X] Implement internal Newton-Raphson division and logic to choose when it is used.
- [ ] **Phase 7 — Higher-Order & Polish**
  - [ ] `BigInt.Pow`, `BigInt.Fact`, `BigInt.Sqrt`.
  - [ ] Range helpers (`BigInt.Min`, `BigInt.Max`).
  - [ ] Abuse-testing Excel engine (Tens of thousands of digits).

## Implementation
The library is implemented across three modules. First, `BigInt`, the public API. Second, `core_BigInt`, the private API. Third, `test_BigInt`, a private benchmarking suite. Excel modules automatically prepend the module name and a period onto its functions. For example, `UAdd`is part of the private API, which can call it without the prefix. The Excel grid and other modules must use `core_BigInt.UAdd` instead.

```excel
/////////////////////////////////////
// Big Integer Arithmetic Library //
//        BigInt Module           //
//          Public API            //
////////////////////////////////////

/**
 * Normalises a BigInt string. Strips leading zeros, resolves "-0" to "0", and throws #VALUE! on invalid characters.
 * @param big_int The BigInt string.
 */
Norm = LAMBDA(big_int,
    LET(
        str, IF(OR(ISOMITTED(big_int), big_int = ""), "0", big_int & ""),
        is_valid, REGEXTEST(str, "^-?\d+$"),
        stripped, REGEXREPLACE(str, "^(-?)0+(?=\d)", "$1"),

        IF(NOT(is_valid), VALUE("#VALUE!"), IF(stripped = "-0", "0", stripped))
    )
);

/**
 * Returns the sign of a BigInt: 1 (positive), -1 (negative), or 0 (zero).
 * @param big_int The BigInt string.
 */
Sign = LAMBDA(big_int,
    LET(
        normed, Norm(big_int),
        SWITCH(LEFT(normed),
            "0", 0,
            "-", -1,
            1
        )
    )
);

// --- Predicates --- \\

/**
 * Returns TRUE if the BigInt is exactly zero.
 * @param big_int The BigInt string.
 */
IsZero = LAMBDA(big_int, Norm(big_int) = "0");

/**
 * Returns TRUE if the BigInt is negative.
 * @param big_int The BigInt string.
 */
IsNeg = LAMBDA(big_int, LEFT(Norm(big_int)) = "-");

/**
 * Returns the unsigned magnitude of a BigInt string (absolute value).
 * @param big_int The BigInt string.
 */
Abs = LAMBDA(big_int,
    LET(
        num, Norm(big_int),
        IF(LEFT(num) = "-", MID(num, 2, LEN(num) - 1), num)
    )
);

/**
 * Compares two BigInt strings. Returns 1 (a > b), -1 (a < b), or 0 (a = b).
 * Evaluates signs and lengths. If identical, delegates to native limb array comparison.
 * @param big_int_a The first BigInt string.
 * @param big_int_b The second BigInt string.
 */
Compare = LAMBDA(big_int_a, big_int_b,
    LET(
        norm_a, Norm(big_int_a),
        norm_b, Norm(big_int_b),
        sign_a, BigInt.Sign(norm_a),
        sign_b, Bigint.Sign(norm_b),

        IF(sign_a <> sign_b,
            SIGN(sign_a - sign_b),
            IF(sign_a = 0,
                0,
                LET(
                    len_a, LEN(norm_a),
                    len_b, LEN(norm_b),

                    IF(len_a <> len_b,
                        SIGN(len_a - len_b) * sign_a,
                        LET(
                            k, core_BigInt.SafeK("ADD"),
                            limbs_a, core_BigInt.Split(BigInt.Abs(norm_a), k),
                            limbs_b, core_BigInt.Split(BigInt.Abs(norm_b), k),
                            mag_comparison, core_BigInt.UCompare(limbs_a, limbs_b),

                            mag_comparison * sign_a
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
 * @param big_int_a The first BigInt string.
 * @param big_int_b The second BigInt string.
 */
Add = LAMBDA(big_int_a, big_int_b,
    LET(
        norm_a, Norm(big_int_a),
        norm_b, Norm(big_int_b),
        sign_a, Bigint.Sign(norm_a),
        sign_b, BigInt.Sign(norm_b),

        k, core_BigInt.SafeK("ADD"),
        limbs_a, core_BigInt.Split(BigInt.Abs(norm_a), k),
        limbs_b, core_BigInt.Split(BigInt.Abs(norm_b), k),

        routed, core_BigInt.AddSubRouter(limbs_a, limbs_b, sign_a, sign_b, k),
        final_sign, CHOOSEROWS(routed, -1),
        final_limbs, DROP(routed, -1),

        merged, core_BigInt.Merge(final_limbs, k),

        IF(final_sign = -1, "-" & merged, merged)
    )
);

/**
 * Subtracts the second BigInt string from the first (A - B).
 * @param big_int_a The minuend BigInt string.
 * @param big_int_b The subtrahend BigInt string.
 */
Sub = LAMBDA(big_int_a, big_int_b,
    LET(
        norm_b, Norm(big_int_b),

        // Invert the sign of B at the text level
        inv_b, IF(norm_b = "0", "0",
            IF(LEFT(norm_b) = "-",
                MID(norm_b, 2, LEN(norm_b)),
                "-" & norm_b
            )
        ),

        // Pass to Add
        Add(big_int_a, inv_b)
    )
);

/**
 * Sums an array or range of BigInt strings via vectorized group matrices.
 * @param range A contiguous range or array of BigInt strings.
 */
Sum = LAMBDA(range,
    LET(
        flat, TOROW(range, 1),

        IF(OR(ISERROR(flat), COLUMNS(flat) = 0), "0",
            LET(
                norms, MAP(flat, LAMBDA(x, Norm(x))),
                signs, MAP(LEFT(norms), LAMBDA(x, IF(x = "0", 0, if(x = "-", -1, 1)))),
                abs_norms, MAP(norms, signs, LAMBDA(x, sign, IF(sign = -1, MID(x, 2, LEN(x)-1), x))),

                pos_strs, FILTER(abs_norms, signs = 1, {"0"}),
                neg_strs, FILTER(abs_norms, signs = -1, {"0"}),

                unified_k, core_BigInt.SafeK("ADD", COLUMNS(flat)),

                pos_limbs, core_BigInt.UMatrixSum(pos_strs, unified_k),
                neg_limbs, core_BigInt.UMatrixSum(neg_strs, unified_k),

                routed, core_BigInt.AddSubRouter(pos_limbs, neg_limbs, 1, -1, unified_k),
                final_sign, CHOOSEROWS(routed, -1),
                final_limbs, DROP(routed, -1),

                merged, core_BigInt.Merge(final_limbs, unified_k),

                IF(final_sign = -1, "-" & merged, merged)
            )
        )
    )
);

/**
 * Multiplies two BigInt strings.
 * @param big_int_a The first BigInt string.
 * @param big_int_b The second BigInt string.
 */
Mul = LAMBDA(big_int_a, big_int_b,
    LET(
        norm_a, Norm(big_int_a),
        norm_b, Norm(big_int_b),
        sign_a, BigInt.Sign(norm_a),
        sign_b, BigInt.Sign(norm_b),
        len_a, LEN(norm_a),
        len_b, LEN(norm_b),

        // Short-circuit: 0 multiplied by anything is 0
        IF(OR(sign_a = 0, sign_b = 0), "0",
        IF(len_a + len_b > 32766, #NUM!,
            LET(
                // Calculate dynamic safe K based on the shortest string
                k, core_BigInt.SafeK("MUL", MIN(len_a, len_b)),

                // Split absolute strings into little-endian arrays
                limbs_a, core_BigInt.Split(BigInt.Abs(norm_a), k),
                limbs_b, core_BigInt.Split(BigInt.Abs(norm_b), k),

                // Calculate magnitude and resolve sign
                final_limbs, core_BigInt.UMul(limbs_a, limbs_b, k),
                final_sign, sign_a * sign_b,

                merged, core_BigInt.Merge(final_limbs, k),

                IF(final_sign = -1, "-" & merged, merged)
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
RandBigIntArray = LAMBDA([num_rows], [num_cols], [min_digits], [max_digits],
    LET(
        n_rows, IF(ISOMITTED(num_rows), 1, num_rows),
        n_cols, IF(ISOMITTED(num_cols), 1, num_cols),
        min_len, IF(ISOMITTED(min_digits), 16, MIN(32767, MAX(1, min_digits))),
        max_len, IF(ISOMITTED(max_digits), 50, MAX(1, MIN(32766, max_digits))),

        MAP(RANDARRAY(n_rows, n_cols, min_len, max_len, TRUE), LAMBDA(length,
            LET(
                sign, CHOOSE(RANDBETWEEN(1, 2), "", "-"),
                digits, CONCAT(RANDARRAY(1, length, 0, 9, TRUE)),
                sign & REGEXREPLACE(digits, "^0*", "")
            )
        ))
    )
);

/**
 * Divides the first BigInt by the second (A / B).
 * Returns the quotient and modulus in rows 1 and 2, respectively.
 * @param big_int_a The dividend BigInt string.
 * @param big_int_b The divisor BigInt string.
 * @param [use_floor] Optional boolean. TRUE for Python-style Floor division, FALSE for Truncated. Defaults to FALSE.
 */
DivMod = LAMBDA(big_int_a, big_int_b, [use_floor],
    LET(
        norm_a, Norm(big_int_a),
        norm_b, Norm(big_int_b),
        sign_a, BigInt.Sign(norm_a),
        sign_b, BigInt.Sign(norm_b),
        is_floor, IF(ISOMITTED(use_floor), FALSE, use_floor),

        IF(sign_b = 0, #DIV/0!,
            IF(sign_a = 0, {"0";"0"},
                LET(
                    k, core_BigInt.SafeK("DIV"),
                    limbs_a, core_BigInt.Split(BigInt.Abs(norm_a), k),
                    limbs_b, core_BigInt.Split(BigInt.Abs(norm_b), k),

                    packed_result, core_BigInt.DivRouter(limbs_a, limbs_b, k),

                    // Extract Remainder and Quotient arrays
                    len_r, INDEX(packed_result, 1, 1),
                    rem_limbs, CHOOSEROWS(packed_result, SEQUENCE(len_r, 1, 2)),
                    quot_limbs, DROP(packed_result, len_r + 1),

                    // Resolve Truncated signs
                    final_sign, sign_a * sign_b,
                    merged_q, core_BigInt.Merge(quot_limbs, k),
                    merged_r, core_BigInt.Merge(rem_limbs, k),

                    trunc_q, IF(AND(final_sign = -1, merged_q <> "0") , "-" & merged_q, merged_q),
                    trunc_r, IF(and(sign_a = -1, merged_r <> "0"), "-" & merged_r, merged_r),

                    // Floor Correction: Trigger ONLY if floor is requested, signs differ, and remainder is non-zero
                    needs_correction, AND(is_floor, final_sign = -1, merged_r <> "0"),

                    final_q, IF(needs_correction, Sub(trunc_q, "1"), trunc_q),
                    final_r, IF(needs_correction, Add(trunc_r, norm_b), trunc_r),

                    VSTACK(final_q, final_r)
                )
            )
        )
    )
);

/**
 * Divides the first BigInt by the second (A / B).
 * @param big_int_a The dividend BigInt string.
 * @param big_int_b The divisor BigInt string.
 * @param [use_floor] Optional boolean. TRUE for Python-style Floor division. Defaults to FALSE.
 */
Div = LAMBDA(big_int_a, big_int_b, [use_floor],
    INDEX(DivMod(big_int_a, big_int_b, use_floor), 1, 1)
);

/**
 * Computes the remainder of A / B.
 * @param big_int_a The dividend BigInt string.
 * @param big_int_b The divisor BigInt string.
 * @param [use_floor] Optional boolean. TRUE for Python-style Modulo. Defaults to FALSE.
 */
Mod = LAMBDA(big_int_a, big_int_b, [use_floor],
    INDEX(DivMod(big_int_a, big_int_b, use_floor), 2, 1)
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
 * @param operation: "ADD" or "MUL".
 * @param [max_items] ADD: number of rows. MUL: shortest string digits count.
 */
SafeK = LAMBDA(operation, [max_items],
    LET(
        items, IF(OR(max_items = "", ISOMITTED(max_items)), 2,  MAX(2, max_items)),

        SWITCH(operation,
            "ADD", 15 - LEN(items),
            "MUL", INT((15 - LEN(items)) / 2),
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
Split = LAMBDA(big_ints, k,
    LET(
        big_int_row, TOROW(big_ints),
        lengths, LEN(big_int_row),
        max_len, MAX(lengths),
        pad_len, ROUNDUP(max_len / k, 0) * k,

        padded, REPT("0", pad_len - lengths) & big_int_row,
        starts, SEQUENCE(pad_len / k, 1, pad_len - k + 1, -k),

        --MID(padded, starts, k)
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
        n, ROWS(limbs),
        base, 10^k,
        scanned, SCAN(0, limbs, LAMBDA(acc, val, val + INT(acc / base))),
        remainders, MOD(scanned, base),
        final_carry, INT(INDEX(scanned, n, 1) / base),

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
        pad_a, EXPAND(limbs_a, max_rows, 1, 0),
        pad_b, EXPAND(limbs_b, max_rows, 1, 0),

        diff, pad_a - pad_b,
        msd_idx, XMATCH(TRUE, diff <> 0, 0, -1),

        IF(ISNA(msd_idx), 0, SIGN(INDEX(diff, msd_idx, 1)))
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
        pad_a, EXPAND(a, max_rows, 1, 0),
        pad_b, EXPAND(b, max_rows, 1, 0),

        Carry(pad_a + pad_b, k)
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
        pad_a, EXPAND(limbs_a, max_rows, 1, 0),
        pad_b, EXPAND(limbs_b, max_rows, 1, 0),

        diff, pad_a - pad_b,
        resolved, Carry(diff, k),

        msd_idx, XMATCH(TRUE, resolved <> 0, 0, -1),

        IF(ISNA(msd_idx), {0}, TAKE(resolved, msd_idx))
    )
);

/**
 * [Internal] Vectorized matrix sum of unsigned BigInt strings.
 * @param ubig_int Horizontal array (1xN) of normalized magnitude strings.
 * @param k The unified dynamic chunk size context.
 */
UMatrixSum = LAMBDA(ubig_ints, k,
    LET(
        grid, Split(ubig_ints, k),
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
        is_same_sign, sign_a = sign_b,

        IF(is_same_sign,
            // Addition: magnitudes combine, sign remains the same.
            VSTACK(UAdd(limbs_a, limbs_b, k), sign_a),

            // Subtraction: evaluate magnitudes to satisfy USub (A >= B) precondition.
            LET(
                mag_comp, UCompare(limbs_a, limbs_b),

                IF(mag_comp = 0,
                    // Total cancellation: return {0} limb and 0 sign.
                    {0;0},

                    IF(mag_comp > 0,
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
        len_a, ROWS(limbs_a),
        len_b, ROWS(limbs_b),
        max_rows, len_a + len_b - 1,

        // Loop purely over the length of the final output array
        raw_limbs, MAKEARRAY(max_rows, 1, LAMBDA(r, c,
            LET(
                // Calculate the valid overlapping window of indices for this magnitude
                start_i, MAX(1, r - len_b + 1),
                end_i, MIN(r, len_a),

                count, end_i - start_i + 1,

                // Sequence i counts UP. Sequence j counts DOWN.
                seq_i, SEQUENCE(count, 1, start_i, 1),
                seq_j, SEQUENCE(count, 1, r - start_i + 1, -1),

                // Slice the exact intersecting limbs, multiply, and sum natively
                SUM(INDEX(limbs_a, seq_i, 1) * INDEX(limbs_b, seq_j, 1))
            )
        )),

        // Resolve all carries bottom-to-top exactly once
        Carry(raw_limbs, k)
    )
);

/**
 * [Internal] Binary search to find the highest scalar quotient limb q (0 <= q < 10^k)
 * such that B * q <= acc. Eliminates the need for Knuth D float-estimation.
 * @param acc The current remainder array.
 * @param b The divisor array.
 * @param k The dynamic chunk size context.
 */
FindQuotientLimb = LAMBDA(acc, b, k,
    LET(
        // Calculate required bits dynamically. For max k=7, this evaluates to 24 bits.
        bits, CEILING.MATH(LOG(10^k, 2)),
        powers, 2 ^ SEQUENCE(bits, 1, bits - 1, -1),

        REDUCE(0, powers, LAMBDA(curr_q, power,
            LET(
                test_q, curr_q + power,

                // Skip if test_q exceeds the limb base
                IF(test_q >= 10^k,
                    curr_q,
                    LET(
                        // Multiply B by scalar test_q and resolve carries natively
                        test_prod, Carry(b * test_q, k),
                        comp, UCompare(acc, test_prod),

                        IF(comp >= 0, test_q, curr_q)
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
        mag_comp, UCompare(limbs_a, limbs_b),

        // Short-circuit routing with the new packed memory contract
        IF(mag_comp < 0,
            VSTACK(ROWS(limbs_a), limbs_a, 0),
            IF(mag_comp = 0,
                VSTACK(1, 0, 1),
                LET(
                    len_a, ROWS(limbs_a),
                    // Iterate over A from MSB (top) to LSB (bottom)
                    reversed_a, CHOOSEROWS(limbs_a, SEQUENCE(len_a, 1, len_a, -1)),

                    // State separator: VSTACK(len_remainder, remainder, quotient)
                    initial_state, VSTACK(1, 0, 0),

                    final_state, REDUCE(initial_state, reversed_a, LAMBDA(state, val,
                        LET(
                            len_remainder, INDEX(state, 1, 1),
                            remainder, CHOOSEROWS(state, SEQUENCE(len_remainder, 1, 2)),
                            quotient, DROP(state, len_remainder + 1),

                            // Shift acc up by 1 limb (Little-Endian: insert at index 1)
                            extended_remainder, IF(AND(ROWS(remainder) = 1, INDEX(remainder, 1, 1) = 0),
                                val,
                                VSTACK(val, remainder)
                            ),

                            next_quotient_limb, FindQuotientLimb(extended_remainder, limbs_b, k),
                            prod, Carry(limbs_b * next_quotient_limb, k),
                            new_remainder, USub(extended_remainder, prod, k),
                            has_quotient, OR(ROWS(quotient) > 1, INDEX(quotient,1,1) > 0),

                            next_quot, IF(has_quotient, VSTACK(quotient, next_quotient_limb), next_quotient_limb),

                            VSTACK(ROWS(new_remainder), new_remainder, next_quot)
                        )
                    )),

                    len_remainder, INDEX(final_state, 1, 1),
                    remainder, CHOOSEROWS(final_state, SEQUENCE(len_remainder, 1, 2)),
                    reversed_quotient, DROP(final_state, len_remainder + 1),

                    // quot_raw was built MSB to LSB (Big-Endian). Reverse to internal Little-Endian.
                    len_quotient, ROWS(reversed_quotient),
                    quotient, CHOOSEROWS(reversed_quotient, SEQUENCE(len_quotient, 1, len_quotient, -1)),

                    // Leading/trailing zeros are continuously removed inside the loop, no need to repeat it.

                    // Return strictly packed 1D array
                    VSTACK(len_remainder, remainder, quotient)
                )
            )
        )
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
        // Digit counts are approximate, but sufficient
        digits_a, ROWS(limbs_a) * k,
        digits_b, ROWS(limbs_b) * k,

        // Machine-learned coefficients from Logistic Regression
        m, 0.107821194271297,
        c, -221.902189462068,
        threshold, (m * digits_a) + c,

        IF(digits_b <= threshold,
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
        len_b, ROWS(limbs_b),
        seed_len, MIN(3, len_b),

        // Extract top 'seed_len' limbs (Little-Endian: take from the bottom)
        b_top, TAKE(limbs_b, -seed_len),

        // Construct Beta^(2 * seed_len) natively
        // e.g., if seed_len = 1, then 1 followed by two zero limbs: VSTACK(0, 0, 1)
        beta_2m, VSTACK(EXPAND(0, 2 * seed_len, , 0), 1),

        // Knuth Division yields exact initial reciprocal
        packed, KnuthDiv(beta_2m, b_top, k),

        // Extract Quotient array from the packed VSTACK
        len_r, INDEX(packed, 1, 1),
        DROP(packed, len_r + 1)
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
        len_b, ROWS(limbs_b),
        seed_len, MIN(3, len_b),
        r_seed_limbs, NewtonRaphsonSeed(limbs_b, k),

        initial_state, VSTACK(seed_len, r_seed_limbs),
        req_iters, MAX(0, CEILING.MATH(LN(target_len / seed_len) / LN(2))),

        IF(req_iters = 0,
            r_seed_limbs,
            LET(
                iterations, SEQUENCE(req_iters),
                final_state, REDUCE(initial_state, iterations, LAMBDA(state, iter,
                    LET(
                        curr_len, INDEX(state, 1, 1),
                        curr_r, DROP(state, 1),

                        next_len, MIN(curr_len * 2, target_len),
                        shift_limbs, next_len - curr_len,

                        r_prime, IF(shift_limbs > 0, VSTACK(EXPAND(0, shift_limbs, , 0), curr_r), curr_r),

                        // Slice B to its natural limits, padding with zeros destroys performance.
                        b_active, TAKE(limbs_b, -MIN(next_len, len_b)),

                        // O(N*M) instead of O(N^2) for P
                        p_raw, UMul(b_active, r_prime, k),

                        // Shifting the result array replicates Beta multiplication
                        shift_p, MAX(0, next_len - len_b),
                        p, IF(shift_p > 0, VSTACK(EXPAND(0, shift_p, , 0), p_raw), p_raw),

                        two_beta, VSTACK(EXPAND(0, 2 * next_len, , 0), 2),
                        err, USub(two_beta, p, k),

                        next_r_unscaled, UMul(r_prime, err, k),
                        next_r, IF(ROWS(next_r_unscaled) <= 2 * next_len, {0}, DROP(next_r_unscaled, 2 * next_len)),

                        VSTACK(next_len, next_r)
                    )
                )),
                DROP(final_state, 1)
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
        len_a, ROWS(limbs_a),
        len_b, ROWS(limbs_b),
        target_len, MAX(len_a, len_b),

        // 1. Calculate reciprocal R scaled dynamically to target_len
        r, NewtonRaphsonReciprocal(limbs_b, target_len, k),

        // 2. Approximate Quotient: Q_approx = A * R / Beta^(target_len + len_b)
        drop_limbs, target_len + len_b,
        q_unscaled, UMul(limbs_a, r, k),
        q_approx, IF(ROWS(q_unscaled) <= drop_limbs, {0}, DROP(q_unscaled, drop_limbs)),

        // 3. Error Correction Sweep (Remainder = A - (B * Q_approx))
        q_prod, UMul(limbs_b, q_approx, k),
        rem_approx, USub(limbs_a, q_prod, k),

        comp, UCompare(rem_approx, limbs_b),

        // Correct floor truncation from integer reciprocal
        final_q, IF(comp >= 0, UAdd(q_approx, {1}, k), q_approx),
        final_r, IF(comp >= 0, USub(rem_approx, limbs_b, k), rem_approx),

        VSTACK(
            ROWS(final_r),
            final_r,
            final_q
        )
    )
);
```

```excel
////////////////////////////////////
// Big Integer Arithmetic Library //
//       test_BigInt Module       //
//       Private Testing API      //
////////////////////////////////////

KnuthDivision=LAMBDA(a, b,
    LET(
        k, core_BigInt.SafeK("DIV"),
        packed, core_BigInt.KnuthDiv(core_BigInt.Split(a, k), core_BigInt.Split(b, k), k),

        core_BigInt.Merge(DROP(packed, INDEX(packed, 1, 1) + 1), k)
    )
);

NewtonRaphsonDivision = LAMBDA(a, b,
    LET(
        k, core_BigInt.SafeK("DIV"),
        packed, core_BigInt.NewtonRaphsonDiv(core_BigInt.Split(a, k), core_BigInt.Split(b, k), k),

        core_BigInt.Merge(DROP(packed, INDEX(packed, 1, 1) + 1), k)
    )
);

/**
 * @param a The dividend string
 * @param b The divisor string
 * @param mode "KNUTH" or "NR"
 * @param iters Inner loop count (to defeat chunky clock resolution)
 * @param repeats Outer loop count (to defeat OS/Garbage Collection noise)
 */
DivisionAlgorithms = LAMBDA(a, b, mode, iters, repeats,
    LET(
        // Map the outer repeats loop
        batch_times, MAP(SEQUENCE(repeats), LAMBDA(r,
            LET(
                start, NOW(),

                // Map the inner execution loop
                run, IF(mode = "KNUTH",
                        MAP(SEQUENCE(iters), LAMBDA(i, KnuthDivision(a, b))),
                        MAP(SEQUENCE(iters), LAMBDA(i, NewtonRaphsonDivision(a, b)))
                ),

                // Force evaluation before pulling stop time
                stop, NOW() + (0 * LEN(INDEX(run, 1, 1))),

                // Total batch time in milliseconds
                (stop - start) * 86400000
            )
        )),

        // Extract the cleanest run and average it down to a single execution time
        MIN(batch_times) / iters
    )
);

/**
 * Executes a strictly sequential benchmark suite to avoid multi-threading contamination.
 * @param test_cases An N x 2 array of test lengths: [Length_A, Length_B]
 * @param iters Number of iterations per benchmark to smooth NOW() resolution.
 */
DivisionAlgorithmComparison = LAMBDA(test_cases, iters, repeats,
    LET(
        n_tests, ROWS(test_cases),

        // Initial state: Data headers for CSV export
        initial, {"Len_A", "Len_B", "Knuth_ms", "NR_ms"},

        REDUCE(initial, SEQUENCE(n_tests), LAMBDA(acc, i,
            LET(
                len_a, INDEX(test_cases, i, 1),
                len_b, INDEX(test_cases, i, 2),

                // Skip Divisor > Dividend—it's never an integer
                IF(len_b > len_a,
                    acc,
                    LET(
                        // Generate exact-length test strings natively
                        str_a, REPT("9", len_a),
                        str_b, IF(len_b = 1, "7", "7" & REPT("3", len_b - 1)),

                        // Execute benchmarks strictly one after the other
                        time_k, DivisionAlgorithms(str_a, str_b, "KNUTH", iters, repeats),
                        time_nr, DivisionAlgorithms(str_a, str_b, "NR", iters, repeats),

                        // Append the results.
                        // Reallocation delay occurs here, safely outside the timed window.
                        VSTACK(acc, HSTACK(len_a, len_b, time_k, time_nr))
                    )
                )
            )
        ))
    )
);

// Generates a 2-column array of every combination of lengths
DivisionBenchmarkCases = LAMBDA(start_len, max_len, step,
    LET(
        seq, SEQUENCE((max_len - start_len) / step + 1, 1, start_len, step),
        n, ROWS(seq),

        // C++ style Cartesian Join
        col_a, TOCOL(IF(SEQUENCE(1, n), seq)),
        col_b, TOCOL(IF(SEQUENCE(n, 1), TOROW(seq))),

        HSTACK(col_a, col_b)
    )
);

// Generates a Cartesian grid from two independent sequences
AsymmetricABGrid = LAMBDA(start_a, max_a, step_a, start_b, max_b, step_b,
    LET(
        seq_a, SEQUENCE((max_a - start_a) / step_a + 1, 1, start_a, step_a),
        seq_b, SEQUENCE((max_b - start_b) / step_b + 1, 1, start_b, step_b),

        n_a, ROWS(seq_a),
        n_b, ROWS(seq_b),

        // Orthogonal broadcast using independent dimensions
        col_a, TOCOL(IF(SEQUENCE(1, n_b), seq_a)),
        col_b, TOCOL(IF(SEQUENCE(n_a), TOROW(seq_b))),

        HSTACK(col_a, col_b)
    )
);
```

## Current Priority
With division full functional, it's time to move on to the higher order functions.