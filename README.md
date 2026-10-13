# arr-im
> Only the README.md is here. I publish the code on Codeberg. [view →](https://codeberg.org/slnknrr/arrim_lib.js-for-pm_npm).

**Array structure algebra. Not lodash improved — the primitives `Array` never had.**

`arr-im` adds exactly the array operations JavaScript is missing — and nothing it already has. It never re-implements `map`, `filter`, `sort`, `includes` or `flat`-into-a-new-array. It implements the things you keep hand-writing (with bugs): deep comparison that **survives cycles**, structural diff, depth and shape, strides and multi-dimensional offsets, flattening **without building an array**, windows **without copying**, frequency and entropy, run detection, and the in-place primitives the standard library skipped — rotate, stable partition, range reverse, block swap.

Every method takes a plain `Array`, any `TypedArray` (by element value) or a `View` — this library's zero-copy window — and walks it through one adapter resolved once per call. Nested containers are entered; everything else is a leaf.

```js
import a from '@slnknrr/arr-im';

a.eq([1, [2, 3]], [1, new Uint8Array([2, 3])]); // -1 — same elements, one container of another kind
a.eq(cyclic, alsoCyclic);                        // 1  — const c = []; c.push(c) is data, not a stack overflow
[...a.diff([1, [2, 3]], [1, [2, 4], 5])];        // [[1, 1], [2]] — paths to what differs
a.shape([[1, 2], [3, 4]]);                        // [2, 2]; a ragged structure gives null
a.offset([3, 4, 5], [1, 2, 3]);                   // 33 — row-major, like NumPy
[...a.flatn([[1, 2], [3], [[4, 5]]])];            // [0, 2, 3, 5] — Arrow-style offsets of the flattened stream
a.rot(a.view(arr, 2, 8), 3);                      // rotate a window of arr in place, O(1) memory
a.minmax(halfAMillion);                           // [min, max] in one pass — Math.min(...arr) would overflow the stack
```

---

## What it is

- **A precision tool for the structure of arrays.** Comparison, diff, nesting, shape, windows, statistics, in-place rearrangement — the questions about *containers* rather than about the business meaning of what is inside them.
- **Cycle-safe throughout.** Every recursive walk carries the stack of the containers it is inside. Equality is decided by bisimulation, `depth` reports `Infinity`, `shape` reports `null`, `cyc` counts the back-references, and `flat` throws instead of streaming forever.
- **Lazy and bounded by default.** Iterators stop when you stop pulling. `max` lets you ask "more than *N* distinct?" and leave. `flat` yields the first leaf without touching the rest.
- **Source-agnostic, copy-free.** A `Uint16Array` is its elements. A `View` is a window into someone else's array, readable *and writable* by every method here. Register your own container type once and the whole family, nesting included, works with it.
- **Pure ESM, dependency-free, sync.** One file, ~800 lines, Node ≥ 20. Hand-authored TypeScript declarations.

## What it is NOT

- **Not a deep-object-equality library.** Inside a structure, arrays, typed arrays and Views are containers; *everything else is a leaf* compared with SameValueZero. A plain object is a leaf. Dates, Maps, prototypes, getters — that is a different library with different questions. `{ leaf }` lets you plug one in where you need it.
- **Not a faster `for` loop.** A hand-written `===` recursion over a numeric array beats `eq` by 3× — see [Performance](#performance). `arr-im` wins where the native idiom is forced to do work you didn't ask for.
- **Not a byte library.** A `Uint16Array([1])` here is one element, `1`. Its two bytes are [`@slnknrr/buf-im`](https://www.npmjs.com/package/@slnknrr/buf-im)'s business. Strings are [`@slnknrr/str-im`](https://www.npmjs.com/package/@slnknrr/str-im)'s.
- **Not lodash.** No `groupBy`, `zip`, `shuffle`, `sortBy`, `uniqBy`. Those are application logic over arrays; this is the algebra underneath.

---

## Install

```sh
npm install @slnknrr/arr-im
```

```js
import a from '@slnknrr/arr-im';                          // ready-to-use singleton — no `new`
import { arrim, _arrim, View } from '@slnknrr/arr-im';    // the class (use() / instanceof), the factory, the window class
```

Requirements: **Node ≥ 20**, ESM only (`"type": "module"` or `import`).

---

## Core conventions

These are load-bearing. Learn them once; they apply everywhere.

**A source is a container of elements.** An `Array`, any `TypedArray` (its elements, by value — a `DataView` is not one), a `View`, or a type registered with `arrim.use`. Strings, Sets, Maps, array-likes and generic iterables are **not** sources (`TypeError`): a source has random access by index, or it is not an array in any sense that matters here.

**Inside a structure, containers are entered and everything else is a leaf.** Leaves compare with **SameValueZero** — `===` with `NaN` equal to `NaN`, the rule of `includes` and `Set`. Plain objects are leaves, compared by identity. `{ depth }` limits how many container levels are entered; `{ leaf }` replaces the leaf equality.

```js
a.eq([{ id: 1 }], [{ id: 1 }]);                                  // 0 — two objects, two leaves
a.eq([{ id: 1 }], [{ id: 1 }], { leaf: (x, y) => x.id === y.id }); // 1
a.eq([1, [2]], [1, [2]], { depth: 1 });                          // 0 — at the limit, containers are leaves: identity
```

**Predicates return `0 | 1 | -1`; the sign is the kind.** `0` = no. `1` = yes, and every pair of containers compared along the way was of the same kind (the same constructor). `-1` = yes, but some pair differed in kind. `-1` is truthy, so `if (a.eq(x, y))` reads naturally — and "same elements" stays distinguishable from "same elements *and* same types" without a second call.

```js
a.eq([1, 2, 3], [1, 2, 3]);                  // 1
a.eq([1, 2, 3], new Uint8Array([1, 2, 3]));  // -1  same elements, another kind
a.eq([1, [2]], [1, new Float32Array([2])]);  // -1  a nested kind mismatch counts
a.eq([1, 2, 3], a.view(big, 10, 13));        // -1  a View of the same elements
a.eq([1, 2, 3], [1, 2, 4]);                  // 0
```

`cmp` is the one exception: it is a comparator (`-1 / 0 / 1` by content, deep, lexicographic) and ignores kind.

**Cycles are data, not crashes.** `const x = []; x.push(x)` is a legal value. Two cyclic structures are equal when nothing in their unrolling distinguishes them — `x = [x]` equals `c = [c]`, and equals `m1 = [m2], m2 = [m1]`. A cyclic structure against a finite one is not equal. `depth` is `Infinity`, `shape` is `null`, `cyc` counts, `flat` throws `RangeError` when it reaches one.

**Validation is synchronous.** Every iterator is primed: all argument checks happen at the call site, not on the first `next()` in someone else's stack.

**Errors are typed.** `TypeError` = wrong type (not a source, a read-only source in a writer, a non-function `leaf`). `RangeError` = an index, count, depth or shape out of range, or a cycle where a finite structure was required.

**Writers fail before the first move.** `rev`, `rot`, `swap` and `part` validate every range up front and return the target; `part` calls the predicate on every element, in order, *before* anything moves, so a throwing predicate leaves the array untouched.

**Saturating counts return `Infinity`.** `a.ulen(arr, 100)` returns `Infinity` once 100 distinct elements are seen — an honest "at least this many".

---

## Sources & adapters

Register your own container type once, process-wide, and the entire method family — nesting, windows, writers — works with it:

```js
arrim.use(Ring, {
  size: (r) => r.length,          // number of elements
  at:   (r, i) => r.get(i),       // the element at i
  put:  (r, i, v) => r.set(i, v), // optional: without it the type is read-only
});
```

A `View` over a registered type reads and writes through its adapter like over anything else.

---

## Performance

`arr-im` is fast where laziness, windows without copies and single-pass answers matter. It does not compete with a tight `for` loop over a flat numeric array, and says so out loud.

Representative run (`npm run bench`, Node 24, one machine — your numbers will differ; the **shape** won't):

| Case | arr-im | the native way | speedup |
|---|--:|---|--:|
| distinct count of 500k, stop at 1000 | 29k/s | `new Set(arr).size` | **~1750×** |
| first 10 leaves of 100k | 330k/s | `flat(Infinity).slice(0, 10)` | **~1400×** |
| first element of 500 chunks of 1000 | 33k/s | `slice` per chunk | **~24×** |
| min and max of 500k numbers | 444/s | `Math.min(...)` in 50k slices | **~15×** |
| shallow equal, 500k numbers | 293/s | `every((v, i) => v === b[i])` | **~2.5×** |
| deep equal, 3 levels, 100k leaves | 279/s | `JSON.stringify(a) === JSON.stringify(b)` | **~2.3×** |
| rotate 500k in place | 462/s | `slice` + `concat` + copy back | **~1.6×** |
| deep equal, 3 levels, 100k leaves | 277/s | a hand-written `===` recursion | 0.32× |
| stable partition 500k in place | 7/s | two `filter`s + copy back | 0.16× |

Why the wins are structural: `ulen(max)` stops at the 1000th distinct element while `Set` must see all 500k; `flat` yields leaf by leaf while `flat(Infinity)` builds all 100k; `chunk` hands out Views while `slice` copies; `minmax` is one pass with no spread. The two losses are honest: the generic walk (adapters, kind, cycle stack) costs ~3× over a bare recursion, and the stable in-place partition trades time (O(n log n) moves) for memory (O(1) instead of a second array).

---

## API

41 methods across 7 groups. `arr`, `a`, `b`, `sub` accept any [source](#sources--adapters); the writers need a writable one. `opts` for the deep operations is `{ depth, leaf }`; for the ordered ones `{ order }`. Iterator methods return lazy `IterableIterator`s.

### 1 · Predicates — `0 / 1 / -1`, deep, cycle-safe
| Method | Asks |
|---|---|
| `eq(a, b, opts?)` | the same structure and the same leaves? |
| `ne(a, b, opts?)` | different? — `ne(a, b) === 0` exactly when `eq(a, b) !== 0` |
| `ge(a, b, opts?)` | does `a` start with `b`, element by element? (prefix match + `a` at least as long) |
| `le(a, b, opts?)` | is `a` a prefix of `b`? |
| `cmp(a, b, {order}?)` | lexicographic `-1 / 0 / 1`, deep, a comparator for `sort`; a leaf sorts before a container; kind ignored |

```js
rows.sort((x, y) => a.cmp(x, y));   // [[1, 2], [1, 9], [2, 1]] — rows of a matrix in order
```

### 2 · Search & compare — elements match deeply
| Method | Returns |
|---|---|
| `find(arr, sub, {all}?)` | lazy indices where the contiguous sub-array `sub` occurs; `all` reports overlapping ones |
| `lfind(arr, sub, {all}?)` | right to left |
| `com(a, b)` / `comb(a, b)` | length of the common prefix / suffix |
| `diffn(a, b)` | lazy top-level indices where `a` and `b` differ, including the indices only the longer one has — one number per difference, nothing allocated |
| `diff(a, b)` | lazy **paths** to every differing leaf: `[1, 0]` means `a[1][0]` vs `b[1][0]`; a length mismatch is the path of the first missing index |

`find` takes one needle: with nesting first-class there is no honest way to tell "many needles" from "a needle whose elements are arrays".

### 3 · Structure
| Method | Returns |
|---|---|
| `depth(arr, max=∞)` | nesting depth: `[]` and `[1]` are `1`, `[[1]]` is `2`; `Infinity` at `max`, and for a cycle |
| `shape(arr)` | the dimensions of a regular structure — `[[1, 2], [3, 4]]` → `[2, 2]` — or `null` when ragged, mixed or cyclic |
| `strides(shape)` | row-major strides: `[3, 4, 5]` → `[20, 5, 1]` |
| `offset(shape, index)` | the linear offset of a multi-index; out of bounds throws |
| `coord(shape, offset)` | the inverse |
| `flat(arr, depth=∞)` | the leaves, lazily, depth-first; `depth` limits the levels entered, exactly as `Array.prototype.flat` counts them; a cycle throws when reached |
| `flatn(arr, depth=∞)` | **offsets** of the flattened stream: where each top-level element begins, and the total — an Arrow-style offset buffer for a ragged array |
| `cyc(arr, max=∞)` | the number of back-references (an element that is its own ancestor); `0` = acyclic; shared references are not cycles |
| `cycn(arr)` | lazy paths to every back-reference |

```js
const dims = a.shape(grid);                    // [rows, cols]
const flat = [...a.flat(grid)];                // row-major
flat[a.offset(dims, [r, c])];                  // grid[r][c]
[...a.flatn(ragged)];                          // offsets: piece k of the flat stream is [o[k], o[k+1])
```

### 4 · Windows — zero-copy
| Method | Returns |
|---|---|
| `view(arr, start=0, end=length)` | a `View` of `[start, end)`; a View of a View composes the offsets, never a chain; out of range throws |
| `chunk(arr, size)` | lazy Views of `size` elements, the last one shorter |
| `chunkn(arr, size)` | the boundaries `0, size, 2·size, …, length`; piece *k* is `[b[k], b[k+1])` |
| `win(arr, n)` | lazy sliding Views of `n` consecutive elements |

A `View` has `length`, `at(i)`, `[Symbol.iterator]`, `toArray()`, `toJSON()` and the fields `src`, `start`, `end`. It reads the live array — a change in `arr` shows through — and it is a source everywhere in this library, **writers included**: `a.rot(a.view(arr, 2, 8), 3)` rotates a window of `arr` in place. `instanceof View` tells one apart; instances come only from `view`, `chunk` and `win`.

### 5 · Statistics — over element values, containers by identity
| Method | Returns |
|---|---|
| `ulen(arr, max=∞)` | distinct elements; `Infinity` at `max` |
| `uniq(arr, max=∞)` / `uniqn(arr, max=∞)` | distinct elements in first-seen order / the index of each first appearance |
| `freq(arr)` | `Map<element, count>`, first-seen order — the one deliberate allocation |
| `ent(arr, base=2)` | Shannon entropy of the element distribution |
| `minmax(arr, {order}?)` | `[min, max]` in one pass; `NaN` is skipped wherever it stands, as `d3.min` does; `undefined` for an empty or all-`NaN` source |
| `sum(arr)` | Neumaier-compensated sum: `[1e100, 1, -1e100]` is `1` here and `0` with `reduce`; a BigInt source is summed exactly as BigInt |

Flatten first when the leaves are what you mean: `a.ulen([...a.flat(nested)])`.

### 6 · Runs — consecutive equal elements
| Method | Returns |
|---|---|
| `consc` / `rconsc` `(arr, max=∞)` | length of the first / last run; `Infinity` at `max` |
| `cons` / `rcons` `(arr, max=∞)` | the element of each run, from the start / the end |
| `consg` / `rconsg` `(arr, max=∞)` | the length of each run |

### 7 · In place — the only writers
| Method | Does |
|---|---|
| `rev(arr, start=0, end=length)` | reverse a range — `Array.prototype.reverse` can only do all of it |
| `rot(arr, k)` | rotate left by `k` (negative: right) with O(1) extra memory — `std::rotate` |
| `swap(arr, i, j, n=1)` | swap the blocks `[i, i+n)` and `[j, j+n)`; overlapping blocks throw |
| `part(arr, pred)` | **stable** partition: the elements for which `pred` holds first, in their order, then the rest, in theirs; returns the split index. No buffer: O(n log n) moves and O(log n) stack, by rotation — `std::stable_partition` without the scratch memory |

Each returns the target (`part` returns the index) and works on an `Array`, a `TypedArray`, a writable registered type, or a `View` of any of them.

### Statics & factory
| Symbol | Purpose |
|---|---|
| `arrim.use(ctor, adapter)` | register a container type |
| `_arrim(overrides?)` | build a configured instance (see below) |
| `arrim` | the class, for `use()` and `instanceof` |
| `View` | the window class, for `instanceof` |

---

## Extending & configuring

The default export is a singleton — you never write `new`. To specialize behavior, pass overrides to `_arrim`. An override may call `super` to wrap the original:

```js
import { _arrim } from '@slnknrr/arr-im';

let comparisons = 0;
const counted = _arrim({
  _eq(x, y, leaf, depth, stack) {
    comparisons++;                              // the instance itself is frozen; count outside
    return super._eq(x, y, leaf, depth, stack);
  },
});
```

Every internal call dispatches through `this`, so replacing one method replaces it **everywhere** it is used.

> ### ⚠️ Keep configurations few and long-lived
> Internal call sites are monomorphic and V8 inlines them to zero cost — **as long as few distinct `arr-im` classes exist in the process.** Create your configuration **once, at module load**, and reuse it. `_arrim` caches by the `overrides` object (a `WeakMap`), so the *same object* always yields the *same instance*.

---

## Design notes

- **Bisimulation, not "recursion depth".** The deep walks carry a stack of the *pairs* being compared. The same pair recurring is equal by assumption; a container recurring with a different partner is simply a new pair, compared on its own. The pairs are finite, so the walk ends, and `x = [x]` comes out equal to `c = [c]` — which a depth counter or an identity map keyed by one side gets wrong.
- **Leaves settle inline.** In `eq`, two primitives are compared without a call; the recursion is paid only for containers. That is what keeps a flat numeric array within 3× of a bare loop while every element still goes through the pluggable adapter.
- **`shape` is judged at the end.** A leaf level seen early cannot be accepted until the whole structure has been walked: `[[1], [[2]]]` looks regular after its first element and is not.
- **`flatn` is `flat`'s ledger.** The offsets count leaves under the same depth rule and the same cycle rule, so `flat(arr, d)` sliced at `flatn(arr, d)` gives back exactly `flat([arr[i]], d)` for every `i`.
- **Stable partition by rotation.** Partition each half, then rotate the middle so the second half's matches follow the first half's: O(n log n) element moves, O(log n) stack, no buffer. The verdicts are taken once, before any move, and travel with the elements.

---

## Scripts

| Command | Does |
|---|---|
| `npm test` | behavioral suite (`node --test`, zero dependencies) — against `Array.prototype.flat`, slice-based rotation, filter-based partition, and brute-force references |
| `npm run types` | type-check the shipped declarations (`tsc --noEmit`) |
| `npm run bench` | the benchmarks above |

---

## Links

- Source: [https://codeberg.org/slnknrr/arrim_lib.js-for-pm_npm/src/branch/main](https://codeberg.org/slnknrr/arrim_lib.js-for-pm_npm/src/branch/main)
- npm: [https://www.npmjs.com/package/@slnknrr/arr-im](https://www.npmjs.com/package/@slnknrr/arr-im)
- Bytes: [@slnknrr/buf-im](https://www.npmjs.com/package/@slnknrr/buf-im) · text: [@slnknrr/str-im](https://www.npmjs.com/package/@slnknrr/str-im) · numbers as text: [@slnknrr/num-im](https://www.npmjs.com/package/@slnknrr/num-im)

## Author

Yury Slinkin (Юрий Слинкин)

- Email: slnknrr.dev@gmail.com
- Codeberg: https://codeberg.org/slnknrr

## License

MIT. See [LICENSE.md](https://codeberg.org/slnknrr/arrim_lib.js-for-pm_npm/src/branch/main/LICENSE.md).
