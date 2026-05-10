# std.experimental

Direct-syscall I/O primitives.  Every machine in this directory
maps to *exactly one* `syscall(2)` instruction — no fat-pointer
indirection, no bounds-checked runtime helpers, no scheduler
hand-off.

## When to reach for this

Use the regular `std.io` / `std.tcp` modules in normal code.  They
go through the compiler-shipped runtime helpers (`rt_write`,
`rt_recv_async`, …) which are bounds-checked and integrate with
the Lake scheduler — async I/O parks the actor on the io_uring
CQE, message-passing actors stay cooperative.

Reach for `std.experimental` when you want one of:

  * **Smaller binaries.**  Skipping `rt_write` shaves ~700 B off
    the resulting executable; on a 6 KB program that's noticeable.
  * **Educational / reverse-engineering code** that wants to make
    the syscall boundary visible at the source level.
  * **Bootstrapping a new platform** where the bounded runtime
    helpers haven't been ported yet but raw `syscall` works.

You give up:

  * Bounds checks (`rt_write` traps on a malformed fat-ptr; the
    raw write happily reads past the end).
  * The async-I/O integration (these wrappers are blocking; the
    whole scheduler thread sits inside the kernel until the
    syscall returns).

## Modules

| module | exports |
|---|---|
| `std.experimental.sys` | `SYS_*` syscall numbers, `FD_*` constants, `exit`, `getpid` |
| `std.experimental.io`  | `print`, `println`, `eprint`, `eprintln` |

## Memory-safety boundary

`rt_str_ptr(s)` returns the *raw byte address* of a string's
payload — distinct from the cast `s as i64`, which gives the
fat-pointer header address.  That value is fine to pass to a
syscall that expects a buffer pointer (`write`, `sendto`, …).
**Don't pass it back to `rt_load_*` / `rt_store`**: those helpers
expect a fat-pointer header at the address, not a raw byte
buffer, and will trap once the bounds check fails or — worse —
silently corrupt the heap if the random bytes at the byte address
happen to look like a plausible `(start, end)` pair.

## Example

```lake
+std.experimental.io.{ println }
+std.experimental.sys.{ exit }

main is {
  _ -> {
    pin println("hello from raw syscalls")
    exit(0)
  }
}
```

Compile and run from the repository root:

```bash
$ STD_PATH=$(pwd)/std \
    lakec examples/experimental_hello/main.lake -r
$ ./examples/experimental_hello/build/main
hello from raw syscalls
```
