# How Ripple's C++ Team Cut the XRP Ledger's Memory Footprint Down to Size

One of the best ways to make software more accessible is to reduce the hardware
resources needed to run it. The peer-to-peer server that powers the XRP Ledger
is no exception. This server is the bedrock of the XRP Ledger, which is already
[one of the greenest blockchains](https://xrpl.org/carbon-calculator.html)
due to its unique consensus protocol, but that's no reason to get complacent:
more efficient resource usage is always helpful to the digital assets ecosystem—
and to natural ecosystems, too.

Making better use of the computing resources means you can participate in the
XRP Ledger network with cheaper hardware and fewer upgrades. If you have a big
workload, maybe you can split it between fewer machines in your server farm,
saving you electricity. If you're just getting started, you can try stuff out
without buying the latest and greatest workstation you can find. And if you're
already using the XRP Ledger just fine, you'll be ready to handle larger
volumes of transactions just by upgrading the software.

This is all great in the abstract, but how achievable are such gains, really?
The C++ team at Ripple spent a significant amount of time in 2020 focusing on
how to make better use of available system resources, and we're proud to report
that the improvements we've built into the upcoming 1.7.0 version of `rippled`,
our reference implementation of the XRP Ledger server, have cut its memory
usage to _less than half_ of what it used to require.

That's not a typo and these aren't theoretical numbers. These are _real-world_
improvements of more than 50%, in servers on the XRP Ledger mainnet. The
operator of <alloy.ee> calls it "sheer voodoo":

<blockquote class="twitter-tweet">
    <p lang="en" dir="ltr">1.7b9 is sheer voodoo - The higher one is 1.6.0
        <a href="https://t.co/NOlqAmpvBU">pic.twitter.com/NOlqAmpvBU</a>
    </p>
    &mdash; Alloy Networks (@alloynetworks) <a href="https://twitter.com/alloynetworks/status/1341495806893973506">December 22, 2020</a>
</blockquote>

If you're curious how these technical marvels came to be, read on for all the
details of the engineering work that made it possible.


## Motivation

Aside from the reasons above, Ripple has a particular interest in minimizing
the resources that XRP Ledger servers require, because we as a company run
several widely-used clusters of them. So, alongside our other experiments and
work on furthering the XRP Ledger ecosystem, we set out to measure and
understand how `rippled` uses resources and how we could improve it.

Back in [version 1.6.0](https://xrpl.org/blog/2020/rippled-1.6.0.html), we
added support for link compression, which helps reduce the amount of bandwidth
used by servers, benefiting operators of single servers and large clusters
alike. We've also been working on optimizing when the server does or doesn't
relay messages throughout the XRP Ledger network, so that every server hears
everything it needs to know without getting redundant copies of the same info.

However, the primary pain-point for operators of `rippled` has been RAM. Memory
is a critical and scarce resource. We had a strong suspicion that the existing
code was not using it as judiciously and effectively as it could or should, so
we focused on finding the largest consumers and seeing what we could do about
them.


## Understanding `SHAMap`

In `rippled`, one of the largest targets was the `SHAMap` and
its constituent parts—specifically, the nodes within the `SHAMap`'s tree-like
structure. This data structure holds the state of the ledger
itself—all the accounts, balances, settings, exchange orders, and other
data that make up the latest state of the XRP Ledger. On the XRP Ledger Mainnet,
the `SHAMap` consists of millions of nodes, which means that even small savings
in the size of individual nodes can result in large savings overall.

More precisely the `SHAMap` is a combination of a [Merkle tree](https://en.wikipedia.org/wiki/Merkle_tree)
and a [Radix tree](https://en.wikipedia.org/wiki/Radix_tree) with a branching
factor of 16. This branching factor is especially relevant in this instance. It
means that the _inner nodes_ in the tree (the nodes other than the "leaves")
can have _up to_ 16 children. Consider, for example, this simple tree:

![Dense Tree](/dense_tree.png)

Each node has 16 children, whether they point to something or not. And that was
the key insight: that nodes can have _up to_ 16 children doesn't mean that they
actually do in practice.

The question then became, how many children does a "typical" inner node have?
We gathered data from Ripple-operated servers, which resulted in the following
distribution:

![SHAMap Child Histogram](/shamap_child_histogram.png)

The data we collected shows that most inner nodes only have a handful of
children.[¹](#histogram_bug) With that in mind, the next question was: what if
the nodes of the tree could be adjusted, at runtime, to accommodate only as
many children as they needed?

In other words, what if we changed the tree to look more like this?

![Sparse Tree](/sparse_tree.png)

### The Naive `SHAMap` Design

Any `SHAMap` is a tree-like data structure where keys are 256-bit hashes and a
node in the tree (other than the leaves) can have up to 16 children. Children
are always stored as a 256-bit hash value that can be used to retrieved the
child node from a database. Children are also sometimes cached directly in a
node for quick traversal. A simplified class that represents a non-leaf node
might look like this:

```c++
struct SHAMapInnerNode {
    std::bitset<16> isBranch_;
    SHAMapHash hashes_[16];
    std::shared_ptr<SHAMapTreeNode> cachedChildren_[16];
}
```

To traverse the tree, the code looks at the 256-bit key four bits at a time.
These 4-bit numbers represent which of the sixteen children to transition to
next. This means the children are ordered, and it's possible that some branches
(such as 2, 5, and 7) are occupied while the rest are empty. The `isBranch_`
variable tracks which branches have children and which are empty,
but the `hashes_[16]` array always leaves space for sixteen 256-bit identifiers
anyway. The XRP Ledger's `SHAMap` implementation was something like this to
begin with.

### Sparse SHAMap

Instead of always storing sixteen children, we want to use a space-saving sparse
representation when there are only a few children, and only switch to the dense
representation when there are more children.

Such a representation could look like this:

```c++
struct SHAMapInnerNode {
    std::bitset<16> isBranch_;
    SHAMapHash* hashes_;
    std::shared_ptr<SHAMapTreeNode>* cachedChildren_;
    std::uint8_t arraySize_;
}
```

A sparse representation would store `hashes_` and `cachedChildren_` without
gaps. For example, if branches 2, 5, and 7 are occupied while the rest are
empty, then `hashes_` would store these values at indexes 0, 1, and 2. To look
up a child in a sparse array, count how many non-empty branches come before it.
So, if branches 2, 5, and 7 are occupied, and you want to the child in branch 5,
there's 1 branch with an index less than 5, you'd look at array index 1 of the
`hashes_` array.

How much memory does this save? Each hash is 256 bits—that's 32 bytes—and each
shared pointer is 16 bytes. That means that every time the sparse
representation doesn't leave empty space for a nonexistent child that the
naive, dense implementation would have, you've just saved 48 bytes.

With the data we had collected on how many children the nodes in the `SHAMap`
really had when used in production, we could calculate how much memory could be
saved by using a sparse representation.

The following Python code demonstrates how to perform such a calculation:

```python

def savings(child_hist, arg_indexes):
    """
    Given a histogram of the number of children, and the array size boundaries,
    calculate the ratio of memory used for a sparse representation/memory used
    for a dense representation
    """
    if not arg_indexes:
        return 1.0
    # add one (and copy so we don't modify original list)
    indexes = [a+1 for a in arg_indexes]
    if indexes[0] != 0:
        indexes.insert(0, 0)
    if indexes[-1] != 17:
        indexes.append(17)
    per_child_bytes = 48.0 # 32 for the hash 16 for the shared_ptr
    dense_bytes = 16.0 * per_child_bytes * sum(child_hist)
    sparse_bytes = 0.0
    for i in range(1, len(indexes)):
        sparse_bytes += ((indexes[i]-1) *
                         per_child_bytes *
                         sum(child_hist[indexes[i-1]:indexes[i]])
                        )
    return (sparse_bytes/dense_bytes)
```

## How Sparse to Go?

It was clear that a sparse representation could save a substantial amount of
memory, but the next challenge was: how sparse to go? If we used 16 different
sparse array sizes, one for every possible number of children, then no space
would be wasted at all, but doing that brings in other costs. For example, when
you have a size-10 array and you need to add an 11th child, you need to allocate
space for a size-11 array, and copy all 10 existing entries into the new array.

Another approach is to have just two sizes: for example, you might use a sparse array
with room for up to 5 children, and a dense array when you need to hold 6 to 16.
This approach still wastes some space—for example, when the sparse array has
just 1 of its 5 spots occupied, or when the dense array uses only 6 or 7 of its
16 spots—but it's simpler and faster than constantly changing the size of the
array. Alternatively, we could have different 3 sizes—say, 3, 6, and 16—or more.

To decide between all the possibilities, we had to consider the trade-offs of
different numbers of sparse array sizes. This kind of analysis is common in
engineering and even in other disciplines; it's kind of like deciding how many
sizes of jeans you want to sell to make the most profit. In our case, we
considered all of the following in deciding how to approach the problem:

1. Costs when moving data when the sparse array changes representations. If you
   need to add a child and the current sparse array is full, you need to
   allocate a new array and move the data into it.

2. Costs of inserting and removing data in the sparse array. Since the sparse
   arrays are sorted, you have to move elements around whenever you add or
   remove children from them. Keeping the sparse arrays small reduces the
   amount of data movement necessary.

3. Space savings. The more sparse array sizes, the greater the savings. However,
   we expected diminishing returns from each additional size option.

4. Implementation considerations. A tagged pointer (described below) can be used
   to store a small number of bits without using additional space. Keeping the
   number of array sizes below this number results in additional savings.

Fortunately, we didn't have to make these decisions based on pure intuition and
generalizations. With our histogram data, we could calculate the actual space
savings of various numbers and sizes of sparse arrays, to figure out where the
diminishing returns really kicked in. The following code helps compute this.

```python
def savings_dim(dim=1):
    if dim == 0:
        s = savings(collected_child_histogram, [])
        return ([s], [()], 0)
    args = [arg for arg in itertools.combinations(range(17),dim)]
    r = [savings(collected_child_histogram, arg) for arg in args]
    return (r, args, numpy.argmin(r))


def doit():
    result = []
    for dim in range(17):
        r, args, arg_min = savings_dim(dim)
        result.append([r[arg_min], args[arg_min]])
    return result
```

This gives the following result:
```python
[[1.000, ()],
 [0.384, (5,)],
 [0.293, (3, 6)],
 [0.259, (2, 4, 6)],
 [0.246, (2, 3, 5, 7)],
 [0.237, (2, 3, 4, 5, 7)],
 [0.233, (1, 2, 3, 4, 5, 7)],
 [0.230, (1, 2, 3, 4, 5, 6, 8)],
 [0.228, (1, 2, 3, 4, 5, 6, 8, 15)],
 [0.228, (1, 2, 3, 4, 5, 6, 7, 9, 15)],
 [0.227, (1, 2, 3, 4, 5, 6, 7, 9, 14, 15)],
 [0.227, (1, 2, 3, 4, 5, 6, 7, 8, 9, 14, 15)],
 [0.227, (1, 2, 3, 4, 5, 6, 7, 8, 9, 13, 14, 15)],
 [0.227, (1, 2, 3, 4, 5, 6, 7, 8, 9, 11, 13, 14, 15)],
 [0.227, (1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 13, 14, 15)],
 [0.227, (1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15)],
 [0.227, (0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15)]]
```

The first row establishes the baseline, with no sparse arrays, just the naive
`SHAMap` design. In other words, we defined "however much memory it was using
originally" as `1.000` or 100%.

The next row represents an approach using one sparse array of size 5 and one
dense array (`(5,)`), and estimates that this would use 38.4% as much space as
the naive design (`0.384`). The third row tests an approach with two sparse
arrays of sizes 3 and 6, which occupies 29.3% as much space, and so on.

Most of the savings occur by having one sparse array of size 5 and one dense
array. There are some additional savings with a 2 or 3 different sparse arrays
sizes, and anything beyond that doesn't make much difference. You can see the
diminishing returns more clearly if we plot this as a graph:

![SHAMap Savings Vs. Dims](/shamap_savings_vs_dims.png)

With this information, we knew we wanted to use either one, two, or three
different sizes of sparse arrays. It was tempting to go with just one sparse
and one dense size, since that already gives us most of the space savings and
the implementation is simpler to understand. However, we decided that the
extra savings of going a little bit farther were worth the slight increase of
complexity.

In the end we opted to use three sparse array sizes (2, 4, and 6) plus the
dense array (16). Conveniently, we can use two bits of a tagged pointer to
encode these four sizes in a convenient, compact way. (More on this later.)

Development note: We actually coded up the "one sparse array" implementation
first, even though we knew it was never going to be used. It's often better to
build and test an intermediate milestone like this rather than implement the
final, complex version right away.

## Further Optimizations

Earlier, when we calculated "48 bytes" of space savings per unused node, that
was a bit of an idealized scenario. In practice, the sparse array design has
several new details that reduce the savings from the potential maximum:

1. Storing pointers to arrays instead of arrays directly.
2. Storing the array sizes.
3. Overhead from `malloc`.

What happened next is a good illustration for how software ends up complex.

The actual core idea of using sparse arrays is relatively simple, and you can
save a lot of space by implementing that in the most straightforward way
possible. If you want to get even more efficient, there are various tricks you
can use to squeeze things into just a little bit less space. These techniques
end up saving RAM, but they also make the code more difficult to understand and
debug. The team is very aware of the long-term costs this imposes on
maintaining the software.

Blindly trying to maximize the efficiency of every piece of code is a "penny
wise, pound foolish" move. You may save an imperceptible amount of RAM or CPU
only to have an update later break the code because someone misunderstood it,
or the people who have to upkeep the software later might just have to spend a
lot of brainpower to prevent that from happening, when they could have spent
that energy elsewhere if you hadn't tried to optimize as much.

But on the other hand, some things _are_ worth optimizing. In the case of the
`SHAMap` nodes, even small improvements to individual nodes are multiplied
millions of times over because there are so many nodes in the overall
structure. Given this, we opted to implement the smaller space savings as well.


### One Opaque Pointer

It takes 8 bytes to store a pointer, and we need to store pointers to two
objects: the array of child hashes, and array of cached child objects. But, we
know exactly how long the `hashes_` array will be, and it happens that the
hashes are 32 bytes so the end of the array will always be cleanly aligned on
an 8-byte boundary in memory. If we always store the `cachedChildren_` array
directly following the `hashes_` array, we can use one pointer to refer to both
of them. This saves 8 bytes for every inner node.

For example, an implementation could be:

```c++
struct SHAMapInnerNode {
    std::bitset<16> isBranch_;
    void* hashesAndCachedChildren_;
    std::uint8_t arraySize_;

    SHAMapHash* hashes(){
        return reinterpret_cast<SHAMapHash*>(hashesAndCachedChildren_);
    }
    std::shared_ptr<SHAMapTreeNode>* cachedChildren(){
        // since hashes are 32 bytes, this will always be correctly aligned
        return reinterpret_cast<SHAMapHash*>(hashesAndCachedChildren_ + arraySize_);
    }
}
```


### Storing Array Size

One of the added pieces of information in the sparse array is the field that
indicates what size array a given node has: 2, 4, 6, or the full 16. If you
recall, our example design for the sparse `SHAMapInnerNode` stored this as a
1-byte value, `arraySize_`. We wanted to avoid storing this value, too.

#### Approach 1: Counting Occupied Branches

If you look over the sparse `SHAMapInnerNode`'s other fields, you might notice
that the `isBranch_` field can pull double duty here: if you look through how
many of its bits are on, you know how many children the node has, and you can
use that to deduce what size array the node is using. This involves slightly
more computing, since the code has to count bits each time you check the node,
but that's negligible if you have a [`popcount` instruction](https://en.cppreference.com/w/cpp/numeric/popcount).

Counting bits in `isBranch_` has a serious drawback, though: it only works if
every node _always_ stores children in the smallest sparse array possible. This
isn't always desirable. Sometimes it's better to temporarily keep a denser
representation. For example, if you're batch adding or removing many child
nodes, it would make sense to set aside a dense array from the start rather
than resize the same node several times in a short period. Alternatively, you
might want to reduce how often arrays need to be copied by using a technique
such as hysteresis or some other predictive heuristic.

Since we wanted to keep these options open, we decided against using
`isBranch_` to determine the array size.

#### Approach 2: Tagged Pointer

Pointers, like the one we use to refer to the array of child hashes, are the
addresses of data structures in memory. On a 64-bit system, a pointer is always  
64 bits (8 bytes) so that it can point to any address in memory. However, the
operating system doesn't allocate structures into literally any address in
memory; it generally aligns things along nice even boundaries of 8 bytes. This
means that, in practice, the smallest 3 bits of a pointer are always going to
be 0's.

The idea of a "tagged pointer" is to use those bits to store some other useful
data. Since we have only four sizes of array (2, 4, 6, and 16), we can encode
the size of the array into just two bits. We then "fold" this information into
the pointer, and pull the separate pieces apart using bit masks when we need to
use them.

One implementation of this could be:

```c++
constexpr std::array<std::uint8_t, 4> boundaries{
    2,
    4,
    6,
    16};

struct SHAMapInnerNode {
    static_assert(
        alignof(SHAMapHash) >= 4,
        "Bad alignment: Tag pointer requires low two bits to be zero.");
    /** Upper bits are the pointer, lowest two bits are the tag
        A moved-from object will have a tp_ of zero.
    */
    std::uintptr_t tp_ = 0;
    /** bit-and with this mask to get the tag bits (lowest two bits) */
    static constexpr std::uintptr_t tagMask = 3;
    /** bit-and with this mask to get the pointer bits (mask out the tag) */
    static constexpr std::uintptr_t ptrMask = ~tagMask;

    // decode the tagged pointer into the tag and pointer
    [[nodiscard]] std::pair<std::uint8_t, void*>
    TaggedPointer::decode() const
    {
        return {tp_ & tagMask, reinterpret_cast<void*>(tp_ & ptrMask)};
    }

    // return array size and the two arrays
    [[nodiscard]]
        std::tuple<std::uint8_t, SHAMapHash*, std::shared_ptr<SHAMapTreeNode>*>
        TaggedPointer::getHashesAndChildren() const
    {
        auto const [tag, ptr] = decode();
        auto const hashes = reinterpret_cast<SHAMapHash*>(ptr);
        std::uint8_t numAllocated = boundaries[tag];
        auto const children = reinterpret_cast<std::shared_ptr<SHAMapTreeNode>*>(
            hashes + numAllocated);
        return {numAllocated, hashes, children};
    };
}
```

The actual implementation in `rippled` wraps this pointer into a
`TaggedPointer` class.

In the end, we were able to store the size of the array into some bits that
were not in use, instead of adding 8 bytes per node for an `arraySize_` field.


### Do not use `std::bitset`

We still have the `isBranch_` field to indicate which branches actually contain
children and which ones are empty. The smallest possible representation of this
field is two bytes (16 bits), which is theoretically what `std::bitset<16>`
represents. However, in practice, most implementations of the C++ standard
library don't go to extremes to save space, so when you ask for a
`std::bitset<16>` you often get 32 or even 64 bits, depending on the machine.
Those extra 2 to 8 bytes can add up when there are millions of instances of the
same data structure.

Instead, we use a `std::uint16_t`, which is more likely to be allocated as
actually 16 bits, or at least closer to it. Depending on the platform, this can
end up saving as much as 6 bytes for every inner node. For this reason, the
`rippled` code never uses the bitset type.


### Custom allocators

There's typically an 8 to 16 byte space overhead when `malloc` is used to get
memory from the heap. `malloc` is also a relatively slow operation.

Since the `SHAMap` is allocating lots of these inner nodes, and they all have
the same size (or rather four sizes, but we can easily use four allocators),
this is a good use-case for custom allocators. C++17 introduced polymorphic
memory resources. The new pool allocator is a good match here: it promises fast
allocation and deallocation and has zero extra space overhead.

Unfortunately, on macOS, the Clang compiler's standard library does not yet
have an implementation for this. Additionally, a microbenchmark showed that the
`gcc` compiler's `std::synchronized_pool_resource` was much too slow[²](#gcc_pmr),
and resulted in a significant performance regression in `rippled`.

Fortunately, Boost has an implementation of pooled resources that is
significantly faster. It fixed this performance regression, and also works on
macOS.

Our team is already looking into why `gcc`'s implementation gave such poor
results. If we identify the root cause we will submit code to fix the problem.
Furthermore, we are also working on a custom slab allocator with thread-aware
freelist caching that promises even better performance.


### Bit twiddling

One of the most common operations you need to do on a sparse array is to figure
out where in the sparse array a given branch is stored. The example we used
earlier was that if branches 2, 5, and 7 are occupied, and we are looking for
a child in branch 5, we count how many children are occupied below that (just
one, branch 2) and use that as the index into the sparse array.

We wanted to make sure that this operation was very fast so that the sparse
arrays performed well. We realized that we could use bitwise operations on the
`isBranch_` field to perform this calculation in just a couple steps: first,
mask out all the children greater than the index you're looking for; then,
count how many bits are still on. The [`popcnt16` instruction](https://en.wikipedia.org/wiki/Hamming_weight#Processor_support)
is built in on modern CPU architectures, so the entire calculation takes just a
few processor instructions to run.

```c++
inline int
TaggedPointer::getChildIndex(std::uint16_t isBranch, int i) const
{
    // Sparse children are stored sorted. This means the index
    // of a child in the array is the number of non-empty children
    // before it. Since `isBranch_` is a bitset of the stored
    // children, we simply need to mask out (and set to zero) all
    // the bits in `isBranch_` equal to to higher than `i` and count
    // the bits.

    // mask sets all the bits >=i to zero and all the bits <i to
    // one.
    auto const mask = (1 << i) - 1;
    return popcnt16(isBranch & mask);
}
```

## Oh... one more thing.

Our CTO, [David Schwartz](https://twitter.com/joelkatz), was inspired to go one
step further, examining some of the other heavy users of memory. He implemented
a change that eliminates a layer of caching, which resulted in further memory
improvements and, surprisingly, _improved_ execution time.

## Real-world Impact

The following chart compares the memory usage of three versions of the
`rippled` server:

- The first (blue) is unpatched, running the previous version of the code
- The second (orange) has the "sparse inner node" changes described here
- The third (green) has both the "sparse inner node" changes and the
  improvements added by David Schwartz:

![b7 memory usage](/b7_memory_usage.png)

![b7 memory savings](/b7_memory_savings.png)

It's easy to see the significant impact these two changes have on memory usage.
Combined, they result in total memory savings of over 50%.

## Conclusions

It's often lamented that as computer hardware gets better, software only gets
worse to compensate. While there is a grain of truth to that statement, at
Ripple we believe it doesn't have to be that way. Ironically, one of the
biggest changes that may come out of this work to reduce the memory usage of
the XRP Ledger is _no change_: with these improvements in place, we won't have
to increase our recommended system specs to run `rippled`, even though the
number of accounts and data stored in the XRP Ledger continue to grow over
time.

We look forward to continuing to help the XRP Ledger grow to be the best, most
efficient, greenest system out there for payments and more.


## Footnotes
<a id="histogram_bug">1</a>: There shouldn't be any nodes with just one child,
so there appears to be a bug in the way this data was collected.

<a id="gcc_pmr">2</a>: We only benchmarked `gcc`'s implementation of pooled
allocators.
