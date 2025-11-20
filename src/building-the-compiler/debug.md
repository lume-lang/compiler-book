# Debugging the compiler

The Lume compiler isn't impervious to bugs - far from it! This chapter exists to outline how one would start with
debugging the compiler, given some bug. Most of this should be transferable to most parts of the compiler.

## Helpful flags

When using a development build of the compiler, additional flags are possible to pass to `lume`. Some of them are
listed here, since they can be useful for some specific issues.

### Getting a backtrace on errors

It can sometimes be hard to find out where an error is being emitted from. The `--panic-on-error` and `--track-diagnostics` flags can be useful, since they can help you place the exact line an error is being pushed.

- `--panic-on-error` does exactly what the name implies - when a diagnostic is pushed, it panics. This makes it possible
  for a developer to print a stacktrace, locating where the error was pushed:
  ```sh
  $ RUST_BACKTRACE=1 lume build ./sample --panic-on-error
  [track_diagnostics] pushed from compiler/lume_typech/src/query/mod.rs:238:32
  thread 'main' (1317499) panicked at compiler/lume_typech/src/query/mod.rs:238:32:
  error emitted with `panic_on_error` enabled: mismatched types
  stack backtrace:
     0: __rustc::rust_begin_unwind
               at /rustc/f04e3dfc87d7e2b6ad53e7a52253812cd62eba50/library/std/src/panicking.rs:698:5
     1: core::panicking::panic_fmt
               at /rustc/f04e3dfc87d7e2b6ad53e7a52253812cd62eba50/library/core/src/panicking.rs:80:14
     2: lume_errors::DiagCtxInner::push
               at ./crates/lume_errors/src/lib.rs:71:13
     3: lume_errors::DiagCtx::emit
               at ./crates/lume_errors/src/lib.rs:156:22
     4: lume_typech::query::<impl lume_typech::TyCheckCtx>::check_params
               at ./compiler/lume_typech/src/query/mod.rs:238:32
     5: lume_typech::query::<impl lume_typech::TyCheckCtx>::check_signature
               at ./compiler/lume_typech/src/query/mod.rs:78:14
    (~~~~ LINES REMOVED FOR BREVITY ~~~~)
  ```

- `--track-diagnostics` is very similar, expect it does not panic. Instead, it prints a single line for each diagnostic,
  outlining where the diagnostic was pushed from:
  ```sh
  $ lume build ./sample --track-diagnostics
  [track_diagnostics] pushed from compiler/lume_typech/src/query/mod.rs:238:32
  × error[LM4001]: mismatched types
      ╭─[/Users/max/Documents/Projects/Lume/samples/playground/main.lm:17:17]
   16 │     let a = 1_i32;
   17 │     let b = a + false;
      ∶                 ^^^^^ expected type Int32, but found type Boolean...
   18 │ }
      ╰──
       ╭─[int.lm:80:34]
   79  │ use Add<Int32> in Int32 {
   80  │     fn external add(self, other: Int32) -> Int32
       ∶                                  ^^^^^ ...because of type defined here
   81  │ }
       ╰──
     help: expected type Int32
              found type Boolean
  ```

### Dumping the MIR

When debugging an issue which only shows itself later in the compilation process, it might be useful to verify
that the MIR looks correct. To dump the MIR to the console output, you can pass the `--dump-mir` flag:
```sh
$ lume build ./sample --dump-mir
(~~~~ LINES REMOVED FOR BREVITY ~~~~)

@!3848283137416547773 fn "std::Range::new" (start: i32 #0, end: i32 #1) -> ptr std::Range {
B0:
    let #2: metadata std::Range = metadata std::Range()
    #3 = alloc std::Range
    mark object(#3)
    *#3[+x0] = #2
    *#3[+x8] = #0
    *#3[+xC] = #1
    let #4: ptr std::Range = +(#3, 8_u64)
    mark object(#4)
    return #4
}
```

Sometimes, an issue crops up because of a specific optimization pass. You can print the MIR before a specific pass
is invoked by passing the name of the pass to the `--dump-mir` flag:
```sh
$ lume build ./sample --dump-mir=mark_gc_refs
(~~~~ LINES REMOVED FOR BREVITY ~~~~)

@!3848283137416547773 fn "std::Range::new" (start: i32 #0, end: i32 #1) -> ptr std::Range {
B0:
    let #2: metadata std::Range = metadata std::Range()
    #3 = alloc std::Range
    *#3[+x0] = #2
    *#3[+x8] = #0
    *#3[+xC] = #1
    let #4: ptr std::Range = +(#3, 8_u64)
    return #4
}
```

## Tracing the compiler

It might sometimes be important to see the execution path of the compiler, to see why a certain operation
is executed. For that reason, the compiler has `#[traced]` attributes on most methods, which can print out
some information about the method call - passed arguments, return values, whether an error was returned, etc.

To see these logs, you need to enable the `tracing` feature on the compiler. Tracing is disabled by default, since
it can cause significant performance loss.

To enable it, you must have a local copy of the compiler and have passed the given command when building:
```sh
$ cargo build --bin lume --features tracing
```

You can then pass log filters via the `LUMEC_LOG` environment variable when running the compiler.

### Filtering traces

While you *could* print all traces to the console, this is likely not what you want. Given a function like this:
```rs,ignore
#[traced(level = Debug, fields(id))]
fn compiler_function(id: NodeId) -> TypeRef {}
```

you can print all it's invocations by using:
```sh
LUMEC_LOG=compiler_function=debug
```

which will cause all calls to `compiler_function` to be logged, along with the passed `id` argument.

This is under the assumption that `type_of_expr` exists without any parent module or external crate. You'll more
like see log filters like the following:
```sh
LUMEC_LOG=lume_infer::query::type_of=trace
```
- `lume_infer::query::type_of` is the full path of the method, including crate and module path.

#### Filter specific invocations

You can also filter traces by the arguments which are passed to them. Each trace has a list of arguments
which are added as "fields" to the trace. For example, the given method within the compiler
(location in the `lume_typech` crate, inside the `query` module):

```rs,ignore
#[traced(level = Trace, fields(name = expr.name()), err, ret)]
fn check_params(
    &self,
    expr: lume_hir::CallExpression<'_>,
    parameters: &lume_types::Parameters,
    arguments: &[&lume_hir::Expression],
) -> Result<bool> {}
```
has a field `name` equal to the name of the passed call expression. So, to filter all calls to `check_params`
which received a call expression to `my_function`, we can use the following filter:

```sh
LUMEC_LOG=lume_typech::query::check_params[name=my_function]=trace
```

For more information about the filtering, you can read the [`libftrace` documentation][EnvFilter] about the feature.

[EnvFilter]: https://docs.rs/libftrace/latest/libftrace/filter/struct.EnvFilter.html
