# migrate eBPF to rCore tutorial

This is a step-by-step tutorial of how to migrate eBPF to [rCore](https://github.com/cubele/rCore-Tutorial-Code-2022A).

## Environment Setup

Note that rCore tutorial only provides rust user programs but their is no rust support for eBPF yet. In that case, we need to use [uCore-test](https://github.com/livingshade/uCore-Tutorial-Test-2022A) to compile the user programs because uCore and rCore share the same ABI.

First, clone the repo

```bash
git clone https://github.com/cubele/rCore-Tutorial-Code-2022A rcore-ebpf
cd rcore-ebpf
git submodule update --init --recursive
```

### Install dependencies

We need rust, llvm, clang, qemu and musl toolchains.

You may need to install toolchains from [musl.cc](https://musl.cc/), qemu version >= 6.0.0, clang version = 12, and add them to your PATH.

```bash
make env # this will install rust dependencies
```

### Change config

You need to mannually change the path config in `user/ebpf/build.sh`, you basically just need to change the prefix of `builddir, ucoredir, targetdir`, use absolute path. Then,

```bash
cd os8
make run
```

You should see that rCore is started and the application is running. Run `ebpf_user_loadprogextest` and `ebpf_user_kernmaptest` for a glimpse.

## Migration

In to the section [overview](ebpf.md), we now needs to add some os-dependent code to make it work. We only needs to add the syscall interface and implement the `osutils.rs` to provide necessary kernel functions.

We should always bear in mind that rCore kernel has its own page table, rather than those share with user space, like Linux and zCore.

### Add syscall interface

We need to add the following syscall interface to `syscall.rs`:

First, add the syscall number to `syscall/mod.rs`:

```rust
//! syscall/mod.rs
const SYSCALL_BPF: usize = 280; 
// ...
pub fn syscall(syscall_id: usize, args: [usize; 4]) -> isize {
    match syscall_id {
        // ...
        SYSCALL_BPF => sys_bpf(args[0] as isize, args[1] as usize, args[2] as usize),
    }
```

Then, implement `sys_bpf`

```rust
//! syscall/ebpf.rs
pub fn sys_bpf(cmd: isize, bpf_attr: usize , size: usize) -> isize {
    let ptr = bpf_attr as *const u8;
    let cmd = cmd as i32;
    use crate::ebpf::BpfCommand::*;
    if let Ok(bpf_cmd) = BpfCommand::try_from(cmd) {
        let ret = match bpf_cmd {
            BPF_MAP_CREATE => sys_bpf_map_create(ptr, size),
            BPF_MAP_LOOKUP_ELEM => sys_bpf_map_lookup_elem(ptr, size),
            BPF_MAP_UPDATE_ELEM => sys_bpf_map_update_elem(ptr, size),
            BPF_MAP_DELETE_ELEM => sys_bpf_map_delete_elem(ptr, size),
            BPF_MAP_GET_NEXT_KEY => sys_bpf_map_get_next_key(ptr, size),
            BPF_PROG_LOAD => todo!(),
            BPF_PROG_ATTACH => sys_bpf_program_attach(ptr, size),
            BPF_PROG_DETACH => sys_bpf_program_detach(ptr, size),
            BPF_PROG_LOAD_EX => sys_preprocess_bpf_program_load_ex(ptr, size),
        };
        if ret < 0 {
            -1
        } else {
           ret as isize
        }
    } else {
        -1
    }
```

We just need to add the syscall interface and use a match to call the corresponding function. We pass the `attr` as argument and let the function to handle it. Note that `BpfCommand` is an enum that represents the command type following the Linux API.

And we are finished with the syscall interface. This is quite simple, isn't it?

### Implement `osutils.rs`

In `osutils.rs`, we need to implement the following parts:

* A trait `ThreadLike` that is an analog to Linux thread. It should have `get_pid`, `get_tid` and `get_name` methods. And the function `os_current_thread` that returns that thread-like object.

In rCore, we know that `TaskControlBlock` is the thread-like object. So we can implement the trait for `TaskControlBlock` and return the current thread.

```rust
impl ThreadLike for TaskControlBlock {
    fn get_pid(&self) -> u64 {
        let proc = self.process.upgrade().unwrap();
        return proc.pid.0 as u64;
    }
    fn get_tid(&self) -> u64 {
        return 0; // no viable in rcore tutor
    }
    fn get_name(&self) -> String {
        return String::from("not viable in rcore tutorial")
    }
}
```

Note that we don't have a `tid` in rCore. So just return 0. And we don't have a process name. So we just return a dummy string. Also, we need to implement the `os_current_thread` function.

```rust
pub fn os_current_thread() -> Arc<dyn ThreadLike> {
    if let Some(thread) = crate::task::current_task() {
        thread
    } else {
        panic!("cannot get current thread!")
    }
}
```

* `os_current_time` that returns the current time in nanoseconds. `os_get_current_cpu` that returns the current cpu id. `os_console_write_str` that writes a string to the console. Those are used in helper functions.

```rust
pub fn os_current_time() -> u128 {
   crate::timer::get_time_us() as u128 * 1000
}

pub fn os_get_current_cpu() -> u8 {
   0 // not viable
}

pub fn os_console_write_str(s: &str) {
    crate::console::Stdout.write_str(s).unwrap();
}
```

Again, rCore is a single-core system. So we just return 0 for `os_get_current_cpu`. And we use `Stdout` to write to the console.

* `os_copy_from_user` and `os_copy_to_user` that copy data from/to user space. Because some OS use virtual memory, we need to translate the address. This two functions are primitives for other memory related functions like `get_generic_from_user` and all other syscall branches.

```rust
pub fn os_copy_from_user(usr_addr: usize, kern_buf: *mut u8, len: usize) -> i32 {
    use crate::mm::translated_byte_buffer;
    use crate::task::current_user_token;
    let t = translated_byte_buffer(current_user_token(), usr_addr as *const u8, len);    
    let mut all = vec![];
    for i in t {
        all.extend(i.to_vec());
    }
    copy(kern_buf, all.as_ptr() as *const u8, len);
    0
}
 
pub fn os_copy_to_user(usr_addr: usize, kern_buf: *const u8, len: usize) -> i32 {
    use crate::mm::translated_byte_buffer;
    use crate::task::current_user_token;
    let dst = translated_byte_buffer(current_user_token(), usr_addr as *const u8, len);
    let mut ptr = kern_buf;
    let mut total_len = len as i32;
    for seg in dst {
        let cur_len = seg.len();
        total_len -= cur_len as i32;
        unsafe {
            core::ptr::copy_nonoverlapping(ptr, seg.as_mut_ptr(), cur_len);
            ptr = ptr.add(cur_len);   
        }
    }
    assert_eq!(total_len, 0);
    0
}

// You don't need to change this two functions
pub fn copy(dst: *mut u8, src: *const u8, len: usize) {
    let from = unsafe { from_raw_parts(src, len) };
    let to = unsafe { from_raw_parts_mut(dst, len) };
    to.copy_from_slice(from);
}

pub fn memcmp(u: *const u8, v: *const u8, len: usize) -> bool {
    return unsafe {
        from_raw_parts(u, len) == from_raw_parts(v, len)
    }
}
```

This part is very tricky, since rCore use different page table for kernel and user space. So for every pointer from user space, we needs to translate and copy it to kernel buffer and vice versa. We use `translated_byte_buffer` to translate the address, which is provided by `mm`.

It is worth note that for zCore, which use the same page table for kernel and user space, the two function is just a `memcpy`.

And that is all for `osutils.rs` ! You can refer to the source code to see we use this two primitives to build `get_generic_from_user`, and use that to get the `attr` from user space. And all other syscall branches are done. That is because their logic is os-independent, we only needs to provide a memory interface.

### BPF map operations

We don't need to change any code here, but I want to show that how different page table effect the migration.

Nte that functions like `bpf_map_lookup_elem` can be called both from kernel space and user space. It is nothing important in zCore, since zCore use the same page table for kernel and user space. But in rCore, we need to be especially careful.

When a user program wants to access a map, it needs to call `bpf_map_lookup_elem` in user space. And the syscall interface will call `sys_bpf_map_lookup_elem` in kernel space. In that case, address are from user space, and we need the additional translation.

But a loaded BPF program can access maps as well and it runs in kernel space. In that case, the address is from kernel space, and we don't need the translation.  

So, we need to pass a flag to tell the function whether the address is from user space or not. Argument `from_user: bool` in that case. For user program, it calls through `sys_bpf_map_...`, which passes `from_user = true`. For BPF program, it calls `bpf_helper_map_...`, which passes `from_user = false`.

Let's take a look at the code of `bpf_map_ops`, in which the actual map operations are done.

```rust
pub fn bpf_map_ops(fd: u32, op: BpfMapOp, key: *const u8, value: *mut u8, flags: u64, from_user: bool) -> BpfResult {
    let bpf_objs = BPF_OBJECTS.lock();
    let obj = bpf_objs.get(&fd).ok_or(ENOENT)?;
    let shared_map = obj.is_map().ok_or(ENOENT)?;
    let mut map = shared_map.lock();
    if from_user {
        let key_size = map.get_attr().key_size;
        let value_size = map.get_attr().value_size;
        let mut key_kern_buf = alloc::vec![0 as u8; key_size];
        let kptr = key_kern_buf.as_mut_ptr();
        os_copy_from_user(key as usize, kptr, key_size);
        let mut value_kern_buf = alloc::vec![0 as u8; value_size];
        let vptr = value_kern_buf.as_mut_ptr();
        match op {
            BpfMapOp::LookUp => {
                let ret = map.lookup(kptr, vptr);
                os_copy_to_user(value as usize, vptr, value_size);
                ret
            },
            BpfMapOp::Update => {
                os_copy_from_user(value as usize, vptr, value_size);
                let ret = map.update(kptr, vptr, flags);
                ret
            },
            BpfMapOp::Delete => map.delete(kptr),
            BpfMapOp::GetNextKey => {
                let ret = map.next_key(kptr, vptr);
                os_copy_to_user(value as usize, vptr, value_size);
                ret
            }
            _ => Err(EINVAL),
        }
    } else {
        match op {
            BpfMapOp::LookUp => map.lookup(key, value),
            BpfMapOp::Update => map.update(key, value, flags),
            BpfMapOp::Delete => map.delete(key),
            BpfMapOp::GetNextKey => map.next_key(key, value),
            _ => Err(EINVAL),
        }
    }    
}
```

We can see that we need extra copy when `from_user` is true. And we use `os_copy_from_user` and `os_copy_to_user` to do the copy. For OS like zCore, those steps are redundant. An extra copy will make it run slower, but does not harm the correctness. This is the fundamental trade off between generality and performance.

## Add user program

Now we need add user program that test our eBPF utility. There are 4 user tests.

* `ebpf_user_naivetest.c` is a naive test that just call `sys_bpf` to test the syscall interface.
* `ebpf_user_maptest.c` is a test that test the map operations from user program.
* `ebpf_user_loadprogextest.c` is a test that load a BPF kernel program `context.c` from user program and print out the registers. This tests the kprobe and some helper functions.
* `ebpf_user_kernmaptest.c` is a test that load a BPF kernel program `map.c` from user program and test the map operations from kernel program. This tests the bpf map in kernel space.

You can run them using `make run`.

### Add more user programs

If you want to add more user programs, you need to add the source file in `ucore/src` just like the 4 tests above.

If you want to add kernel BPF programs and load them, you need to add the kernel source file in `user/ebpf/kern` just like `map.c` and `context.c`, and do the following thing:

```bash
# suppose you add map2.c as kernal program
cd user/ebpf/kern
make
python3 hex2char.py 
# copy the output, which is the length of kern prog
# copy map2.dump to your user program, take ebpf_user_loadprogextest.c as example
```

That is because rCore tutorial does not support `mmap` or `malloc`, we have to mannually copy the byte code into a `char []` . `hex2char.py` reads an ELF and print out its hex value byte by byte. We also have to mannually set the length of the program as argument. You can refer to zCore test program if you have implmented `mmap` or `malloc` and `fstat` for kernel.

Don't forget to change `build.sh` before you type `make run`

### Details about Makefile

This section can be skipped if you are not interested in the details. But to do it from scratch, you need to know the whole procedure.

We need to compile the user program and link it with our library. We just use uCore-Test's makefile because its compatitble with rCore. But for eBPF programs, we need a user library `bpf.h` and `bpf.c`, in `ucore/include`, `ucore/lib`, respectively.

When we call `make` in `/ucore`, it will compile the user program and the ELF is in `ucore/build/riscv64`.

Then we need to copy the elf to rCore's filesystem, which is done by `user/ebpf/build.sh`. In the script, we touch a empty file called `[testcase].rs` and copy the elf to target directory. Then we run `make run` in `os8`.

You can refer to makefile `fs-img` and `fn easy_fs_pack` for the reason. Those steps are quiet stupid and you might find better solution.

## Further work

Currently, the symbol table is not supported, thus the kprobe attach target is hardcoded in `tracepoint.rs::resolve_symbol` as `sys_open`. You can try to support it by adding the symbol table utility. You may refer to [zCore's repo](https://cubele.github.io/probe-docs/ebpf%E7%A7%BB%E6%A4%8D/zCore%E5%9F%BA%E7%A1%80%E8%AE%BE%E6%96%BD/) and [this blog](https://cubele.github.io/probe-docs/ebpf%E7%A7%BB%E6%A4%8D/zCore%E5%9F%BA%E7%A1%80%E8%AE%BE%E6%96%BD/)
