# hippocampus-otel-collector

An OpenTelemetry Collector **logs exporter** that writes each log record into
[Hippocampus](https://github.com/fastbean-au/hippocampus) as a memory — and a ready-to-build
collector distribution that ships it.

Logs are the case the service was built for. A collector batch is thousands of records of which a
handful matter; the exporter maps severity onto significance and lets Hippocampus's decay cycle
forget the rest, so what survives is the part somebody would have kept.

| Directory | What it is |
| :--- | :--- |
| [`hippocampusexporter/`](hippocampusexporter/README.md) | The exporter component: configuration, severity→significance mapping, metadata selection, and the batch write. Add it to your own collector build. |
| [`collector/`](collector/README.md) | An [OCB](https://github.com/open-telemetry/opentelemetry-collector-releases) manifest, a runnable demo config, and the Dockerfile that produces `ghcr.io/fastbean-au/hippocampus-otel-collector`. |

## Quick start

```bash
docker run --rm ghcr.io/fastbean-au/hippocampus-otel-collector:latest components
```

To build it yourself, from this directory:

```bash
go install go.opentelemetry.io/collector/cmd/builder@v0.160.0   # must match otelcol_version
cd collector && builder --config builder-config.yaml
./_build/hippocampus-otelcol --config config.yaml
```

See [`collector/README.md`](collector/README.md) for the end-to-end walkthrough, and
[`hippocampusexporter/README.md`](hippocampusexporter/README.md) for every configuration key.

## Why this is its own repository

It tracks somebody else's release train. The exporter depends on twelve collector modules published
in two version series (`v1.x` and `v0.x`) that move together on a fortnightly cadence, against
roughly one commit in seventeen that touched the Hippocampus contract. Inside the monorepo those
bumps arrived mixed in with the service's own, behind a `replace` that made the service dependency
invisible — which is how this module came to declare `hippocampus v0.39.0` while the service was at
`v0.47.0`, with nothing noticing.

Two things changed on the way out, and both are improvements rather than consequences:

- **The collector image is built on every push.** In the monorepo it was built *only* at release
  time, so an OCB or manifest break first surfaced after the tag existed. CI here builds it and then
  runs `components` against the result, because a manifest with a wrong module path compiles
  perfectly and ships a collector that cannot export to Hippocampus.
- **One `replace`, not two.** The OCB manifest needed a second one pointing four levels up at the
  monorepo root for the contract. The exporter now requires a released version of that module like
  any other dependency, which is also what makes the version it depends on visible.

## Versioning

The version line **continues** the service's rather than restarting: this image has been published
at the service version since it existed, so the first release cut from here is `v0.48.0`. It is free
to diverge after that, and the module it requires is tracked separately —
`.github/workflows/contract-bump.yaml` opens a pull request raising
`github.com/fastbean-au/hippocampus` whenever the service cuts a release, carrying the build result
against the new contract.

The import path changed with the move, to
`github.com/fastbean-au/hippocampus-otel-collector/hippocampusexporter`. If you reference the
exporter from your own OCB manifest, that is the line to update.

## Licence

MIT — see [LICENSE](LICENSE).
