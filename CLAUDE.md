# picoquic-cc — session handoff

Compare picoquic **sender** congestion control (CUBIC / BBRv3 / NewReno) over **Starlink**.
A Raspberry Pi downloads 2 MiB and 50 MiB over HTTP/3 with curl. Metrics: curl write-out
(throughput, download time), CPU (client + server), Starlink dish status via gRPC at 1 Hz.
Folder was renamed from `mp-quic` to `picoquic-cc` on 2026-11-14.

## User rules (must follow)
- Code: only a **scientific header comment** at the top of each file, no inline comments;
  clean, senior-level architecture.
- Server setup lives in `server/`, client scripts in `scripts/`.
- **Tell the user and get approval before pushing anything to the server or the Pi.**
  Read-only SSH inspection is fine.
- Reply in Vietnamese.

## Hosts
| Role | Access | Path |
|---|---|---|
| Server (AWS t3.medium, Ubuntu 24.04, 2 vCPU) | `ubuntu@47.129.174.173` | `/data/starlink-cc/` (code in `server/`; generated `build/ www/ tls/ state/ logs/`) |
| Client (Raspberry Pi 4, Debian 13) | `trungnt@100.72.249.11` (Tailscale) | `~/starlink-cc/` (+ `.venv/`, `results/`) |

Deploy with `make deploy-server|setup-server|deploy-client|setup-client|fetch`.
`make fetch` pulls data into local `results/client` and `results/server`.

## Current state (2026-11-14): deployed and smoke-tested
- Server: picoquic built at commit `36f49b7` (Release). systemd template `picoquic@.service`
  runs three instances:
  `cubic` → `-G cubic -p 4451`, `bbrv3` → `-G bbr -p 4452` (picoquic "bbr" = BBRv3),
  `reno` → `-G newreno -p 4453`.
  `picoquic-cpu.service` logs per-cgroup CPU and steal to `logs/cpu.csv`.
  `www/manifest.json` is the single source of truth for CC→port and payload sizes; the client reads it.
- AWS Security Group: every UDP port probed (4400–4499 and others) reached the server from
  the Pi's current IP. Re-check once the Pi's traffic goes out through Starlink.
- Client: curl at `~/quic-curl/curl-http3/bin/curl` (8.19-DEV with ngtcp2 1.20, nghttp3,
  OpenSSL 3.5), forced ChaCha20. venv has grpcio 1.85 and protobuf 7.36.
- Smoke tests: `ROUNDS=1 ./run.sh run --no-dish` gave 6/6 valid. 50 MB ≈ 85 Mbps,
  2 MB ≈ 35 Mbps, client curl CPU 26–41 % of one core.
- Schedule: each round visits each payload size once (random order). Within a size block,
  all 3 CCs run back to back (random order, seed recorded). Transfers are sequential with
  `GAP_SECONDS=5` before each one; failures are kept as `valid=0` and never retried.
  Default 30 rounds = 180 transfers ≈ 25 min.

## Open issues before the real campaign
1. Pi `vcgencmd get_throttled` = `0x50005` (under-voltage and throttled now) → needs a
   proper 5V/3A PSU. Every run is flagged `THROTTLED`.
2. Pi egresses via `wlan0` (gateway 10.50.40.1). Dish `192.168.100.1:9200` was unreachable,
   so the Pi is not yet on Starlink. Verify with `./run.sh dish`.
3. Then run the full campaign on the Pi: `tmux new -s cc './run.sh run'`.

## Findings / caveats (already in README and code headers)
- `picoquicdemo` **server ignores `-F`** (perf log is client-only) and writes ticket/token
  stores to the working directory, so each instance runs from `state/<label>`.
- Server CPU is not a measure of CC compute cost: picoquic waits in `select()` with a timeout
  set by pacing, so CPU grows with transfer duration (≈4.5 CPU-s per 50 MiB at 85 Mbps,
  versus 0.17 CPU-s at 2.4 Gbps on loopback).
- t3 is burstable: when CPU credits run out the hypervisor withholds CPU, so watch
  `steal_pct` and `server_steal_runs`.
- The curl ngtcp2 back end raises each stream window to 10 MiB, so flow control does not
  cap throughput.

## Possible next steps (user has not asked yet)
- Add a `--cc` filter to run a single CC.
- Paired analysis per (round, size) block.
- Optional `QLOG=1` in `server.conf` for sender cwnd/RTT traces.
