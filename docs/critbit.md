# Critbit Tree

> A crit-bit tree stores a prefix-free set of bit strings: a set of 64-bit integers, for example, or a set of variable-length 0-terminated byte strings.

A crit-bit tree for a nonempty prefix-free set S of bit strings 
has an external node for each string in S; an internal node for 
each bit string x such that x0 and x1 are prefixes of strings in S; 
and ancestors defined by prefixes. The internal node for x is compressed: 
one simply stores the length of x, identifying the position of the critical 
bit (crit bit) that follows x.

## Critbit struct
```rust
#[repr(C)]
#[derive(Copy, Clone)]
pub struct Critbit<
    V: Default + Copy + Clone + Pod + Zeroable,
    const NUM_NODES: usize,
    const MAX_SIZE: usize,
> {
    _padding0: u64,
    /// Root node of the critbit tree
    pub root: u32,
    _padding1: u32,
    /// Allocator corresponding to inner nodes and leaf pointers of the critbit
    node_allocator: NodeAllocator<CritbitNode, NUM_NODES, 4>,
    /// Allocator corresponding to the leaves of the critbit. Note that this
    /// requires 4 registers per leaf to support proper alignment (for aarch64)
    leaves: NodeAllocator<V, MAX_SIZE, 4>,
}
```
### Generics
1. `NUM_NODES` - This is total number of nodes (internal, external and leaves) in this tree
`NUM_NODES` is always double of `MAX_SIZE`
2. `MAX_SIZE` - This is maximum size of the tree (maximum data objects)

### Fields
1. `_padding0` and `_padding` are two prefixes of any bit string in the tree
2. `root` - the root node index
3. `node_allocator` - The allocator for inner nodes and leaf pointers of the critbit
4. `leaves` - This allocator is for critbit tree's leaves with 4 required registers for alignment

## CritbitNode struct
```rust
#[repr(C)]
#[derive(Default, Copy, Clone)]
pub struct CritbitNode {
    pub key: u128,
    pub prefix_len: u64,
    pub _padding: u64,
}
```

### Fields
1. `key` - The search key
2. `prefix_len` - The trailing zeroes 