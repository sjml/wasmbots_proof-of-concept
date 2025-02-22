# WasmBots proof-of-concept

Playing around with a basic TypeScript host that can run WASM programs of a certain specification. 

At the moment, each of them implements: 
* `setup` function, which takes a parameter saying how much space to reserve for host communication. In whatever way makes sense to the language, set aside a block of memory of the given size (2048 bytes at the moment). The idea is that the host will write to this memory for non-trivial data like strings and structs that cannot be simply passed as function parameters between WASM and the host. **returns** a pointer to where that memory resides in the overall WASM memory block. 
* `runFib` function, which is largely just a proof that two-way communication is working via the reserved memory space allocated in `setup`. Takes two parameters, one a lookup location (offset into the reserved memory space) for where to find a single byte value which indicates which Fibonacci number to calculate. The second parameter is the offset in the shared space where the WASM program should write the output as a little-endian unsigned 64-bit integer. **returns** whether the whole operation was successful (could fail if given a Fibonacci number too large to fit in a u64, or a result offset where the answer could not fit, etc.)

The `runFib` functions all intentionally use an inefficient recursive Fibonacci algorithm, just to show that some actual computation is happening. The host only asks for the 42nd Fibonacci number (267,914,296) right now, so none of them churn for _too_ long. 

## Testing

Prereqs on macOS; modify this appropriately if you're using something else: 
```
brew install emscripten wabt deno zig rust node
```

Run [`scripts/build_and_run_wasms.sh`](./scripts/build_and_run_wasms.sh). 

## (Lack of) Memory Constraints

My original idea was to limit the memory sizes of the WASM programs to something very small, just as an exercise in old-school parsimony. At first just a single page of memory (64k) then I went up to a megabyte... but what I found was that when the WASM **host** manages memory, the programs themselves have to be compiled with directives to import their memory, and you also have to manually tell them how much to expect to have. Seems simple enough EXCEPT most programming languages these days don't seem to play well with such restrictions, at least not if I make them low enough to be interesting. Rust's default allocator in particular [doesn't work if it can't grow the memory](https://github.com/rustwasm/wasm-bindgen/issues/1389#issuecomment-476224477). 

This is all based on very cursory experiments and googling around. I could get all of the example programs building with imported memory, but they broke in different and somewhat unpredictable ways. I am not (yet?) enough of a master of WASM<->Host shared memory and all the various flags one can pass to LLVM backends to simulate a constrained environment. In any case, it's not the interesting part of what I'm doing here so for now I'm letting each hosted program just do whatever it wants with memory, at least until it runs into the actual engine's memory limits (which I think are around 4 GB?)

First step in constraining this would be just leaving it as-is but killing any hosted program that used more than _[X]_ amount of memory... but it would be much nicer if the language's allocator could just "properly" fail instead. (This could probably be solved by writing custom allocators? No need to shave those yaks just yet.)
