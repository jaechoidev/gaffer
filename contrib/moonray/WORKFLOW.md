# Workflow: Pattern A — Claude on host, container for builds

Operational runbook for the OpenMoonRay → Gaffer port project. Read
`PLAN.md` for the project plan; this document is the day-to-day "how"
on macOS hosts.

## Architecture

- **Two Claude Code sessions on the host**, one per repo. They edit
  files natively on the host filesystem.
- **A persistent Linux build container** (`ghcr.io/gafferhq/build/build:3.4.0`)
  named `gaffer-build`, running in the background, with the host work
  directory bind-mounted at `/work` inside the container.
- **A `dexec` wrapper script** (`contrib/moonray/scripts/dexec`) that
  the Claude sessions and the user invoke for build, test, and CMake
  commands.

```
+----------------------+         +--------------------------------+
| Host terminal 1      |         | Host terminal 2                |
| claude (in gaffer/)  |         | claude (in openmoonray/)       |
| edits .cpp .py etc.  |         | edits CMakeLists.txt, patches  |
+-----------+----------+         +--------------+-----------------+
            |                                   |
            | dexec scons build ...             | dexec cmake --build ...
            v                                   v
+----------------------------------------------------------------+
| Container: gaffer-build (Linux x86_64, Gaffer's CI deps)       |
|   /work/gaffer       <--->   <host>/work/gaffer                |
|   /work/openmoonray  <--->   <host>/work/openmoonray           |
|   /work/_install     <--->   <host>/work/_install              |
+----------------------------------------------------------------+
```

## File layout (host)

```
<host>/work/
├── gaffer/                       # Gaffer fork (jaechoidev/gaffer)
│   └── contrib/moonray/
│       ├── PLAN.md               # The plan
│       ├── WORKFLOW.md           # This file
│       ├── CLAUDE.md.template    # Copy to openmoonray/CLAUDE.md
│       ├── bootstrap-gaffer.txt  # Paste into Gaffer Claude session
│       ├── bootstrap-moonray.txt # Paste into Moonray Claude session
│       └── scripts/
│           └── dexec             # Container exec wrapper
├── openmoonray/                  # Moonray fork (jaechoidev/openmoonray)
│   ├── CLAUDE.md                 # Auto-loaded by Claude in this repo
│   └── ... (Moonray source)
└── _install/                     # Build outputs (persisted on host)
    ├── moonray-gaffer/           # MOONRAY_ROOT consumed by Gaffer
    └── gaffer/                   # Gaffer install (optional)
```

`_install/` lives under `/work/` so that `MOONRAY_ROOT` artifacts
survive container restarts (anything outside `/work/` lives only in the
container's writable layer and is lost on `docker rm`).

## Initial setup (do once)

Replace `<host work dir>` with your actual path
(e.g. `/Users/jaechoi/Documents/workspace/repo/work`).

### 1. Stop any earlier interactive container

If you ran `docker run -it --rm ghcr.io/...` earlier, exit it. We're
switching to a named persistent container.

### 2. Launch the persistent build container

```bash
docker run -d --name gaffer-build \
  --platform linux/amd64 \
  -v <host work dir>:/work \
  -w /work \
  ghcr.io/gafferhq/build/build:3.4.0 \
  sleep infinity
```

- `-d` — detached / background
- `--name gaffer-build` — stable name `dexec` looks for
- `--platform linux/amd64` — for Apple Silicon hosts; drop on Intel/Linux
- `sleep infinity` — keeps the container alive without a foreground process

Verify:
```bash
docker ps --filter name=gaffer-build
# STATUS column should show "Up <time>"
```

### 3. Install the `dexec` helper on PATH

Append one line to `~/.zshrc` (or `~/.bashrc`):

```bash
export PATH="<host work dir>/gaffer/contrib/moonray/scripts:$PATH"
```

Reload: `source ~/.zshrc`

Test:
```bash
dexec hostname        # prints container hostname
dexec ls /work        # expect: gaffer  openmoonray
```

### 4. Drop CLAUDE.md into the Moonray fork

```bash
cp <host work dir>/gaffer/contrib/moonray/CLAUDE.md.template \
   <host work dir>/openmoonray/CLAUDE.md
cd <host work dir>/openmoonray
git checkout -b feature/gaffer-deps-port  # if not already on it
git add CLAUDE.md
git commit -m "Add CLAUDE.md for Gaffer-port soft fork"
git push -u origin feature/gaffer-deps-port
```

## Daily workflow

### Starting a working session

1. Verify container is up (start if needed):
   ```bash
   docker ps --filter name=gaffer-build
   docker start gaffer-build       # if it was stopped
   ```

2. **Terminal 1** (Gaffer side):
   ```bash
   cd <host work dir>/gaffer
   claude
   ```
   First message: paste the contents of `contrib/moonray/bootstrap-gaffer.txt`.

3. **Terminal 2** (Moonray side):
   ```bash
   cd <host work dir>/openmoonray
   claude
   ```
   First message: paste the contents of `../gaffer/contrib/moonray/bootstrap-moonray.txt`.

### Build and test commands

Always go through `dexec`. Examples:

```bash
# Moonray side: configure and build
dexec bash -c 'cd /work/openmoonray && cmake --preset linux'
dexec cmake --build /work/openmoonray/build --parallel

# Gaffer side: build with MOONRAY_ROOT
dexec bash -c 'cd /work/gaffer && MOONRAY_ROOT=/work/_install/moonray-gaffer scons build'

# Run a test
dexec bash -c '/work/_install/gaffer/bin/gaffer test GafferMoonrayTest'

# Drop into an interactive shell inside the container (no args = bash)
dexec
```

### Pausing / resuming

```bash
docker stop gaffer-build         # before laptop sleep, optional
docker start gaffer-build        # resume
```

State inside `/work` is preserved across stop/start (it's a host bind
mount). State elsewhere in the container persists across stop/start
but is lost on `docker rm`.

### Tear down

```bash
docker stop gaffer-build && docker rm gaffer-build
```

## Troubleshooting

**`dexec: command not found`** — PATH not picking up the scripts dir.
Verify with `echo $PATH | tr ':' '\n' | grep moonray`. Re-source
`~/.zshrc`.

**`dexec: container 'gaffer-build' is not running`** — start it:
`docker start gaffer-build`.

**Slow builds on Apple Silicon** — expected; the image is x86_64 and
runs under Rosetta/QEMU emulation. M-series builds typically run
~3–5× slower than native Linux. Tolerable for Phase 0/1; for Phase 2+
interactive testing, plan time on a real Linux box.

**File permission oddities** — files written inside the container
appear as `root:root` from inside but with your host user from
outside (Docker Desktop bind-mount handling on macOS). Usually fine;
don't `chown` unless something concretely breaks.

**Moonray install is gone after `docker rm`** — install to
`/work/_install/...` instead of `/opt/...` so it lives on the host
filesystem and survives container teardown.
