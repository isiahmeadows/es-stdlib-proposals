# Proposal of various array additions

Arrays are incredibly useful. They make working with structured data much easier. But there are a few things I'd like to see. Each of them are detailed with their related algorithm and rationale.

## Array.prototype.set(*array* [ , *srcOffset* [ , *destOffset* [ , *count* ] ] ])

This assigns to `this`, starting at `destOffset`, the first `count` items in `array` starting from `srcOffset`. I included more spec-like text modeled after `Array.prototype.copyWithin` and `TypedArray.prototype.set` to make it a little clearer what I'm looking for.

(The parameter order is to provide sane defaults - the common case for Java's `System.arraycopy` uses `srcPos = 0` and `len = src.length`.)

1. Let *O* be ? ToObject(**this** value).
1. Let *A* be ? ToObject(*array*).
1. Let *destLen* be ? ToLength(? Get(*O*, **"length"**)).
1. Let *srcLen* be ? ToLength(? Get(*A*, **"length"**)).
1. If *srcOffset* is **undefined**, then let *srcOffset* be 0. Else, let *srcOffset* be ? ToLength(*srcOffset*).
1. If *destOffset* is **undefined**, then let *destOffset* be 0. Else, let *destOffset* be ? ToLength(*destOffset*).
1. If *count* is **undefined**, then let *count* be *destLen*. Else, let *count* be ? ToLength(*count*).
1. If *count* < 0, throw a **RangeError** exception.
1. Let *srcEnd* be min(*srcLen*, *srcOffset* + *count*).
1. Let *destEnd* be min(*destLen*, *destOffset* + *count*).
1. Let *i* be *srcOffset*.
1. Let *j* be *offset*.
1. Repeat, while *i* < *srcEnd* and *j* < *end*:
    1. Let *toKey* be ! ToString(*i*).
    1. Let *fromKey* be ! ToString(*j*).
    1. Let *fromPresent* be ? HasProperty(*A*, *fromKey*).
    1. If *fromPresent* is **true**, then:
        1. Let *value* be ? Get(*A*, *fromKey*).
        1. Perform ? CreateDataPropertyOrThrow(*O*, *toKey*, *value*).
    1. Else *fromPresent* is **false**:
        1. Perform ? DeletePropertyOrThrow(*O*, *toKey*).
    1. Increase *i* by 1.
    1. Increase *j* by 1.
1. Return **undefined**.

### Why?

For `Array.prototype.set`, consider the surprisingly frequent usage of [`System.arraycopy`](https://docs.oracle.com/javase/9/docs/api/java/lang/System.html) within Java circles. In performance-sensitive code when you need to copy items across two arrays, it'd be nice to have a native JIT primitive that can do it in a very highly vectorized fashion. Such a method already exists in typed arrays, but it'd be nice to have that parity be moved over to normal arrays, too, since most normal JS code (even perf-sensitive code) can't lower *all* operations into plain numbers. As for other precedent:

- JS already has `TypedArray.prototype.set`.
- I've seen `Object.assign(array, values)` in the wild more than once, even though it's clearly wrong.
- Python has the `s[i:j] = t` idiom, which replaces one sublist with another (an extended version of our `.splice`).
- C# has [`System.Array.Copy`](https://docs.microsoft.com/en-us/dotnet/api/system.array.copy?view=netframework-4.7) (which is effectively Java's `System.arraycopy`), [`System.Buffer.BlockCopy`](https://docs.microsoft.com/en-us/dotnet/api/system.buffer.blockcopy?view=netframework-4.7) (equivalent specialized for primitives), and [`System.Array.CopyTo`](https://docs.microsoft.com/en-us/dotnet/api/system.array.copyto?view=netframework-4.7) for the common case of copying a smaller array's contents into a larger array.
- Rust has [`Vec::copy_from_slice`](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.copy_from_slice).
- OCaml has [`Array.blit`](https://caml.inria.fr/pub/docs/manual-ocaml/libref/Array.html) that's effectively Java's `System.arraycopy`.
- C++ has [`std::copy`](http://www.cplusplus.com/reference/algorithm/copy/), which operates on pointer offsets only.
- C, of course, has [`memcpy`](www.cplusplus.com/reference/clibrary/cstring/memcpy/).

## %TypedArray%.prototype.set(*overloaded* [ , *srcOffset* [ , *destOffset* [ , *count* ] ] ])

Update the offset handling to work similarly to my proposed `Array.prototype.set`. Note that the *srcOffset* is functionally the same as the existing second *offset* parameter within the spec, so it's not super breaking.

### Why?

I find it surprising that they're *not* there in the typed array variant, and I feel it limits its utility by quite a bit. It's something engines already have to have just to perform the algorithm, so it shouldn't be that hard to allow users to provide info to hook into it.

## Array.prototype.reject(*func* [ , *thisValue* ])

Basically the same as `Array.prototype.filter`, but with a negated `func`. It could be fully polyfilled as this:

```js
const filter = Function.call.bind(Function.call, Array.prototype.filter)

Object.assign(Array.prototype, {
    reject(func, thisValue = void 0) {
        return filter(this, (...args) => !func.apply(thisValue, args))
    }
})
```

### Why?

1. It gets annoying having to do `.filter(x => !isEmpty(x))` when the inverse, `.filter(isNotEmpty)`, is much shorter to type.
1. We already have `.filter`. This is just a few small lines of code.
1. [Underscore](http://underscorejs.org/#reject), [Lodash](https://lodash.com/docs#reject), and [Ramda](http://ramdajs.com/docs/#reject) all three have had it for pretty much the whole time they've existed.
1. Most languages that have a `filter` method/function (e.g. most FP languages) or equivalent (e.g. Ruby/Smalltalk) have an inverse with that name.
    - Python is one of the few exceptions, with a `filter` global and an [`itertools.filterfalse`](https://docs.python.org/3/library/itertools.html#itertools.filterfalse) standard library export.

## Array.prototype.insert(*index*, ...*items*)

Basically the same as `Array.prototype.splice`, but with a couple differences:

1. No ability to remove items when replacing.
1. Returns the new length like `.push`, rather than an empty array.

### Why?

1. It'd be nice to have something more explicit than `array.splice(index, 0, ...items)`, with that magic-seeming 0.
1. Many languages already offer something like the above (and idiomatically prefer it), even though they have an equivalent to `.splice`:
    - Python via [`MutableSequence.insert`](https://docs.python.org/3/library/stdtypes.html#mutable-sequence-types) (`s.insert(i, x)` &harr; `s[i:i] = [x]`)
    - Ruby via [`Array#insert`](https://docs.ruby-lang.org/en/2.0.0/Array.html#method-i-insert) (`ary.insert(i, xs...)` &harr; `ary[i, i] = [xs...]`)
    - Java via [`ArrayList.add`](https://docs.oracle.com/javase/9/docs/api/java/util/ArrayList.html#add-int-E-) (`list.add(i, x)` &harr; `list.addAll(i, new T[] {x})`)

## Array.prototype.shuffle([ *randomFunc* ])

Shuffle the array's entries in-place via the [Fisher-Yates shuffle](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle) (maybe via [the "inside-out" algorithm](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle#The_%22inside-out%22_algorithm)?) and returns **`this`**.

- By default, *randomFunc* is morally equivalent to `max => Math.floor(Math.random() * max)` (don't actually do this - it will likely fail the last requirement), but if *randomFunc* is a function, it uses that instead. (This is in case a better random number generator is required, like with large arrays.)
- *randomFunc* is called with the number of remaining indices *max*, and it should return an integer *result*, which is then clamped to 0 ≤ *result* < *max* and used as the index.
- For all arrays and array-likes with length at most 64, all permutations must be theoretically reachable from a shuffle without a custom *randomFunc*. (This is for things like card draws and similar.)

### Why?

1. This substantially hard to get right. See [this thread](https://esdiscuss.org/topic/proposal-array-prototype-shuffle-fisher-yates) for some background. (If you read it, I did express skepticism, so I'm not *very* behind this, but I can see the use of it.)
    - The fact many of the answers [here](https://stackoverflow.com/questions/2450954/how-to-randomize-shuffle-a-javascript-array/18650169#18650169) are either based on well-known algorithms, really inefficient (map with random tags, sort, map to remove tags), or just flat out wrong (`.sort` by `Math.random() - 0.5` should be telling.
    - With pseudo-random number generators, [there's a very strong risk of bias](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle#Pseudorandom_generators), just due to the limitations of the generator and the way the algorithm uses random numbers. Assuming a high-quality generator with 128-bit seeds (like xorshift128), you can only safely shuffle up to 34-entry arrays while still being able to hit every permutation. You can stretch it up to 170 with something with 1024-bit states (like [xorshift1024*](https://en.wikipedia.org/wiki/Xorshift#xorshift*)) or a few thousand entries with the slower, heavier [Mersenne Twister](https://en.wikipedia.org/wiki/Mersenne_Twister) family or (higher-quality, MT-derived) [WELL family of generators](https://en.wikipedia.org/wiki/Well_equidistributed_long-period_linear), but you rarely find anything better that's not a cryptography primitive (like a high-entropy hardware random number generator).
3. Many languages offer something similar:
    - Ruby via [`Array#shuffle!`](https://ruby-doc.org/core-2.3.0/Array.html#method-i-shuffle-21)
    - PHP via [`bool shuffle(array &$array)`](https://secure.php.net/manual/en/function.shuffle.php)
    - Python via [`random.shuffle(x)`](https://docs.python.org/3.6/library/random.html#random.shuffle)
    - Java via [`Collections.shuffle(List<?> list)`](https://docs.oracle.com/javase/9/docs/api/java/util/Collections.html#shuffle-java.util.List-)
    - Clojure via [`clojure.core/shuffle`](https://clojuredocs.org/clojure.core/shuffle)
