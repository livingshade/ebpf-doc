# zCore reproduce guide

```bash
git clone https://github.com/cubele/zCore zcore-ebpf
cd zcore-ebpf
```

## dependencies

Install rust and set up the environment.
You might needs to run `rustup default nightly`

Install musl tool chain.

```bash
wget https://musl.cc/riscv64-linux-musl-cross.tgz
tar -xf riscv64-linux-musl-cross.tgz
export PATH = $PATH:$(pwd)/riscv64-linux-musl-cross/bin # you need to change this to your own path
```

Install qemu 7.1.0, you might need to compile from source. And add qemu to your path.

Install clang-12.

And install other dependencies described in zCore readme.

## build and run

zCore uses xtask for project build

```bash
cargo other-test --arch=riscv64 # this will put eBPF test cases in rootfs
cargo qemu --arch=riscv64 # this will run qemu
```

Then you can see the test cases in qemu. `kernmaptest` and `loadprogextest` are the eBPF test cases.

## source code

The eBPF related code is in `zircon-object/src/ebpf` folder.

The eBPF related syscall is in `linux-syscall/src/ebpf` folder.

The eBPF testcase is in `linux-syscall/test/ebpf` folder.

You can inspect the code and add your own testcases.
