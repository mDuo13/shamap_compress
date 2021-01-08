# Saving Memory In rippled 1.7

The rippled 1.7 release has several patches that reduce memory consumption by
more than 50%. This blog post describes one way we achieved this savings.

The idea is very simple: take a tree that looks like this:

![Dense Tree](/dense_tree.png)

And change it to a tree that looks like this:

![Sparse Tree](/sparse_tree.png)

The tree can hold up to 16 children, but often holds fewer. All those small,
empty boxes are wasted space. By changing the way the tree stores its children
the number of empty boxes can be reduced, saving space.

The data structure used to keep ledger state and transactions in main memory is
stored in such a tree. A rippled server will keep millions of these tree nodes
in memory. Even small savings in node size is worthwhile when multiplied by the
large number of nodes.


# SHAMap

A SHAMap is a tree-like data structure where keys are 256-bit hashes and a node
in the tree can have up to 16 children. Children are always stored as a 256-bit
hash value that can be used to retrieved the child node from a database.
Children are also sometimes cached directly in a node for quick traversal. A
simplified class that represented a non-leaf node might look like this:

```c++
struct SHAMapInnerNode {
    std::bitset<16> isBranch_;
    SHAMapHash hashes_[16];
    std::shared_ptr<SHAMapTreeNode> cachedChildren_[16];
}
```

The tree is traversed by looking at the key four bits at a time. These four bit
number represents which of the sixteen children to transition to next. This
means the children are ordered and it's possible that slots 1, 5, and 7 are
occupied while the rest are empty. `isBranch_` tracks which children are present
and which are empty.

Since slots can be empty, that brings up an interesting question: in a typical
SHAMap, how many children does a typical inner node have?

Measurements from a rippled validator showed the following distribution:

![SHAMap Child Histogram](/shamap_child_histogram.png)

The data above shows that most inner nodes only have a handful of children.<sup>[1](#histogram_bug)</sup>

# Sparse SHAMap

Instead of always storing sixteen children, what if a node used a space-saving
sparse representation when there are only a few children and the usual dense
representation when there are more children?

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
gaps. For example, if children 1, 5, and 7 are occupied while the rest are
empty, then `hashes_` would store these values at indexes 0, 1, and 2. To look
up a child in a sparse array, count the number children present that have a
smaller index. With the example above (1, 5, and 7 set), there is 1 child with
an index less than 5, so child 5 is stored in array index 1.

How much memory does this save? Each hash is 32 bytes, and each shared pointer
is 16 bytes. That's 48 bytes saved for every child that isn't stored in a sparse
representation that would have been stored in a dense representation.

Given the data collected above, it's easy to calculate how much memory would be
saved if we used a sparse representation.

```python
# Given a histogram of the number of children, and the array size boundaries, 
# calculate the ratio of memory used for a sparse representation/memory used 
# for a dense representation
def savings(child_hist, arg_indexes):
    if not arg_indexes:
        return 1.0
    indexes = [a+1 for a in arg_indexes] # add one (and copy so we don't modify original list)
    if indexes[0] != 0:
        indexes.insert(0, 0)
    if indexes[-1] != 17:
        indexes.append(17)
    per_child_bytes = 48.0 # 32 for the hash 16 for the shared_ptr
    dense_bytes = 16.0*per_child_bytes*sum(child_hist)
    sparse_bytes = 0.0
    for i in range(1, len(indexes)):
        sparse_bytes += (indexes[i]-1)*per_child_bytes*sum(child_hist[indexes[i-1]:indexes[i]])
    return (sparse_bytes/dense_bytes)
```

For example, if a sparse array were used when there are 5 or fewer children, it
would take about 38% of the of the original memory to store the children.

If there are only two children, and our sparse array has room for 5 children,
there is still wasted space. Instead of using just 2 array sizes: dense (16) and
sparse (5), we could use 3 sizes, say 3, 6, and 16. If we used 16 sparse array
sizes, one for every possible number of children, then no space would be wasted
at all. There are reasons not to do that. The trade-offs considered when
choosing how many sparse array sized to use were:

1. Costs when moving data when the sparse array changes representations. I.e.,
   when a child is added and there is no more space in the current sparse array,
   a new array must be allocated and the data moved into it.
   
2. Costs when inserting and removing data in the sparse. The sparse arrays are
   sorted by child index. This allows for a space efficient representation, but
   requires moving elements to keep the array sorted. Keeping sparse arrays
   small reduces these data movements.

3. Space savings. The more sparse array sizes, the greater the savings. However,
   data shows there usually diminishing returns in adding one more array size.

4. Implementation considerations. A tagged pointer (described below) can be used
   to store small number bits without additional space. Keeping the number of
   array sizes below this number results in additional savings.

A good data point to help resolve these trade-offs is to look at the space
savings vs. number of sparse arrays, and the sparse array sizes. The following
code helps compute this.

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

Most of the savings occur my having one sparse array of size 5 and one dense
array. There are some additional savings if we allow 2 or 3 sizes, after that
adding more sizes doesn't make much difference. The following plot helps to see
this:

![SHAMap Savings Vs. Dims](/shamap_savings_vs_dims.png)

Considering the diminishing returns of more sparse array sizes, only 3 sparse
sizes (2, 4, and 6) and one dense size (16) is used in the implementation. It's
also convenient that these four sizes can be encoded as two bits in a tagged
pointer (described below).

Just using one sparse size and one dense size was considered. This still gives a
large space savings, and the implementation is simpler to understand. However,
it was decided the extra space savings was worth the extra implementation
complexity. 

Development note: the "one sparse array" implementation was still coded up first,
even though we knew it was never going to be used. It's often better to code up
and test these intermediate milestones rather than implement the final complex
version right away.

# Additional space savings

When the space savings was calculated above, the "48 bytes" saved isn't correct.
There are several ways sparse arrays increased the size: 

1. Storing pointers to arrays instead of arrays directly.
2. Storing the array sizes.
3. Overhead from malloc.

The following sections addresses how to reduce this newly added space.

## A note on software complexity

The following changes are a good illustration for how software ends up complex.
The actual core idea is relative simple, and there's a large space savings to be
had just by having two representations and storing the pointers and sizes
explicitly. However, since we were focusing on saving ram, we implemented other
changes that saved some extra space as well. While these memory saving technique
save RAM, they also make the code more difficult to understand and debug. The
team is very aware of the importance of simple implementations. However, given
how many inner nodes are in the system, the team opted to implement these
smaller space savings as well.


## One Opaque Pointer

It takes 8 bytes to store a pointer, and we need to store two of them. One way
to save this space is to use a single opaque pointer, and place the
`cachedChildren_` array immediately after the `hashes_` array. For example, an
implementation could be:

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

This saves 8 bytes per node.

## Tagged Pointer

A new one byte value `arraySize_` is now stored. We could get rid of this by
always counting bits in `isBranch_`, but this has one minor and one serious
drawback. The minor drawback is it's more compute intensive (with a `popcount`
instruction, this would be negligible). The serious drawback is it forces
children to be stored in the smallest possible sparse array available. This
isn't always desirable. Sometimes it's better to temporarily keep a denser
representation. For example, a fully dense representation might be temporally
used while batch adding and removing many children. Or hysteresis may be used to
reduce how often arrays need to be copied.

Some bit values in pointers always have the same value. For example, if a data
structure must be aligned on an eight byte boundary, the last three bits must
always be zero. Since there are four array sizes (2, 4, 6, and 16), these values
can be encoded as the last two bits of the pointer.

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

(Note that the actual implementation in rippled wraps this pointer into a `TaggedPointer` class).

This saves 8 bytes per node.

## Do not use `std::bitset`

While a `std::bitset<16>` could be represented with two bytes, on my system
`sizeof(std::bitset<16>)` A `std::uint16_t` is used to represent this bitset and
saves 6 bytes over a bitset.<sup>[2](#no_bitset)</sup>

## Custom allocators

There's typically an 8 to 16 byte space overhead when `malloc` is used to get
memory from the heap. `malloc` is also a relatively slow operation.

Since the SHAMap is allocating lots of these inner nodes, and they all have the
same size (or rather four sizes, but we can easily use four allocators), this is
a good use-case for custom allocators. C++-17 introduced polymorphic memory
resources. The new pool allocator is a good match here: it promises fast
allocation and deallocation and has zero extra space overhead.

Unfortunately, clang's standard library does not yet have an implementation for
this, which would be needed on macs. Additionally, a microbenchmark showed gcc's
`std::synchronized_pool_resource` was much too slow <sup>[3](#gcc_pmr)</sup>,
and resulted in a significant performance regression in rippled.

Fortunately, boost has an implementation of pooled resources that is significantly
faster. It fixed this performance regression, and also work well on macs.

As a side note, one of my colleagues (Howard Hinnant) is looking into why gcc's
implementation gave such poor results and will report this upstream. Another
colleague (Nik Bougalis) is working on a custom allocator for this that promises
even better performance.

## Bit twiddling

A child's index in a sparse array is equal to the number of non-empty children
that precede it. This can be done cheaply by masking out all the children >= to
that index in the `isBranch_` bitset, and then counting the set bits. The
following code does this quickly (popcnt16 can be a builtin assembly
instruction).


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

# Conclusion and Measurements

This blog post describes just one way rippled 1.7 saves memory. Other patches in
the 1.7 also focused on saving memory. One such patch focused on removing a
cache (in such a way that _improved_ execution time). The following chart
compares the memory usage of an unpatched rippled server vs. a server running
just the "sparse inner node" patch described here vs. a server with both the
"sparse inner node" and "rm cache" patches.

![b7 memory usage](/b7_memory_usage.png)

![b7 memory savings](/b7_memory_savings.png)

It's easy to see the SHAMapInnerNode patch results in significant savings, and
"rm cache" patch results in even greater. The two combined results in over 50%
memory savings.

Also, this tweet made me very happy:
https://twitter.com/alloynetworks/status/1343639441898954754?s=20

# Footnotes
<a name="histogram_bug">1</a>: There shouldn't be any nodes with just one child,
so there appears to be a bug in the way this data was collected.

<a name="no_bitset">2</a>: The rippled implementation never used a bitset.

<a name="gcc_pmr">3</a>: Only gcc's implementation of pooled allocators were
benchmarked.
