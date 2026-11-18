# Deployment & Smoke Test

## Baseline

- Upstream: https://github.com/domoticz/domoticz, stable tag `2026.4` =
  `cc668a12b6ef9334928ae1fec6a7d20876e1f9f8` (tag and development head are the
  same commit). This branch is that commit plus `tester-env` tooling only, so
  scenario patches apply directly to upstream paths.
- Fork: https://github.com/sma-rl-dev/domoticz, branch `tester-env-baseline`.
- Runtime: one C++ process with embedded SQLite; no external database, cache,
  worker or credential.

## Quick Start

```bash
./tester-env deploy   # Build editable source in Docker, start on localhost:8093
./tester-env seed     # Create the fixed household baseline over the JSON API
./tester-env verify   # Assert login, seeded devices, rooms and favorites
./tester-env status   # Show container, image, volume, URL and credentials
./tester-env logs     # Follow container logs
./tester-env stop     # Stop container, preserving the data volume
./tester-env reset    # Remove container and data volume (keeps image)
```

For isolated runs, pass a run id and port. A non-default run id suffixes the
container, compose project and volume names (never the image name):

```bash
./tester-env reset  --run-id dz-pass --port 18093
./tester-env deploy --run-id dz-pass --port 18093
./tester-env seed   --run-id dz-pass --port 18093
./tester-env verify --run-id dz-pass --port 18093
```

`scripts/rl-env` exports `IMAGE_TAG` (content-addressed over baseline + patches);
the CLI honours it and falls back to `tester-env-domoticz:dev` for manual runs.
`TESTER_ENV_BROWSER_ORIGIN` is printed by `status` and `deploy`.

## Credentials

- Host URL: http://localhost:8093/
- Browser URL from a container: http://host.docker.internal:8093/
- Username: `admin`
- Password: `domoticz`

Admin is provisioned from `DOMOTICZ_ADMIN_USERNAME` / `DOMOTICZ_ADMIN_PASSWORD`
on the fresh-database path (`main/SQLHelper.cpp`). Those env vars are ignored
once an admin exists, so a password change requires a reset.

## Build from Source (required for mutation scenarios)

`Dockerfile.tester-env` is a two-stage Debian bookworm build:

- It fetches the five upstream submodules at pinned SHAs
  (`extern/jsoncpp` 89e2973c, `extern/jwtcpp` 3e037df3, `extern/libwebem`
  dde28f23, `extern/minizip` aee7fbdd, `extern/sqlite-amalgamation` 66f3bd93)
  instead of copying them, so builds also work from a git worktree where
  `extern/` is absent (`.dockerignore` excludes `**/extern/`).
- It bundles sqlite/jsoncpp/minizip/jwt-cpp
  (`-DUSE_BUILTIN_*=YES`) and disables Python plugins, OpenZWave, USB, I2C and
  SPI, plus the updater (`-DDISABLE_UPDATER=YES -DBUILD_MASTER=YES`).
- `/build` is a BuildKit cache mount, so an ordinary source mutation recompiles
  only the changed translation units instead of the whole tree.

Measured on this host: cold build 2m44s; comment-only edit in
`main/WebServerCmds.cpp` rebuild 21.8s; cached redeploy 2.3s; reset + deploy +
verify 6.6s.

Server flags: `-www 8080 -wwwbind 0.0.0.0 -sslwww 0 -nomdns -noupdates -nomcp
-approot /opt/domoticz -userdata /data`. SQLite database and all mutable state
live in the `domoticz-data` named volume.

## Reset

```bash
./tester-env reset
```

`docker compose down -v` for this app's compose project only: removes the
container and the `domoticz-data` volume, keeps the image. Never touches the
shared Chromium browser containers or other apps' resources.

## Deterministic Seed Data

```bash
./tester-env reset
./tester-env deploy
./tester-env seed
./tester-env verify
```

`seed` is idempotent and repairs drift: it reconciles by name through the
authenticated JSON API (`/json.htm`), reusing rows that already exist.

- Hardware: one enabled Dummy hardware (htype 15) `Maple Street Hub`.
- Devices (5): `Living Room Lamp` On, `Kitchen Ceiling Light` Off,
  `Porch Light` On, `Outdoor Temperature` 18.5 C, `Hallway Thermostat`
  setpoint 21.0.
- Room plans: `Living Room` (2 devices), `Kitchen` (1), `Hallway` (2).
- Dashboard favorites: `Living Room Lamp` and `Outdoor Temperature`.

`verify` asserts page 200, admin login, authenticated `getconfig`, `/app/app.js`,
the Dummy hardware row, the exact five-device inventory with those states and
favorite flags, and the plan device counts and Living Room membership. Two
isolated `reset/deploy/seed/verify` cycles matched fingerprints; two main-stack
cycles reproduced the same browser state.

## Smoke Test

1. Open http://localhost:8093/ (or http://host.docker.internal:8093/ from a
   container) and log in as `admin` / `domoticz`; the Dashboard loads.
2. Fresh database: Setup > Hardware is empty (`No data available in table`,
   `Showing 0 to 0 of 0 entries`). Add an enabled Dummy
   `Household Virtual Hub`; it appears as ID 2 with 1 entry and survives a
   browser refresh (browser-checker session
   `ses_e0fe0b51dffee0mEAt0Q45NCnX`).
3. After `reset` + `deploy`, a fresh navigation/login again requires login and
   Hardware shows 0 entries; the previous marker is absent (same session).
4. After `seed`: Dashboard shows exactly `Living Room Lamp` On and
   `Outdoor Temperature` 18.5 C; Devices shows 5 with the fixed values;
   Hardware shows 1 enabled `Maple Street Hub`; Roomplan shows Living Room 2,
   Kitchen 1, Hallway 2 with matching membership; the Living Room dashboard
   filter lists both favorites. Reproduced on both seeded cycles (same
   browser-checker session; seed-engineer session
   `ses_e0fe0662affe1uvsoS83vWitDU`).

## Runtime Checkpoint

Last verified deployment checkpoint: the seeded main stack was stopped with
`./tester-env stop` (compose stop; container `tester-env-domoticz` Exited(0),
`:8093` not listening, `status` reports NOT RUNNING) with the `domoticz-data`
volume and `tester-env-domoticz:dev` image retained. `deploy` restarts it
against that seeded volume; `reset` returns to the empty database.

## Tooling Notes

- Browser-visible verification only; no source, database or log access is a
  valid oracle for a scenario result.
- Python plugin scripting is compiled out (`-DUSE_PYTHON=NO`); Lua and dzVents
  remain available.
- The build context excludes `.git`, so Build Hash/Date report `0/1970`;
  `BUILD_MASTER=YES` keeps the UI version at `V2026.4-stable`.
- The log warns `Default admin password has NOT been changed` on every start;
  expected for this training baseline.
- Optional integrations (USB/I2C/SPI, OpenZWave) must stay disabled in
  scenarios; only Dummy hardware is used.
