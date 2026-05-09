# lake-stdlib

Standard library for the [Lake](https://github.com/morphqdd/lake-native-compiler)
programming language.  Pure Lake source — no Rust, no FFI per module —
everything lives on top of the runtime functions the compiler ships with.

## Modules

| module      | exports                          | uses                       |
|---          |---                               |---                         |
| `std.io`    | `print`, `println`, `eprint`, `eprintln` | `rt_write`, `len`     |
| `std.math`  | `abs`, `min`, `max`, `mod`       | (none — pure compute)      |
| `std.sys`   | `exit`                           | `rt_exit`                  |

Every machine here is `pub`.  Internal helpers (none yet) would be
private by default.

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
