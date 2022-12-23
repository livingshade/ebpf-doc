# source level docs

Due to copy right issue, you have to generate document by yourself.

```bash
git clone git clone https://github.com/cubele/rCore-Tutorial-Code-2022A rcore-ebpf
cd rcore-ebpf
cd os8
cargo doc --no-deps --document-private-items
```

And then get the doc in target folder. You can use `--open` to open it in a folder. Mods other than `ebpf` part is about rCore, you should only care about `ebpf` part.
