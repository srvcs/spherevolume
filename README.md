# srvcs-spherevolume

The sphere-volume orchestrator of the srvcs.cloud distributed standard library.

Its single concern: **geometry: volume of a sphere.** It owns the *control
flow* — composing three float primitives — but does no arithmetic of its own.
It asks [`srvcs-pi`](https://github.com/srvcs/pi) for the constant, then chains
[`srvcs-floatmultiply`](https://github.com/srvcs/floatmultiply) to build
`pi * r^3 * 4`, and finally [`srvcs-floatdivide`](https://github.com/srvcs/floatdivide)
to divide by `3`.

```
spherevolume(radius):
    p  = pi()                       # constant service, called with an empty body
    r2 = floatmultiply(radius, radius)
    r3 = floatmultiply(r2, radius)
    t  = floatmultiply(p, r3)
    t2 = floatmultiply(4, t)
    return floatdivide(t2, 3)       # V = (4/3) * pi * r^3
```

The result is an `f64` — a JSON number that may be fractional. For example
`spherevolume(3) == 113.09733552923255`.

Validation is not handled here. This service never calls `srvcs-isnumber`
directly; instead its dependencies validate their own operands, and any `422`
they raise is forwarded verbatim.

## API

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/` | Service identity, concern, and dependency list |
| `POST` | `/` | Compute the volume of a sphere with the given `radius` |
| `GET` | `/healthz` `/readyz` `/metrics` `/openapi.json` | srvcs service standard surface |

```sh
curl -s -X POST localhost:8080/ -H 'content-type: application/json' -d '{"radius": 3}'
# {"radius":3,"result":113.09733552923255}
```

Responses:

- `200 {"radius": r, "result": n}` — evaluated; `result` is a float.
- `422` — a dependency rejected the input (forwarded verbatim).
- `500` — a reachable dependency returned a `200` without a numeric `result`
  (a contract violation).
- `503` — a dependency is unavailable.

## Dependencies

- [`srvcs-pi`](https://github.com/srvcs/pi)
- [`srvcs-floatmultiply`](https://github.com/srvcs/floatmultiply)
- [`srvcs-floatdivide`](https://github.com/srvcs/floatdivide)

## Configuration

| Variable | Default | Purpose |
| --- | --- | --- |
| `SRVCS_BIND_ADDR` | `0.0.0.0:8080` | Bind address |
| `SRVCS_PI_URL` | `http://127.0.0.1:8090` | Base URL of `srvcs-pi` |
| `SRVCS_FLOATMULTIPLY_URL` | `http://127.0.0.1:8091` | Base URL of `srvcs-floatmultiply` |
| `SRVCS_FLOATDIVIDE_URL` | `http://127.0.0.1:8092` | Base URL of `srvcs-floatdivide` |
| `SRVCS_ENV` | `development` | Environment label for logs |
| `RUST_LOG` | `info,tower_http=info` | Tracing filter |

## Local checks

```sh
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
```

Orchestration tests stand up *computing* mock dependency services in-process —
they read the request body and return the real `a * b` / `a / b` and the `pi`
constant, so the composition is genuinely exercised against the asserted cases
(compared approximately, since the result is a float). See
[`srvcs/platform`](https://github.com/srvcs/platform) for the shared standard.

> Note: the `cargoHash` in `flake.nix` is inherited from the template and must be
> refreshed with a `nix build` before the Nix gates pass.
