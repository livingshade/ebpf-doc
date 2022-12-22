# eBPF strcuture

eBPF is bascially a method for user to run a program in kernel space.

User needs to choose a hookpoint(some kernel functions) and the bpf program. Program should consits only limited eBPF instructions, and the program will be executed when the hookpoint is triggered.

To achieve that, kernel needs to provide a way to load program into kernel, a bridge to transfer data between user space and kernel space(BPF maps). In Linux, a dedicated BPF syscall is provided. We will use the same API as Linux, but the implementation is different and much simple.

## Hook

In Linux kernel, eBPF is widely applied in network stack.(i.e. xDP) However, given that some Rust-based OS does not implment netowrk stack, we use kprobes as hookpoint.

Since eBPF does not care about the hookpoints, it only needs a instruction to call the BPF program, and, provide necessary contexts. One could imagine that the hookpoint is something like `call $bpf_program(arg0 = context)`. And for kprobes, the context is the registers.

Thus, the implmentation detail of kprobes is not the focus of this project, so we will not discuss it here. One should refer to [this](https://www.kernel.org/doc/html/latest/core-api/kprobes.html) for more information.

## Overview

As discussed above, eBPF is a method for user to run a program in kernel space. The program is loaded by a syscall, and the program will be executed when the hookpoint is triggered. The program is a set of instructions, and the program can access data in kernel space through BPF maps. So, for any OS, we need the following components:

* A new bpf syscall
* Kernel memory to store the BPF programs and BPF maps
* The actual implementation of abstract functionalities

We will talk about the last component, since the first two are OS-dependent.

### BPF instructions and helper functions

BPF codes are executed in bytecode, and an ISA is provided [here](https://www.kernel.org/doc/html/latest/bpf/bpf_devel_QA.html#how-to-write-a-bpf-program). The instruction set is very limited, and it is designed to be safe.

Since the BPF instruction are limited, the kernel needs to provide some special helper functions for BPF program to use. For example, `bpf_helper_printk` is a helper function that prints a string to kernel log. Thus user can read the output to know what is happening.

But writing assembly code is hard. So we write C code and use clang to compile it to BPF bytecode. And the helper functions are just symbols in user codes, kernel needs to relocate them to the actual address.

### BPF maps

A BPF map a key-value store in kernel space. The key and value are both fixed size, and the size is defined when the map is created. The map is created by the syscall, and the map is identified by a file descriptor.

Note that the communication between BPF program and maps are through file descriptors and helper functions like `bpf_helper_map_lookup_elem`.

There is only two kinds of maps `Array` and `HashTable` and their per-CPU version. The difference between `Array` and `HashTable` is that `Array` is indexed by a number, and `HashTable` is indexed by a key. We omit the per-CPU version and use mutex for concurrency.

The maps are no different from their user space corrspodent, except that the map is in kernel space. One could refer to the source code level [documentation](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/bpf.h#L100) for more information.

### load BPF programs

A BPF program some thing that user wants to execuate in kernel space. The user program will send the bytecode to kernel space, and the kernel will first verify the bytecode for safty concern. There are also some relocation needs to be. And then the code is JITed into machine code and stored in kernel space. So, it looks like:

```c
void *bpf_prog_load(void *prog, size_t prog_len) {
    if (!verify_and_relocate(prog, prog_len)) {
        return NULL;
    }
    void *machine_code = jit(prog, prog_len);
    return machine_code;
}
```

And when the program needs to be executed, the `machine_code` is cast to a function pointer and called.

We won't talk about verification and JIT here. We simple take them as library. One could refer to [this](https://www.kernel.org/doc/html/latest/bpf/bpf_devel_QA.html#how-to-write-a-bpf-program) for verification and [this](https://www.kernel.org/doc/html/latest/bpf/bpf_devel_QA.html#how-to-jit-a-bpf-program) for JIT.

### BPF relocations

We can skip verification, but we can't skip relocation. The relocation is needed mainly because the program is loaded into kernel space, and the program needs to access the BPF maps.BPF helper functions also need to be relocated.

Note that in Linux kernel, the relocation and verification are coupled and it is more than maps and helpers that needs to be relocated. For simplicity, we will not discuss it here, but one could refer to [this](https://www.kernel.org/doc/html/latest/bpf/bpf_devel_QA.html#how-to-write-a-bpf-program) for more information.
