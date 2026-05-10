# lake-stdlib

Standard library for the [Lake](https://github.com/morphqdd/lake-native-compiler)
programming language.  Pure Lake source — no Rust, no FFI per module —
everything lives on top of the runtime functions the compiler ships with.

## Modules

| module      | exports                                  | uses                                              |
|---          |---                                       |---                                                |
| `std.io`    | `print`, `println`, `eprint`, `eprintln` | `rt_write`, `len`                                 |
| `std.math`  | `abs`, `min`, `max`, `mod`               | (none — pure compute)                             |
| `std.sys`   | `exit`                                   | `rt_exit`                                         |
| `std.tcp`   | `listen`, `accept`, `recv`, `send`, `close` | `rt_listen_tcp`, `rt_accept_async`, `rt_recv_async`, `rt_send_async`, `rt_close`, `len` |

Every machine here is `pub`.  Internal helpers (none yet) would be
private by default.

Most TCP wrappers are ret-typed — `let conn = accept(srv)` blocks the
caller until the underlying io_uring CQE arrives, the same way the
underlying `rt_accept_async` parks a direct caller.  The wrapper costs
one extra mailbox roundtrip; for tight per-byte loops bypass the
stdlib and call `rt_*` directly.

## Output ordering

`std.io` machines (`print`, `println`, `eprint`, `eprintln`) are all
ret-typed.  This gives you a per-call choice:

```lake
println("a")       // async — fire-and-forget; output order is
println("b")       // whatever the scheduler picks
println("c")
//
// $ ./prog
// c
// b
// a   ← typical: scheduler runs in LIFO order
```

```lake
pin println("a")   // sync — block until the wrapper actor
pin println("b")   // finishes its rt_write and replies
pin println("c")
//
// $ ./prog
// a
// b
// c   ← guaranteed
```

`pin <expr>` is sugar for `let __pin_<id> = <expr>`: it forces the
ret-machine sync return path and discards the value.  Use `let r = M(args)`
when you want to capture the returned value.

The `pin` form costs one mailbox roundtrip (~µs on the current
scheduler) per call.  Programs that want guaranteed sequential output
in a hot loop should bypass the stdlib and call `rt_write` directly —
it's a blocking syscall, no actor involved.

## Installation

There is no package manager yet.  Two ways to expose this library to
your projects:

### Per-library env var (recommended)

```bash
export STD_PATH=/path/to/lake-stdlib/std
```

The Lake compiler resolves `+std.io.println` by looking for a file
`<STD_PATH>/io.lake` (the `std` segment is consumed by the env var
name).

### Generic search path

```bash
export LAKE_PATH=/path/to/lake-stdlib
```

Each segment of the import path appears verbatim under `LAKE_PATH`,
so `+std.io.println` resolves to `<LAKE_PATH>/std/io.lake`.

Both can be set; resolution order is project root → `<NAME>_PATH` →
`LAKE_PATH`.

## Usage

```lake
+std.io.{ println }

main is {
  _ -> {
    println("hello from lake-stdlib")
  }
}
```

```bash
$ STD_PATH=/path/to/lake-stdlib/std lakec my_app.lake -r
$ ./build/my_app
hello from lake-stdlib
```

## Examples

`examples/hello/` — minimal program that imports `println` and prints a
greeting.

`examples/echo/` — TCP echo server using `std.tcp.{ listen, accept,
send, close }`.

`examples/http_print/` — listens on `:8080`, prints every incoming
HTTP request to stdout, replies with a tiny `HTTP/1.0 200 OK`.
Demonstrates the full `recv` → `print` → `send` → `close` flow over
the stdlib.

```bash
$ STD_PATH=/path/to/lake-stdlib/std \
    lakec examples/http_print/main.lake -r
$ ./examples/http_print/build/main &
listening on :8080
$ curl -s http://127.0.0.1:8080/hello
hi
$ # server printed:
GET /hello HTTP/1.1
Host: 127.0.0.1:8080
User-Agent: curl/...
Accept: */*
```

## License

MIT.

## Status

Bootstrap stage.  Functions are minimal; expect rapid churn until Lake
gains:

- typed PIDs (so message types are checked at compile time)
- generic types (so `result(T, E)` and `option(T)` can be modeled)
- struct / sum types (so `list`, `result`, `option` can be implemented)

Until then, this library covers the ergonomic primitives that the
existing language already supports cleanly.
