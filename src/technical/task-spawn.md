# Task Spawning

The FreeRTOS task spawning API in PROS works as follows:
* It takes a C calling convention function pointer to a function with one argument of type `void*`.
* It takes a `void*` that should be passed to the given function.
* It takes general information about the task. This includes stack size and priority.

You can pass the pros-rs task spawning API any type implementing `FnOnce() + Send + 'static`
and general info about the task.
This is significantly better because it means that you can use closures
and rust calling convention function pointers.
This is a huge improvement because you can avoid code like this:
```rust
unsafe extern "C" fn simple_task(args: *mut core::ffi::c_void) {
    let args: Box<String> = unsafe { Box::from_raw(args as *mut _) };
    loop {
        println!("{}", args);
    }
}
```
and instead write code like this:
```rust
let message = "Hi";
let simple_task = move || {
    loop {
        println!("{}", message);
    }
};
```

Internally, pros-rs is taking your functions,1
putting them on the heap, and then passing them through the arguments pointer
to a pros-rs `extern "C"` function that calls your function in the newly created task. 
The majority takes place in `TaskEntrypoint`,
specifically the `TaskEntrypoint::cast_and_call_external` function.

### Drawbacks

The biggest drawback of this implementation is that the functions passed by the user need to be put on the heap.
If you pass a closure that closes over 100MB of data, pros-rs will allocate 100MB of data on the heap
just to spawn your task.
Honestly speaking, it would take a pretty huge performance issue to outweight the ease of use
that this implementation brings.