# Contributing

These notes cover both [pipewire-flow](https://github.com/pipewire-flow/pipewire-flow)
(the C library) and [pipewire-flow-rs](https://github.com/pipewire-flow/pipewire-flow-rs)
(the Rust bindings). Anything a repository needs to say differently, it says
in its own README.

Both target Linux, because PipeWire does. Tests that need the graph connect
real streams to a running PipeWire daemon rather than mocking it.

## Building and testing

The C library builds with Meson:

```sh
meson setup build
meson compile -C build
meson test -C build
```

`meson test` needs no hardware. Tests that do — they link a real camera,
microphone or sensor into a filter — and tests that measure timing live in a
separate suite, and report SKIP rather than failing when the device is
absent:

```sh
meson test -C build --suite hardware
```

The Rust bindings build with Cargo, against either an installed C library or
the copy pinned as a submodule under `pipewire-flow-sys/vendor/`:

```sh
git clone --recurse-submodules https://github.com/pipewire-flow/pipewire-flow-rs
cargo build --features vendored
cargo test --workspace --features vendored
```

That runs only the tests that need no daemon. The ones that do are ignored by
default and run against a live daemon with a null sink and a null source,
which CI creates:

```sh
cargo test -p pipewire-flow --test smoke -- --ignored --test-threads=1
```

Set `PWF_SMOKE_SINK` or `PWF_SMOKE_SOURCE` to point them at other nodes.

The committed bindings have to match what bindgen produces today, or the
`bindings` job fails. Two things move them: a change to the C headers, and a
change to the bindgen version itself, since a new release can generate
different code from identical headers. Regenerate either way:

```sh
PIPEWIRE_FLOW_SYS_UPDATE_BINDINGS=1 cargo build -p pipewire-flow-sys
```

On a machine with no PipeWire, add `PIPEWIRE_FLOW_SYS_HEADERS_ONLY=1` to read
the submodule's headers and skip linking.

Dependabot raises the bindgen requirement but never regenerates, so its pull
request needs that command run on its branch before it can merge. bindgen
0.73 dropped `Copy` and `Clone` from the opaque handle types, which is the
shape this takes: mechanical, and worth reading before accepting.

## Sending a change

`main` is protected in both repositories: it takes no direct pushes, and a
pull request merges only once CI is green. Branch from `main`, open a pull
request, and let the checks run. Merged branches are deleted automatically.

CI runs the full test suite, and for the C library also builds it under
AddressSanitizer and UndefinedBehaviorSanitizer. A pull request that changes
a public C signature gets a comment saying so — that is a prompt to check the
change is deliberate, not a failure.

For the Rust crates, one job packages `pipewire-flow-sys` and builds the
extracted copy. That is what catches a build reaching for a file the
published crate would not carry, so keep anything the build reads inside the
crate directory.

## Commit messages

Subjects are `area: Imperative sentence`, where the area is either the kind
of change (`build`, `ci`, `docs`, `tests`, `examples`, `api`) or the
subsystem touched (`stream`, `filter`, `video`, `log`, `utils`, `sys`). No trailing
period.

```
docs: Correct what the routing setters actually refuse
video: Add H.264 encoded video format support
```

Write the body in full sentences, and use it to say why the change is right
rather than to restate the diff. Keep it short.

## Documentation

Public C headers carry Doxygen comments, and a test enforces their shape:
`@brief` on everything, `@return` on every non-void function, `@param` names
matching the declaration, out-parameters after in-parameters, and every
`@see` resolving to a real symbol. `meson test` runs that check, so a header
that drifts fails the build like any other test.

## Versioning

The C library and the two Rust crates version independently, because a change
to the Rust interface and a change to the C API are different events.
`pipewire-flow-sys` names the C release it pins as build metadata, as in
`0.3.0+pwf0.13.0`.

**Moving the pinned C release bumps the crate version too.** Build metadata
is ignored when versions are compared, so `0.3.0+pwf0.13.0` and
`0.3.0+pwf0.14.0` are the same version to Cargo and to crates.io, and the
second could never be published. Raise the patch alongside the pin. The
metadata records which C release a crate carries; it never distinguishes two
releases on its own.

Changing the pin also means regenerating the committed bindings, since the
`bindings` job compares them against the pinned headers.

### Tags

A release is tagged on the commit it was cut from. The C library and the
crate a reader is most likely to depend on take a bare version; the
supporting crate carries its name, so the two are told apart at a glance.

| What | Tag |
| --- | --- |
| pipewire-flow (the C library) | `v0.13.0` |
| pipewire-flow (the crate) | `v0.3.0` |
| pipewire-flow-sys | `pipewire-flow-sys-v0.3.0+pwf0.13.0` |

The `-sys` tag keeps its build metadata, so the C release a crate shipped is
readable from the tag list without opening anything.

Tags go on at publication rather than ahead of it. A tag names the commit a
release was cut from, and there is nothing to name until one exists.

## License

Contributions are accepted under the MIT license, matching both projects.
