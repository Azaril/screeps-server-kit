# screeps-server-kit

> Spin up a local Screeps private server, bootstrap a world, deploy your bot, and watch it run — from one Rust CLI.

`screeps-server-kit` is a local private-server toolkit for Screeps bot development. It drives a Dockerized [screeps-launcher](https://github.com/screepers/screeps-launcher) stack end to end: bringing the containers up, bootstrapping a fresh world (users, passwords, tick rate, spawn placement), controlling ticks, deploying a WASM bot, and capturing console/metrics output. It is **mechanism, not policy** — everything here is bot-agnostic, so the same library code backs both an operator CLI and automated smoke/soak loops. It is a component extracted from the [screeps-ibex](https://github.com/Azaril/screeps-ibex) workspace, where the bot-specific evaluation policy lives in a separate consumer crate.

The crate ships as both a library (`screeps_server_kit`) and an operator CLI (`screeps-server-kit`). Every CLI subcommand is a thin wrapper over a library function, so anything you can do by hand you can also script.

## Installation

As a library, add it as a git dependency:

```toml
[dependencies]
screeps-server-kit = { git = "https://github.com/Azaril/screeps-server-kit" }
```

As a CLI, install the binary:

```bash
cargo install --git https://github.com/Azaril/screeps-server-kit
```

Or run it from a source checkout (the form used throughout this README):

```bash
cargo run -- <subcommand> [flags]
```

## Getting Started

### Prerequisites

- **Docker Desktop**, running. The kit talks to the local Docker daemon directly (via [bollard](https://crates.io/crates/bollard)) — no `docker compose` required.
- **A Rust toolchain** with the `wasm32-unknown-unknown` target (needed for `deploy`, which builds your bot to WASM).
- **A Steam Web API key** (https://steamcommunity.com/dev/apikey) — the Screeps backend cannot start without one.
- **Two config files** (details below): a repo-root `.screeps.yaml` for credentials and a crate-local `config/local.yml` for stack settings.

### Configure the two files

The kit reads from **fixed paths** (no env vars, no directory walking):

1. **`../.screeps.yaml`** (next to the crate, gitignored) holds server *credentials* under `servers:` — the same unified file that `screeps-pack` and `screeps-prospector` read. The default acting identity is the `private-server` entry:

   ```yaml
   servers:
     private-server:
       host: 127.0.0.1
       port: 21025
       username: ibex
       password: choose-a-password
   ```

2. **`config/local.yml`** (crate-local, gitignored) holds *stack settings* — the Steam key, ports, tick rate, spawn placement, and the list of bots to bootstrap. Copy the committed template and edit it:

   ```bash
   cp config/local.example.yml config/local.yml
   ```

   At minimum, set your `steamKey`. Every other key is optional and documented inline in `config/local.example.yml`. See [Configuration](#configuration) for the full reference.

Verify what the kit resolved (secrets are redacted by construction):

```bash
cargo run -- config
```

### Bring up a server, bootstrap, deploy, and watch

```bash
# 1. Start the container stack (launcher + mongo + redis) and wait for the
#    game API. First boot npm-installs the server in-container (~10 min);
#    warm restarts are fast.
cargo run -- server up

# 2. Reset/initialize the world: register users, set passwords, apply the
#    tick rate, and place one spawn per configured bot in a distinct room.
cargo run -- bootstrap --reset

# 3. Build your bot to WASM and upload it to the server.
cargo run -- deploy

# 4. Tail the bot's live console output (Ctrl-C to stop).
cargo run -- console --user ibex
```

Open the web client to watch in the browser:

```bash
cargo run -- open      # prints http://127.0.0.1:21025/ and tries to launch it
```

When you are done, stop the stack but keep the world data for a warm restart:

```bash
cargo run -- server down
```

## Common Use Cases

### Start and stop the stack

```bash
cargo run -- server up        # create/start containers, wait for the game API
cargo run -- server status    # container table + live game-API and CLI probes
cargo run -- server down      # stop containers; keep data (warm next `up`)
cargo run -- server destroy --yes   # remove containers, network, AND volumes (all world data lost)
```

### Bootstrap a fresh world

`bootstrap` converges the world to match your config: it registers each `bots:` entry (or converges its password if it already exists), signs in to verify credentials, re-applies the tick rate, places one spawn per bot in a distinct room, and raises each bot's GCL so its expansion logic can run.

```bash
cargo run -- bootstrap            # converge users/spawns without wiping
cargo run -- bootstrap --reset    # wipe all world data first (system.resetAllData), then bootstrap
```

The first bot's spawn honors the optional `spawn:` block in `config/local.yml`; later bots auto-pick distinct rooms. Set `spawnPlacement: prospector` to route placement through the [screeps-prospector](https://github.com/Azaril/screeps-prospector) base-planning pipeline.

### Set or converge a bot's password

A 401 on deploy means the server-side password drifted from `.screeps.yaml`. `bootstrap` converges it automatically, or you can do it directly through the world CLI:

```bash
cargo run -- cli "setPassword('ibex', 'password-from-.screeps.yaml')"
```

`setPassword` arguments are masked in all echoed output and logs.

### Deploy a build and run ticks

```bash
cargo run -- deploy                  # release build, default identity
cargo run -- deploy --debug          # debug build (no --release)
cargo run -- deploy --user ibex-2    # deploy as a specific .screeps.yaml entry

cargo run -- tick set 50             # fast-forward: 50 ms/tick (the floor; values below 100 warn, below 50 are rejected)
cargo run -- tick pause              # pause the simulation
cargo run -- tick resume             # resume
```

### Capture or tail console output

```bash
# Live tail (no artifacts), optionally time-boxed and filtered:
cargo run -- console --user ibex --grep Squad --seconds 60

# Run a single JS expression once in the BOT's runtime (Game/Memory):
cargo run -- exec --user ibex "delete Memory._features"

# Run a single admin command against the world CLI (port 21026):
cargo run -- cli "system.getTickDuration()"

# Interactive admin REPL:
cargo run -- cli
```

For recorded console + metrics artifacts (a `console.jsonl` / `metrics.jsonl` / `summary.json` run), use the `capture::run` library entry point from a consumer crate — see [Usage](#usage).

### Spawn an offense-soak target or NPC bots

Use the world CLI passthrough to drive the server's admin objects directly. To create a real combat target with ramparts and towers, or to add an NPC AI opponent:

```bash
# Spawn an invader-core stronghold (the combat-soak driver):
cargo run -- cli "strongholds.spawn('W4N5', { templateName: 'bunker3' })"

# Spawn an NPC AI bot in a room:
cargo run -- cli "bots.spawn('screeps-bot-tooangel', 'W7N4')"

# Discover available admin commands live:
cargo run -- cli "help(strongholds)"
cargo run -- cli "help(bots)"
```

> Note: on a private server, neutral/NPC rooms default to `active=false` and are **frozen** (cores never deploy, creeps never act). After spawning a target in a neutral room, set the rooms active and **restart** the stack: `cargo run -- cli "storage.db.rooms.update({active:false},{\$set:{active:true}},{multi:true})"` then `server down` && `server up`.

## Configuration

Two files, fixed paths, no environment variables and no directory walking:

| File | Path | Holds | Tracked? |
|---|---|---|---|
| credentials | `../.screeps.yaml` (the parent directory — your bot repo's root; `--config` is the only override) | `servers:` entries only — one per identity (bots **and** official servers); shared with your deploy pipeline | gitignored |
| stack settings | `config/local.yml` | `steamKey`, ports, `tickMs`, `gcl`, spawn preference, the `bots:` list, the `image:` block | gitignored; every key documented in the committed `config/local.example.yml` |

The kit reads **only** `servers:` from `.screeps.yaml`; other sections (e.g. build configs) are ignored.

### `config/local.yml`, fully populated

```yaml
steamKey: your-steam-web-api-key-here   # REQUIRED before `server up`
                                        # (https://steamcommunity.com/dev/apikey)
ports:
  game: 21025   # game/API port (published host-side and bound in-container)
  cli: 21026    # server CLI port (likewise)
tickMs: 100     # written to serverConfig.tickRate; floor 50
gcl: 10         # GCL level each bot is raised to in bootstrap (raise-only;
                # 1 disables — lifts the owned-room cap so bots can expand)
spawn:          # first bot's spawn preference (optional; see "Bootstrap a fresh world")
  room: W5N3
bots:           # bot identities to bootstrap; each is a servers: entry
  - ibex        # in ../.screeps.yaml
  - ibex-2
image:          # launcher image policy (optional; see "Building the launcher image")
  name: screepers/screeps-launcher:latest
```

`config/local.yml` is optional for read-only commands (`config`, `open`, `cli`, `tick` against a running stack use the defaults), but `server up`/`bootstrap` need the `steamKey` — the error points here when it is missing. A typo'd key is a hard parse error (unknown fields are rejected), never a silently-ignored setting.

### Operator identity vs bot identities

**You are yourself; the bots are their own users.**

- **The operator** logs in as a *person*: the Steam client or the web client at `http://127.0.0.1:21025/` — Steam auth works because the server runs with your `steamKey`, and your Steam persona becomes your in-game user the first time you sign in. If you also want password/API access to that user, set one through the masked CLI passthrough: `cargo run -- cli` then `setPassword('YourName', 'your-password')` (the password is masked in every echo/log).
- **Each bot** is a `servers:` entry in `../.screeps.yaml` with its own username/password — registered by `bootstrap`, targeted by `deploy --user <entry>`. **Name bot entries after the bot (`ibex`, `ibex-2`, …), not after yourself**, so your own name stays the operator identity. This supports bot-vs-bot matches and playing alongside your bot on one world.

Server-level commands (`deploy`, `config`, capture) act as **one** entry — an explicit `--server-name` if given, otherwise the **first `bots:` entry** (falling back to `private-server` when no bots are configured). `bootstrap` registers only the `bots:` entries, so pointing the acting identity anywhere else is a guaranteed 401 on a fresh world.

```yaml
# ../.screeps.yaml — one entry per bot identity
servers:
  ibex:
    host: 127.0.0.1
    port: 21025
    secure: false
    username: ibex
    password: <bot password>
    branch: default
  ibex-2:
    host: 127.0.0.1
    port: 21025
    secure: false
    username: ibex-2
    password: <bot password>
    branch: default
```

With `bots: [ibex, ibex-2]` in `config/local.yml`, `bootstrap --reset` registers both users and places two spawns in distinct rooms; `deploy --user ibex` / `deploy --user ibex-2` push code independently.

### Building the launcher image (optional)

```bash
cargo run -- server build-image
```

By default the launcher image is **pulled** (`screepers/screeps-launcher:latest`). To build it locally instead (the hermetic option), point `image.build.context` at a **full clone of [screepers/screeps-launcher](https://github.com/screepers/screeps-launcher)** — the Dockerfile lives at that repo's root. `server build-image` builds and tags it (`image.name`); `server up` also builds automatically when `build:` is configured and the image is absent. The build runs through BuildKit (the upstream Dockerfile is BuildKit-only).

> On Windows, clone the launcher repo with `git clone -c core.autocrlf=false …` — a CRLF checkout breaks the Dockerfile's bash heredocs. A launcher *deployment* directory (just `config.yml` + `docker-compose.yml`) is **not** a buildable context.

## Usage

The library exposes every CLI capability as a function. A minimal program that brings the stack up, bootstraps the world, and deploys a bot:

```rust
use screeps_server_kit::config::KitConfig;
use screeps_server_kit::{deploy, docker, server};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // Resolve config from the fixed paths (../.screeps.yaml + config/local.yml).
    // `None, None` = default credentials path and default acting identity.
    let cfg = KitConfig::load(None, None)?;

    // Bring the container stack up and wait for the game API.
    docker::up(&cfg.stack).await?;

    // Reset and initialize the world (register users, place spawns, set tick rate).
    let outcome = server::bootstrap(&cfg, /* reset = */ true).await?;
    println!("{outcome}");

    // Build the bot to WASM and upload it as the kit's acting identity.
    let report = deploy::deploy(&cfg, &cfg.server_name, /* debug = */ false).await?;
    println!("{report}");

    Ok(())
}
```

For recorded runs, `capture::run` drives a tick-bounded capture loop and writes JSONL artifacts. It takes a `capture::CaptureSpec` so the caller (not the kit) decides what a panic line, a deserialization failure, or the live-stats memory segment is — keeping the kit bot-agnostic:

```rust
use screeps_server_kit::capture::{self, CaptureSpec, MarkerSpec};
use screeps_server_kit::config::KitConfig;

async fn capture_a_run() -> anyhow::Result<()> {
    let cfg = KitConfig::load(None, None)?;
    let spec = CaptureSpec {
        markers: MarkerSpec {
            panic_markers: vec!["panicked at".to_string()],
            ..Default::default()
        },
        stats_segment: Some(0), // memory segment carrying your bot's live stats
        ..Default::default()
    };

    let artifacts = capture::run(&cfg, /* ticks = */ 600, "smoke", &spec).await?;
    println!("{}", artifacts.summary);
    Ok(())
}
```

## CLI

Global flags (valid on any subcommand):

- `--config <path>` — override the credentials file path (default: `../.screeps.yaml` next to the crate).
- `--server-name <name>` — the `servers:` entry the kit acts as (default: the first `bots:` entry, falling back to `private-server`).

| Subcommand | Description |
|---|---|
| `server up` | Pull/create/start the launcher + mongo + redis containers and wait until the game API answers. First boot is slow (in-container npm install). |
| `server down` | Stop the containers, keeping containers/network/volumes (next `up` is a warm restart). |
| `server destroy --yes` | Remove containers, network, **and volumes** — all world data is lost. |
| `server status` | Container table (state, health, discovered published ports) plus live game-API and CLI-port probes. |
| `server build-image` | Build the launcher image from `config/local.yml`'s `image.build` context (a full screeps-launcher clone). |
| `server logs [-f/--follow] [--tail N]` | Print (or stream) the launcher container's logs (default tail 100). |
| `bootstrap [--reset]` | Reset/initialize the world: users, passwords, tick rate, spawn placement (one spawn per `bots:` entry, distinct rooms). `--reset` wipes all data first. |
| `deploy [--debug] [--user <entry>]` | Build the bot via `screeps-pack` (cargo build → wasm-bindgen → glue → wasm-opt → upload). `--debug` for a dev build; `--user` selects the `.screeps.yaml` identity. |
| `cli [<command>]` | Passthrough to the world server CLI. With a command argument, send it once; without, open an interactive REPL. |
| `tick set <ms>` | Set the tick duration in milliseconds (floor 50; warns below 100). |
| `tick pause` / `tick resume` | Pause / resume the simulation. |
| `console [--user <entry>] [--seconds N] [--grep <substr>]` | Live-tail a bot's console over the websocket (identity defaults to `--server-name`). `--seconds` time-boxes it; `--grep` keeps only matching lines (case-insensitive). |
| `exec [--user <entry>] <expression>` | Run a single JS expression once in a bot's runtime console (`POST /api/user/console`); identity defaults to `--server-name`. |
| `open` | Print (and try to launch) the web-client URL. |
| `config` | Show the resolved configuration (secrets redacted). |

> `run` and `smoke` evaluation commands are **not** in this binary — they are bot-specific policy and live in the consumer crate (e.g. `screeps-ibex-eval`), which drives the same `screeps_server_kit::capture::run` library function.

## How it works

The kit manages a three-container stack — the screeps-launcher image plus `mongo:8` and `redis:7` — natively through the Docker API. All objects are named `screeps-eval-*` so the stack is recognizable and `destroy` cannot touch anything else. On `up`, it merges a vendored keyless launcher-config template with your `config/local.yml` settings (forcing the game/CLI binds to `0.0.0.0` so the host can reach them, and injecting the Steam key), writes the merged config to a gitignored `target/runtime/` location, and bind-mounts it into the launcher.

Two protocols drive the running server: the **world server CLI** (an HTTP endpoint on port 21026 that evaluates JavaScript in a Node `vm` sandbox — used for `resetAllData`, `setPassword`, tick control, and direct DB queries) and the **game REST/websocket API** (port 21025 — used for registration, signin, spawn placement, console streaming, and metrics). The game-API plumbing lives in the shared [screeps-rest-api](https://github.com/Azaril/screeps-rest-api) crate; deploy is a library call into [screeps-pack](https://github.com/Azaril/screeps-pack).

Credentials are wrapped in `SecretString` from the moment they are parsed, so `Debug`/`Display` redact them by construction. The one credential-bearing command — the world CLI's `setPassword(...)` — is masked on every echo, log, and error path (the `vm` echoes command source in stack traces, so even error bodies are masked).

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `connecting to Docker — is Docker Desktop running?` | Start Docker Desktop and wait for the engine, then re-run. |
| `server up` seems stuck after image pulls | First boot npm-installs the server in-container (~10 min budget). Progress lines show the launcher's latest log every 15 s; watch detail with `cargo run -- server logs -f`. |
| `no Steam key available: ...` | Set `steamKey` in `config/local.yml` (copy `config/local.example.yml`; key from <https://steamcommunity.com/dev/apikey>). |
| `reading credentials file ...\..\.screeps.yaml` | The parent directory's `.screeps.yaml` is missing — create it (see [Configuration](#configuration)). The path is fixed (`--config` is the only override); there is no directory walking. |
| `config/local.yml has an unexpected shape` | A typo'd key (the parser rejects unknown fields by design). Compare against `config/local.example.yml` — keys are camelCase (`steamKey`, `tickMs`). |
| `resolving bots entry '<name>'` | Every `bots:` name in `config/local.yml` must be a `servers:` entry in `../.screeps.yaml` with `username`+`password` (token-auth entries cannot be bots). |
| `image.build.context ... has no Dockerfile` from `build-image` | The context is a launcher *deployment* dir (config + compose), not the launcher *source* repo. Clone <https://github.com/screepers/screeps-launcher> and point `image.build.context` at the clone root. |
| `image build failed: ... exit code: 2 ([stage-1 2/4] RUN <<-EOT bash)` | CRLF checkout: the Dockerfile's bash heredocs break under Windows line-ending conversion. Re-clone with `git clone -c core.autocrlf=false`. |
| Port already in use (create/start error mentioning `0.0.0.0:21025`) | Another server holds the port. Stop it, or set `ports.game/cli` in `config/local.yml` to free ports, then `server destroy --yes` and `server up`. |
| Changed `ports` but `status` shows the old ones | Published ports are fixed at container creation; `up` warns about this. Run `server destroy --yes` then `server up`. |
| `server CLI tcp://127.0.0.1:21026 -> refused/timeout` in `status` | The CLI bind is forced to `0.0.0.0` in the merged config, but if you're running a foreign/manually-started stack (which binds in-container `127.0.0.1` by default and publishes no CLI port), use the fallback: `docker exec -it screeps-eval-launcher screeps-launcher cli` (substitute the container name from `status`). |
| World/server behaving oddly after config experiments; `mongosh`/auth errors in logs | Stale volumes from an earlier configuration. Factory-reset: `server destroy --yes` then `server up` (full first boot again). |
| `launcher container exited while waiting for the API` | The error includes the last launcher logs. Typical causes: no/invalid steam key in `config/local.yml`, mongo/redis not healthy (re-run `server up`), or a broken locally-built image (re-run `server build-image`). |
| `signin as '<user>' rejected (401)` | The server-side password differs from `.screeps.yaml` (e.g. the entry changed since the user was created). Run `bootstrap` — it converges the password via `setPassword` before signing in. |
| `registering user ... failed: Registration is automatically disabled` | The server has the `SERVER_PASSWORD` env var set (screepsmod-auth closes registration). Not set by this kit. |
| `spawn.room ... is not a valid first-spawn room` | The room lacks an unowned controller or 2 sources, or was claimed by an earlier bot (the error lists valid candidates), or the world was seeded with a different map. Pick a listed room or drop `spawn.room` for auto-pick. |
| Simulation racing (hundreds of ticks/s) after a manual CLI `system.resetAllData()` | A reset flushes redis, including the tick duration — the loop runs unthrottled. `tick set 100` (or re-run `bootstrap`, which always re-applies `tickMs`). |
| `cargo build failed` from `deploy` | The real compiler error is in the streamed output just above. Common toolchain gaps: nightly not installed (`rust-toolchain.toml` pins it), missing `rust-src` component (build-std needs it), missing `wasm32-unknown-unknown` target. |
| `wasm-bindgen ... emitted JS whose wasm-load tail does not match` from `deploy` | The bot's `Cargo.lock` moved to a wasm-bindgen version whose output [screeps-pack](https://github.com/Azaril/screeps-pack)'s anchored glue patcher has not been verified against. |
| `uploading ... modules to ...` failed | The upload threw (server down/unreachable, or signin rejected). Run `server status`, then retry; `bootstrap` converges a stale password. |
| `websocket auth failed (token rejected)` during a capture run | The signin token was rejected — usually a stale server-side password. Run `bootstrap` to converge credentials, then retry. |
| `run did not reach tick ... within the ... safety budget` | The simulation is paused (`tick resume`) or crawling far below the configured rate. Check `server status` and the tick rate. |
| `console.jsonl` is empty/small for short runs | Normal if the bot logs sparsely and empty per-tick console events are not written. CPU/creeps still prove liveness in `metrics.jsonl`. |

## Related crates

- [screeps-rest-api](https://github.com/Azaril/screeps-rest-api) — the shared Screeps REST/websocket client (auth, pinned endpoints, console socket) the kit's API access is built on.
- [screeps-pack](https://github.com/Azaril/screeps-pack) — the WASM deploy pipeline (cargo build → wasm-bindgen → glue → wasm-opt → upload) that `deploy` drives as a library.
- [screeps-prospector](https://github.com/Azaril/screeps-prospector) — base/spawn-site selection; the `spawnPlacement: prospector` bootstrap mode routes first-spawn placement through it.
