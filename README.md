# starlink-cc — So sánh CUBIC / BBRv3 / Reno của picoquic qua Starlink

Raspberry Pi 4 (sau Starlink) tải file qua HTTP/3 từ server picoquic trên AWS. Mỗi thuật toán
congestion control (CC) chạy trong một process picoquic riêng, nghe trên một cổng UDP riêng.
Với chiều tải xuống, server là bên gửi dữ liệu nên CC của server chính là biến cần so sánh.

```
 Raspberry Pi 4 (trungnt@100.72.249.11)               AWS 47.129.174.173  (/data/starlink-cc)
 ┌──────────────────────────────┐   Starlink   ┌────────────────────────────────────────┐
 │ curl --http3-only (ngtcp2)   │ ───────────► │ picoquic@cubic  udp/4451  -G cubic      │
 │ bench: runs / cpu / dish     │              │ picoquic@bbrv3  udp/4452  -G bbr (v3)   │
 │ gRPC 192.168.100.1:9200 1 Hz │              │ picoquic@reno   udp/4453  -G newreno    │
 └──────────────────────────────┘              │ picoquic-cpu    logs/cpu.csv (cgroup)   │
                                               │ www/ 2MB.bin 50MB.bin manifest.json     │
                                               └────────────────────────────────────────┘
```

## Cấu trúc

| Thư mục | Chạy ở đâu | Nội dung |
|---|---|---|
| `server/` | server, đồng bộ tới `/data/starlink-cc/server` | `server.conf` (map CC → cổng, payload, commit picoquic), `setup.sh` (idempotent), `bin/serve.sh`, `bin/cpu_monitor.py`, unit systemd |
| `scripts/` | Pi, đồng bộ tới `~/starlink-cc` | `client.env`, `setup.sh`, `run.sh`, package `bench/` |
| `Makefile` | máy trạm | deploy, setup, lấy kết quả về |

Trạng thái sinh ra (`build/`, `www/`, `tls/`, `state/`, `logs/` trên server; `.venv/`, `results/` trên Pi)
nằm ngoài vùng đồng bộ nên deploy lại không ghi đè dữ liệu.

Client không lưu bản sao của bảng CC → cổng. Nó đọc `manifest.json` do server sinh ra từ
`server.conf` (cổng, thuật toán, kích thước và SHA-256 payload, commit picoquic). Nhờ vậy một
lượt tải không thể bị gán nhầm thuật toán, và thông tin phía server được lưu kèm mỗi bộ dữ liệu.

## Metric

| Metric | Nguồn | File |
|---|---|---|
| Throughput, thời gian tải | `curl --write-out '%{json}'`: `speed_download` (→ `throughput_mbps`), `time_total` (→ `download_time_s`), kèm `time_connect`, `time_appconnect`, `time_starttransfer` (TTFB), `size_download`, `http_version` | `runs.csv` |
| CPU client | CPU time chính xác của process curl lấy từ `wait4()` rusage (`cpu_pct`, 100 = 1 core). Chuỗi 1 Hz: CPU toàn máy, CPU của curl, nhiệt độ SoC. Cờ `vcgencmd get_throttled` trước và sau mỗi lượt | `runs.csv`, `cpu.csv` |
| CPU server | `cpu.stat:usage_usec` của cgroup từng instance, 1 Hz (100 = 1 vCPU), kèm `steal_pct` của host | server `logs/cpu.csv` |
| Trạng thái dish | gRPC `SpaceX.API.Device.Device/Handle` `get_status`, 1 Hz, resolve schema bằng server reflection. Có latency/drop tới PoP, throughput, obstruction, SNR, góc anten, alert | `dish.csv` (các trường chọn lọc), `dish.jsonl` (bản đầy đủ) |

Một lượt tải được tính là hợp lệ khi curl exit 0, HTTP 200, giao thức HTTP/3 và số byte đúng
như trong manifest. Lượt không hợp lệ vẫn được giữ lại (`valid=0`) vì tỉ lệ hoàn thành cũng là
một kết quả.

## Thiết kế thí nghiệm

- **Khối theo kích cỡ file**: mỗi round đi qua từng kích cỡ file (thứ tự kích cỡ được xáo). Trong
  mỗi kích cỡ, cả 3 CC chạy liền nhau theo thứ tự xáo. Seed được ghi lại. Vì CC được so sánh trong
  cùng một kích cỡ, 3 lượt của một khối (round, kích cỡ) nằm trong một khoảng thời gian ngắn và
  gặp điều kiện Starlink gần như nhau. Điều kiện này thay đổi theo chu kỳ cấp phát vệ tinh 15 s,
  tải beam, thời tiết và vật cản. Mỗi khối cho đúng một mẫu cho mỗi CC, dùng được cho so sánh
  ghép cặp.
- **Khoảng nghỉ** `GAP_SECONDS` trước mỗi lượt để hàng đợi do lượt trước tạo ra kịp xả hết.
- **Kết nối mới cho mỗi lượt** (1-RTT, không 0-RTT, không fallback TCP), nên CC luôn bắt đầu từ
  initial window.
- Payload 2 MiB chủ yếu đo handshake và slow start/startup. Payload 50 MiB chủ yếu đo giai đoạn
  ổn định và khôi phục mất gói.

## Chạy

```bash
make setup-server        # cài libssl-dev, build picoquic, tạo cert + payload + manifest, bật 4 service
make setup-client        # tạo venv (grpcio, grpcio-reflection, protobuf), kiểm tra curl có HTTP3 + ngtcp2

ssh trungnt@100.72.249.11
cd ~/starlink-cc
./run.sh dish                       # kiểm tra đọc được trạng thái dish
ROUNDS=1 ./run.sh run               # smoke test: 1 block = 6 lượt tải
tmux new -s cc './run.sh run'       # chạy đầy đủ (30 round ≈ 180 lượt); không bị dừng khi rớt SSH

make fetch                          # trên máy trạm: kéo results/ của Pi và logs/ của server về
scripts/.venv/bin/python -m bench summarize results/client/<id> --server-cpu results/server/cpu.csv
```

Vận hành server: `systemctl status 'picoquic@*' picoquic-cpu`, `journalctl -u picoquic@bbrv3 -f`.
Để ghi qlog phía bên gửi, đặt `QLOG=1` trong `server.conf`, rồi chạy `make deploy-server` và
`ssh ... /data/starlink-cc/server/setup.sh manifest services`.

## Kết quả (`results/<UTC>/`)

`meta.json` (config, manifest, version curl/OS/kernel/gói Python, route ra ngoài, NTP, seed,
cipher đã thương lượng) · `runs.csv` · `cpu.csv` · `dish.csv` · `dish.jsonl` · `summary.csv`
(median/P25/P75 throughput, median/P95 thời gian tải, TTFB, CPU client/server, latency PoP,
drop rate, % lượt gặp vật cản, số lượt bị throttle).
