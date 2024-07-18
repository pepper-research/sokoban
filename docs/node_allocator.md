# Node Allocator

> Almost all the data structures can be represented by some sort of connected graphs of nodes and edges.

The `NodeAllocator` is a raw node allocation structure implemented for _contiguous buffers_.
The objects contained should also be of the _same underlying type_ (allocator's). Each entry (node) will have a fixed
number of `registers` (predefined `NUM_REGISTERS`) that contains the metadata related to the current node. These registers
will be interpreted as _graph edges_.

## Example
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