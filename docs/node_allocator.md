# Node Allocator

> Almost all the data structures can be represented by some sort of connected graphs of nodes and edges.

The `NodeAllocator` is a raw node allocation structure implemented for _contiguous buffers_.
The objects contained should also be of the _same underlying type_ (allocator's). Each entry (node) will have a fixed
number of `registers` (predefined `NUM_REGISTERS`) that contains the metadata related to the current node. These registers
will be interpreted as _graph edges_.

### Example
```rust
// register aliases
pub const PREV: u32 = 0;
pub const NEXT: u32 = 1;

#[derive(Copy, Clone)]
pub struct DLL<T: Default + Copy + Clone + Pod + Zeroable, const MAX_SIZE: usize> {
    pub head: u32,
    pub tail: u32,
    allocator: NodeAllocator<T, MAX_SIZE, 2>
}
```

The above Doubly Linked List is wrapped as a `NodeAllocator` 
having 2 higher level variables - `head` & `tail` and an allocator
with `MAX_SIZE` constant for maximum number of nodes it can hold
in the allocator. There are 2 registers allocated for every node in this
allocator. `register[0]` is always used for `free_list_head` which we will 
discuss in further sections. The final `register[1]` will be used to indicate
if to go next or prev with the help of `register aliases`. This register will hold
the enum values of these actions.

## NodeAllocator Struct
```rust
#[repr(C)]
#[derive(Copy, Clone)]
pub struct NodeAllocator<
    T: Default + Copy + Clone + Pod + Zeroable,
    const MAX_SIZE: usize,
    const NUM_REGISTERS: usize,
> {
    pub size: u64,
    
    bump_index: u32,
    /// Buffer index of the first element in the free list. The free list is a singly-linked list
    /// of unallocated nodes. The free list operates like a stack. When a node is removed from the
    /// allocator, the removed node becomes the new free list head. When new nodes are added,
    /// the new index to allocated is pulled from the `free_list_head`
    free_list_head: u32,
    /// Nodes containing data, with `NUM_REGISTERS` registers that store arbitrary data  
    pub nodes: [Node<T, NUM_REGISTERS>; MAX_SIZE],
}

// source: https://github.com/Ellipsis-Labs/sokoban
```
### Generics
The `NodeAllocator` struct is made up of three generics that modifies
how the way allocator will behave and serve the need.
1. `T` - A generic type indicating Critbit / AVL / Hash / Red-Black tree.
2. `MAX_SIZE` - It's a constant representing the maximum size of this allocator 
(can be interpreted as maximum data points/nodes we can store).
3. `NUM_REGISTERS` - It's a constant representing number of registers to allocate
for each Node to store the metadata.

### Fields
The `NodeAllocator` struct has 4 fields:
1. `pub size: u64` - The size of the Allocator `(size <= MAX_SIZE)`
2. `bump_index: u32` - This is the index starting from 1 till `MAX_SIZE`
and indicates that all nodes has been used at least once and already allocated
nodes must be pulled from the free list stack.
3. `free_list_head: u32` - The free list is a singly-linked list (operates like a stack) for all the unallocated nodes
When a node is removed from the allocator, the removed node becomes the new free list head and when new nodes are added,
the new index to allocated is pulled from the `free_list_head`
4. `pub nodes: [Node<T, NUM_REGISTERS>; MAX_SIZE]` - These are actual nodes containing
data, with `NUM_REGISTERS` registers for each node and `MAX_SIZE` total nodes.

### Implementations

#### fn assert_proper_alignment
The `assert_proper_alignment` function is a safety check to ensure that the memory 
alignment and size constraints for a NodeAllocator structure are properly maintained. 
This function verifies the `alignment` and `size` properties of the `NodeAllocator` and the `type T` 
stored within it, ensuring that the memory layout adheres to expected standards, 
which is crucial for preventing undefined behavior and improving performance.

#### inline fn get(i: u32) -> &Node
Returns the Node reference based on the index provided (assumes indices are 1-based)

#### inline fn get_mut(i: u32) -> &mut Node
Same as above, only mutable Node reference is returned 

#### fn add_node(node: T) -> u32
`node` can be of type T (let's assume Crit-bit Node) and this function returns the
current pointer to the free list, where the new node is inserted.
- it checks if `free_list_head` equals `bump_index` and if true then again checks for
bump_index to not be full and then increment bump_index and assign free_list_head to bump_index
- else get the `free_list_register` with the help of Node index `i` and set `free_list_head` to that value
and later mutate the register to `SENTINEL`
- mutate that index's Node and set `node` as its value
- increment the size and return the index

#### fn remove_node(i: u32) -> u32
Removes the node at index `i` and adds the index to the free list. All the registers 
must be cleared prior to calling this function. 

#### fn disconnect(i, j, r_i, r_j)
This simply clears the registers for both the `i` and `j` node.

#### fn connect(i, i, r_i, r_j)
This sets the r_i to register of `i` and similar for node `j`

