---
title: "Fixing a bug in the Go compiler as a newbie: a deep dive (I)"
heading: "Fixing a bug in the Go compiler as a newbie: a deep dive (I)"
description: ""
summary: "The first part of this series of posts defines the issue we are going to try to fix, how I got to work on it and an initial investigation to fully understand what the bug is."
slug: optimizing-go-compiler-1
date: 2021-09-27T12:01:00-05:00
categories:
    - "go"
author: Alejandro García Montoro
github: agarciamontoro
community: alejandro.garcia
---

[bf48163e8f2b604f3b9e83951e331cd11edd8495](https://github.com/golang/go/commit/bf48163e8f2b604f3b9e83951e331cd11edd8495). That is the hash of one of the 49778 commits that, at the time of this writing, build the whole Git history of the Go compiler. It is definitely not the most important commit there, and most of the other ones probably deserve way more attention. But I like `bf48163`. It is a good commit, I think. I may be a bit biased, of course, as it is my first contribution to Go.

Not only that, it is my first contribution _ever_ to any compiler!

What's really interesting here is that I am not an expert in Go---nor in compilers in general---at all. Sure, I know my way through the language to write software using it, but I had never thought I would be able to modify the compiler itself.

And that is why I wrote this, to let you know that fixing a bug in a compiler is not only meant for a few chosen ones, but within reach of anyone.

Although this was conceived as a single post, it got a little bit out of hand, so I decided to split it in two:

-   Part 1: What the bug actually is.
-   [Part 2](/blog/optimizing-go-compiler-2): How to fix the bug.

Don't get discouraged by the length of the content: It doesn't correlate with the complexity of the issue, but with the level of detail I tried to use when explaining it. I wanted to make sure I explained the ins and outs of how I approached the fix, so I tried to discuss everything chapter and verse.

Before reading these two posts, if you know nothing about Static Single Assignment (SSA) and rewrite rules (as I did when starting with this issue), make sure to read [the intro I wrote about it](/blog/ssa-rewrite-rules), we'll be using SSA and rewrite rules intensively during this journey.

Enough with the intro, let's start!

# I'd like to work on this one!

Back in September 2020, my colleague [Agniva](https://github.com/agnivade) posted [a message in the Go channel](https://community.mattermost.com/core/pl/pd74uyf1z7rr9d46jfrww581na), where the Mattermost community hangs out to discuss all things Go, inviting people to start contributing to the compiler. He posted links to three issues in the Go project, one of which was [issue #41663](https://github.com/golang/go/issues/41663).

Agniva told that this issue was "particularly suitable for newcomers looking to get started". That's me!, I said. I am a newcomer, and I am looking to get started. After double checking with him that "newcomer" meant someone that knew nothing about compilers, I picked it up, which was as easy as writing "[I'd like to work on this one!](https://github.com/golang/go/issues/41663#issuecomment-699878893)" on the issue itself.

So that was it. I had then committed to fix an issue I knew nothing about. Of course, I started regretting that decision the very moment after I hit Send. I know now that it was a great decision, but at that time it looked quite daunting.

# The issue

Ok, so let's dive into the issue itself. The bug in the compiler can be explained with the following function:

```go
import "encoding/binary"

func f(b []byte, x *[8]byte) {
	_ = b[8]
	t := binary.LittleEndian.Uint64(x[:])
	binary.LittleEndian.PutUint64(b, t)
}
```

The report said that this function compiled to a series of small `MOVx`, but that it could be optimized to use instead a couple of `MOVQ`: one for loading and one for storing.

If this sounds as gibberish, that's ok. Bear with me and we will get to what that sentence means. First, let's see what the function is supposed to do. To make things a bit easier, let's simply rewrite the variable names and add a couple of comments:

```go
import "encoding/binary"

func copyArrayToSlice(dest []byte, src *[8]byte) {
	_ = dest[8]                                // 1. bounds check
	temp := binary.LittleEndian.Uint64(src[:]) // 2. read the contents of the source array into temp
	binary.LittleEndian.PutUint64(dest, temp)  // 3. write the contents into the destination slice
}
```

It is now more clear what the function does. It stores the contents of the second argument, the `src` array, into the first argument, the slice `dest`. It uses the [encoding/binary](https://golang.org/pkg/encoding/binary/) package for reading and writing the 8 bytes as an unsigned integer of 64 bits. For doing the copy, we use a temporary variable, `temp`.

The only weird line might be the first one, which is a bounds check. This forces the compiler to check that `dest[8]`, and, thus, all `dest[0]`, `dest[1]`, ..., `dest[7]`, are valid indices. We do that at the very beginning, so the compiler knows that all indices we are using in the copy are valid, for the whole scope of the function. We do not need to do such a check with `src`, because its length is known at compile time: 8 bytes.

So what's the bug? This is where things start to get interesting. The compiler is generating correct code, but it is not as optimized as it can be.

# The investigation

Let's study the SSA output to understand what the compiler is generating. Generating SSA for a function `foo` declared in a file `main.go` is as easy as running the command `GOSSAFUNC=foo go build main.go`, as we saw in [the SSA post](/blog/ssa-rewrite-rules/#static-single-assignment).

If we do that with the original function, `copyArrayToSlice`, and look at the `lower` phase, we'll see the following:

```
b1:

    v1 (?) = InitMem <mem>
    v7 (5) = Arg <*[8]byte> {src} (b.ptr[*byte], src[*[8]byte])
    v8 (?) = MOVQconst <int> [8] (b.cap[int], b.len[int])
    v9 (+6) = Arg <int> {dest} [8] (b.len[int], dest+8[int])
    v248 (84) = Arg <*byte> {dest}
    v153 (6) = CMPQconst <flags> [8] v9

UGT v153 → b43 b3 (likely) (6)

b3: ← b1

    v12 (6) = LoweredPanicBoundsC <mem> [0] v8 v9 v1

Exit v12 (6)

b43: ← b1

    v18 (+7) = LoweredNilCheck <void> v7 v1
    v29 (7) = InlMark <void> [0] v1
    v152 (+8) = InlMark <void> [1] v1
    v34 (+79) = MOVQload <uint64> v7 v1
    v171 (+84) = MOVBstore <mem> v248 v34 v1
    v168 (+85) = SHRQconst <uint64> [8] v34
    v216 (+89) = SHRQconst <uint64> [40] v34
    v180 (+91) = SHRQconst <uint64> [56] v34
    v219 (88) = MOVLstore <mem> [1] v248 v168 v171
    v243 (90) = MOVWstore <mem> [5] v248 v216 v219
    v255 (91) = MOVBstore <mem> [7] v248 v180 v243

Ret v255 (+8)

```

The interesting blocks are `b1`, which initialize the values with the arguments of the function, and `b43`, which makes the actual copy. The `b3` block simply handles the situation where the bounds check in the first line of our functions fails, issuing a `panic` as expected.

## Definition block

Let's take a closer look at the block where the values are initialized, `b1`:

```
v1 (?) = InitMem <mem>
v7 (5) = Arg <*[8]byte> {src} (b.ptr[*byte], src[*[8]byte])
v8 (?) = MOVQconst <int> [8] (b.cap[int], b.len[int])
v9 (+6) = Arg <int> {dest} [8] (b.len[int], dest+8[int])
v248 (84) = Arg <*byte> {dest}
v153 (6) = CMPQconst <flags> [8] v9
```

There are a couple of formalities there: the first line represents the initialized memory with the value `v1`, and the last one computes which block will be the following one to execute (the copy block or the panic block). The interesting lines are the ones in the middle, which define the parameters passed.

In particular, we see that `v7` is the second parameter of the function, `src`, whereas `v248` is a pointer to the memory of the slice `dest`, the first parameter. Although I _believe_ that `v8` and `v9` are also part of the definition of the slice (the capacity and length), there are some details that I don't completely understand, so I will not try to explain this in more detail.

For now, knowing that `v7` represents `src` and `v248` represents `dest` is more than enough!

## Copy block

Once we have our values initialized, we can study the interesting instructions, those in block `b43`. Omitting the first three lines of the original code, which are again formalities, we have the following code:

```
v34 (+79) = MOVQload <uint64> v7 v1
v171 (+84) = MOVBstore <mem> v248 v34 v1
v168 (+85) = SHRQconst <uint64> [8] v34
v216 (+89) = SHRQconst <uint64> [40] v34
v180 (+91) = SHRQconst <uint64> [56] v34
v219 (88) = MOVLstore <mem> [1] v248 v168 v171
v243 (90) = MOVWstore <mem> [5] v248 v216 v219
v255 (91) = MOVBstore <mem> [7] v248 v180 v243
```

### Loading

The first line contains our first real instruction, `MOVQload`! Now, what does that cryptic name means? The best way to know is to go directly to [its definition](https://github.com/golang/go/blob/bf48163e8f2b604f3b9e83951e331cd11edd8495/src/cmd/compile/internal/ssa/gen/AMD64Ops.go#L697):

```go
// load 8 bytes from arg0+auxint+aux. arg1=mem
{
	name: "MOVQload",
	argLength: 2,
	reg: gpload,
	asm: "MOVQ",
	aux: "SymOff",
	typ: "UInt64",
	faultOnNilArg0: true,
	symEffect: "Read",
}
```

The data in this struct defines what the `MOVQload` instruction does; the corresponding code in assembly, `MOVQ`; its returning type, `UInt64`; or the number of arguments it receives, 2. The comment does a pretty good job explaining what the instruction does: it loads 8 bytes from the data pointed to by the first argument (plus some auxiliary arguments, if present, which we'll see are constant numbers in square brackets) into the second, which should represent the memory.

If we go back to our line:

```
v34 (+79) = MOVQload <uint64> v7 v1
━━┓                           ━┓ ━┓
  ┗ we can see this            ┃  ┗ memory
    one as temp                ┗ src
```

We now can understand that this loads the contents from the `src` argument (remember that `v7`represented`src`), into the memory we initialized at the very beginning (represented by `v1`). There are no auxiliary arguments, so we can safely forget about those! The result of this instruction is represented by the `v34` value, which we can understand as the `temp` variable.

This first line of the block, then, seems perfectly fine: it loads the whole contents of the `src` array into memory (which is effectively the `temp` variable we defined in the code), and it does it with a single instruction. We can no longer optimize this.

### Storing

But remember that the original function did two things: First, it loaded the contents of the `src` array into a temporary variable, and then it stored those contents into the `dest` slice. We've already loaded the contents into memory with the line we discussed above, so the rest of the block should do the rest of the work: Store those contents from memory into the `dest` slice (which was represented by value `v248`). Let's see the rest of the block again:

```
v171 (+84) = MOVBstore <mem> v248 v34 v1
v168 (+85) = SHRQconst <uint64> [8] v34
v216 (+89) = SHRQconst <uint64> [40] v34
v180 (+91) = SHRQconst <uint64> [56] v34
v219 (88) = MOVLstore <mem> [1] v248 v168 v171
v243 (90) = MOVWstore <mem> [5] v248 v216 v219
v255 (91) = MOVBstore <mem> [7] v248 v180 v243
```

See that the values defined in the second, third, and fourth lines are used in the three last lines. Let's rewrite the block, simply replacing the values where they're used:

```
v171 (+84) = MOVBstore <mem> v248 v34 v1
v219 (88) = MOVLstore <mem> [1] v248 (SHRQconst <uint64> [8] v34) v171
v243 (90) = MOVWstore <mem> [5] v248 (SHRQconst <uint64> [40] v34) v219
v255 (91) = MOVBstore <mem> [7] v248 (SHRQconst <uint64> [56] v34) v243
```

SPOILER: These are the lines that store to the slice what we just read into memory. As you can see, we're using four instructions instead of one, which is what we're going to optimize!

But let's not get ahead of ourselves! To better understand what's going on here, we need to understand what `MOVBstore`, `MOVLstore`, and `MOVWstore` do, as well as `SHRQconst`. The first set of instructions look quite similar to the one we already understand: `MOVQload`.

We can study the first one, `MOVBstore`, as before, looking into [its definition](https://github.com/golang/go/blob/bf48163e8f2b604f3b9e83951e331cd11edd8495/src/cmd/compile/internal/ssa/gen/AMD64Ops.go#L698):

```go
// store byte in arg1 to arg0+auxint+aux. arg2=mem
{
	name: "MOVBstore",
	argLength: 3,
	reg: gpstore,
	asm: "MOVB",
	aux: "SymOff",
	typ: "Mem",
	faultOnNilArg0: true,
	symEffect: "Write",
}
```

Again, the comment does a good job explaining what's going on. The instruction receives three arguments, and it stores a single byte from the memory pointed to by the first argument (plus the auxiliary arguments) into the second argument. The state of the memory is represented by the third argument. The `typ` of the struct tells us that what this instruction returns is the new state of the memory after the operation.

In short, this instruction gets a single byte from a place in memory and stores it somewhere else.

Ok, and what about `MOVWstore` and `MOVLstore`? We can take a look at [their](https://github.com/golang/go/blob/bf48163e8f2b604f3b9e83951e331cd11edd8495/src/cmd/compile/internal/ssa/gen/AMD64Ops.go#L704) [definitions](https://github.com/golang/go/blob/bf48163e8f2b604f3b9e83951e331cd11edd8495/src/cmd/compile/internal/ssa/gen/AMD64Ops.go#L699-L700), but we can also see what those suffixes (`B`, `W`, `L` and `Q`) mean. This is explained at [the beginning of the file that contains the definition of the instructions](https://github.com/golang/go/blob/bf48163e8f2b604f3b9e83951e331cd11edd8495/src/cmd/compile/internal/ssa/gen/AMD64Ops.go#L26-L30):

```go
// Suffixes encode the bit width of various instructions.
// Q (quad word) = 64 bit
// L (long word) = 32 bit
// W (word)      = 16 bit
// B (byte)      = 8 bit
```

Knowing what we know about `MOVBstore`, it's clear what the others do:

-   `MOVBstore` stores a single **B**yte (8 bits, 1 byte)
-   `MOVWstore` stores a **W**ord (16 bits, 2 bytes)
-   `MOVLstore` stores a **L**ong word (32 bits, 4 bytes)

That's it, these operations simply store a fixed number of bytes from some place in memory to another place.

That leaves us with the last instruction: `SHRQconst`. We can do the same as before, and go straight to [its definition](https://github.com/golang/go/blob/bf48163e8f2b604f3b9e83951e331cd11edd8495/src/cmd/compile/internal/ssa/gen/AMD64Ops.go#L397) to know what it does:

```go
// unsigned arg0 >> auxint, shift amount 0-63
{
	name: "SHRQconst",
	argLength: 1,
	reg: gp11,
	asm: "SHRQ",
	aux: "Int8",
	resultInArg0: true,
	clobberFlags: true,
}
```

It's a good ol' shift to the right! That is: An operation that takes a binary number, and shifts it to the right a specific number of places, specified by the `auxint` argument (the constant integer we see in square brackets in the instruction). If shifting to the left was the same as multiplying by a power of two, as we saw in the [SSA post](/blog/ssa-rewrite-rules/#rewrite-rules), shifting a number to the right `n` places means halving it `n` times, which is equal to dividing it between `2^n`.

We have all the ingredients now:

-   `MOVXstore` stores the contents from the second argument into the first one, with the number of bytes moved depending on the prefix `X`.
-   `SHRQconst` shifts the number in its only argument a number of places specified by the auxiliary argument, the number in square brackets.

Let's see the lines again, one by one. The first one was:

```
v171 (+84) = MOVBstore <mem> v248 v34 v1
```

This one is straightforward: it stores a single **B**yte from `v34` (the contents in `temp`) into the memory pointed to by `v248` (which is `dest`). That is, we are copying `temp[0]` into `dest[0]`.

The second one is more interesting, let's break it down:

```
                   this L means 4 bytes        ┏ dest               temp ┓
                   ━━━━━━━━┳━━━━━━━━━━━        ┃                         ┃
                           ┃                ━━━┛                         ┗━━
     new                                                                               previous
state of ━┫ v219 (88) = MOVLstore <mem> [1] v248 (SHRQconst <uint64> [8] v34) v171 ┣━━ state of
  memory                                                                               memory
                                        ━┳━                          ━┳━
                                         ┃                            ┃
         ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓      ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
           auxiliary argument to MOVLstore, so it           auxiliary argument to SHRQconst, so
           starts writing data one byte after the           it shifts the contents 8 bits to the
           memory pointed to by the second argument         right, effectively discarding them
         ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛      ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
```

Now we're storing a **L**ong word (4 bytes) from the second argument, `(SHRQconst <uint64> [8] v34)`, into the memory pointed to by `v248`, which is `dest`. But we have the `[1]` auxiliary argument! So we don't write directly into the memory pointed to by `v248`, but one byte after that pointer: That is, the first byte of `dest` is left untouched, and we write contents from the second byte on. In terms of Go code, as we're storing 4 bytes, it means that we're writing to `dest[1]`, `dest[2]`, `dest[3]` and `dest[4]`.

But what 4 bytes do we write? That's what `(SHRQconst <uint64> [8] v34)` is telling us: We already wrote `temp[0]` in `dest[0]`, so it would make sense to read now from `temp[1]` on. And that's exactly what we're doing with the shift: As the auxiliary argument is `[8]`, that means that we're shifting the content of `v34` (or `temp`) to the right 8 bits. Those 8 bits are already in `dest[0]`, so we can safely discard them and only read the next ones: from `temp[1]` on. As `MOVLstore` is writing 4 bytes, that means that we're getting 4 bytes' worth of data; that is: `temp[1]`, `temp[2]`, `temp[3]`, and `temp[4]`.

So that's it! That long, complex, full of arguments and auxiliary arguments, instruction is simply doing this:

```go
dest[1] = temp[1]
dest[2] = temp[2]
dest[3] = temp[3]
dest[4] = temp[4]
```

That was a bit hard, I agree. But we already have everything we need to know, because the two following lines are structurally identical!

```
v243 (90) = MOVWstore <mem> [5] v248 (SHRQconst <uint64> [40] v34) v219
```

This one stores 2 bytes (see the `W` prefix there?) into the memory pointed to by `v248` plus `[5]` bytes; that is, we write into `dest[5]` and `dest[6]`. And what we write is the content of `v34` (`temp`) shifted to the right `[40]` bits (or 5 bytes, as we've already copied from `temp[0]` to `temp[4]`). That's right, this line is doing the following:

```go
dest[5] = temp[5]
dest[6] = temp[6]
```

So this leaves us with the last line of the block! And you already know what will happen:

```
v255 (91) = MOVBstore <mem> [7] v248 (SHRQconst <uint64> [56] v34) v243
```

Yup, we're simply copying the last byte from `temp` (skipping the first `[56]` bits, or 7 bytes!), into the `dest` slice (starting to write after `[7]` bytes, or 56 bits!). That's it, in terms of Go code, we're simply doing this:

```go
dst[7] = temp[7]
```

Wrapping up everything we've learned up until now, we know now that we're doing the following:

```go
dst[0] = temp[0]  // v171 (+84) = MOVBstore <mem> v248 v34 v1

dst[1] = temp[1]  // v219 (88) = MOVLstore <mem> [1] v248 (SHRQconst <uint64> [8] v34) v171
dst[2] = temp[2]  //
dst[3] = temp[3]  //
dst[4] = temp[4]  //

dst[5] = temp[5]  // v243 (90) = MOVWstore <mem> [5] v248 (SHRQconst <uint64> [40] v34) v219
dst[6] = temp[6]  //

dst[7] = temp[7]  // v255 (91) = MOVBstore <mem> [7] v248 (SHRQconst <uint64> [56] v34) v243
```

Cool, we did it! We understood what a whole block of SSA is doing, so feel free to treat yourself with a snack before going on.

# Next steps

Now that we understand what the compiler is doing with our original function, we can plan how to fix it. In [the next and last part of the series](/blog/optimizing-go-compiler-2) we will see how to convert that beast of 4 instructions---which store a byte, then 4 more, then another 2, and finally one more ¯\\\_(ツ)\_/¯---into a single one that directly stores the 8 bytes.
