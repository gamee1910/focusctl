# 🛡️ Focus Guard (`focus`)

> **Personal Internet Firewall & Anti-Distraction Daemon for Linux**  
> Chặn mạng ở tầng hệ thống, kiểm soát thời gian tập trung và ngăn chặn triệt để thói quen lướt web mất tập trung khi làm việc.

---

## 📌 Mục lục
1. [Tổng quan dự án](#-tổng-quan-dự-án)
2. [Kiến trúc hệ thống](#-kiến-trúc-hệ-thống)
3. [Tài liệu & Thư viện kỹ thuật cốt lõi](#-tài-liệu--thư-viện-kỹ-thuật-cốt-lõi)
4. [Cơ chế Chặn mạng & Chống Bypass](#-cơ-chế-chặn-mạng--chống-bypass)
5. [Nguồn Blocklists & Quản lý danh mục](#-nguồn-blocklists--quản-lý-danh-mục)
6. [Cấu trúc mã nguồn đề xuất](#-cấu-trúc-mã-nguồn-đề-xuất)
7. [Lộ trình phát triển (Roadmap)](#-lộ-trình-phát-triển-roadmap)
8. [Hướng dẫn cài đặt & Phát triển](#-hướng-dẫn-cài-đặt--phát-triển)

---

## 🎯 Tổng quan dự án

### Vấn đề
Các giải pháp dạng **Browser Extension** hay ứng dụng cảnh báo đơn thuần rất dễ bị vô hiệu hóa khi người dùng mất tập trung (chỉ mất 2 click để tắt extension hoặc mở trình duyệt khác/ẩn danh).

### Giải pháp của Focus Guard
* **System/Network-Level Blocking**: Can thiệp trực tiếp vào tầng phân giải tên miền (DNS) và tường lửa hạt nhân Linux (`nftables`).
* **Cam kết cao (Anti-Bypass / Commitment Device)**: Trong Focus Session, không có cơ chế `stop` dễ dàng; ngăn chặn các phương thức lách DNS (DoH/DoT).
* **Quản lý linh hoạt**: Hỗ trợ hẹn giờ phiên (`focus start 90m`), đặt lịch theo khung giờ (`focus schedule`), và lọc theo Category (Social, Adult/Porn, Gaming, Gambling, Custom).

---

## 🏗️ Kiến trúc hệ thống

```text
                     ┌──────────────────┐
                     │   focusctl CLI   │ (Chạy quyền User thường)
                     └─────────┬────────┘
                               │ Unix Domain Socket (/var/run/focus-guard.sock)
                               ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                   focus-guardd (Daemon)                     │ (Chạy nền / CAP_NET_ADMIN)
  │                                                             │
  │  ┌──────────────┐   ┌──────────────┐   ┌─────────────────┐  │
  │  │  Scheduler   │   │ Policy/Lists │   │ SQLite Storage  │  │
  │  │ (Timer/Cron) │   │ (Categories) │   │ (Logs/Sessions) │  │
  │  └──────┬───────┘   └──────┬───────┘   └─────────────────┘  │
  │         └───────────┬──────┘                                │
  │                     ▼                                       │
  │         ┌───────────────────────┐                           │
  │         │ Network Enforcement   │                           │
  │         │ - Local DNS Proxy     │                           │
  │         │ - nftables Controller │                           │
  │         └───────────┬───────────┘                           │
  └─────────────────────┼───────────────────────────────────────┘
                        ▼
             Hạt nhân Linux (Netfilter / DNS / IP Stack)
```

---

## 📚 Tài liệu & Thư viện kỹ thuật cốt lõi

### 1. Ngôn ngữ & Nền tảng
* **Go (Golang)**: v1.22+
* **Hệ điều hành**: Linux (hỗ trợ Kernel v5.x+ có `nftables` và `systemd`).

### 2. Thư viện Go được lựa chọn

| Thành phần | Thư viện khuyến nghị | Mục đích & Lý do lựa chọn |
| :--- | :--- | :--- |
| **DNS Server / Sinkhole** | [`github.com/miekg/dns`](https://github.com/miekg/dns) | Thư viện chuẩn de-facto trong Go để xây dựng DNS server/proxy hiệu năng cao, hỗ trợ UDP/TCP, tùy biến câu trả lời DNS (`0.0.0.0` hoặc `NXDOMAIN`). |
| **nftables Controller** | [`github.com/google/nftables`](https://github.com/google/nftables) | Tương tác trực tiếp với Netlink API của Linux kernel trong pure Go, không cần CGO (`libnftnl`). Dùng để thiết lập redirect port 53 và block DoH/DoT. |
| **Cơ sở dữ liệu** | [`modernc.org/sqlite`](https://gitlab.com/cznic/sqlite) | Trình điều khiển SQLite thuần Go 100% (Pure Go), không phụ thuộc CGO, giúp quá trình cross-compile và build nhị phân tĩnh cực kỳ dễ dàng. |
| **CLI Framework** | [`github.com/spf13/cobra`](https://github.com/spf13/cobra) | Framework tiêu chuẩn cho CLI (`focusctl`), hỗ trợ subcommands, autocompletion và flag parsing. |
| **IPC (Giao tiếp nội bộ)** | Chuẩn `net` (Unix Domain Socket) | Giao tiếp giữa CLI người dùng và Daemon qua socket `/var/run/focus-guard.sock` với định dạng tin nhắn JSON-RPC hoặc REST-like JSON. |
| **Systemd Service** | [`github.com/coreos/go-systemd/v22`](https://github.com/coreos/go-systemd) | Tích hợp thông báo trạng thái `sd_notify` cho systemd watchdog. |

---

## 🔒 Cơ chế Chặn mạng & Chống Bypass

### 1. DNS Sinkhole & Forwarder
* Daemon khởi chạy một DNS Proxy lắng nghe trên `127.0.0.1:5353` (hoặc `127.0.0.53:53`).
* **Cơ chế kiểm tra**:
  * Khi nhận request phân giải domain: Tra cứu trong Trie Cache (cây hậu tố tên miền).
  * Nếu domain nằm trong danh sách cấm $\rightarrow$ Trả về `0.0.0.0` (A record) hoặc `::` (AAAA record) ngay lập tức.
  * Nếu domain an toàn $\rightarrow$ Forward truy vấn đến upstream resolver (ví dụ: Cloudflare `1.1.1.1` hoặc System Resolver) và trả kết quả về cho client.

### 2. Chống Bypass DNS (Transparent Redirect bằng nftables)
Để ngăn chặn các phần mềm tự đổi DNS sang `8.8.8.8` hoặc `1.1.1.1`:
* **NAT Redirect**:
  ```bash
  # Điều hướng toàn bộ DNS traffic (port 53 UDP/TCP) về DNS Proxy của Focus Guard
  nft add table inet focus_guard
  nft add chain inet focus_guard prerouting '{ type nat hook prerouting priority dstnat; }'
  nft add rule inet focus_guard prerouting udp dport 53 redirect to :5353
  nft add rule inet focus_guard prerouting tcp dport 53 redirect to :5353
  ```

### 3. Vô hiệu hóa DoH (DNS over HTTPS) & DoT (DNS over TLS)
Trình duyệt hiện đại (Chrome/Firefox) thường tự động bật DoH để mã hóa truy vấn DNS:
1. **Canary Domain (Firefox/Chrome standard)**:
   * Chặn truy vấn DNS cho tên miền `use-application-dns.net` bằng cách trả về `NXDOMAIN`. Khi nhận kết quả này, Firefox sẽ tự động tắt chế độ DoH và fallback về DNS hệ thống.
2. **Block Port DoT**: Chặn cổng `853` (TCP/UDP) thông qua nftables.
3. **Block DoH Resolver IPs**: Drop kết nối trực tiếp đến danh sách IP của các nhà cung cấp DoH công cộng phổ biến (Cloudflare, Google, Quad9, NextDNS...) trong thời gian phiên Focus đang hoạt động.

### 4. Cam kết phiên (Anti-Cheat Mechanism)
* Khóa chức năng dừng ngang (`focus stop` trả về lỗi từ chối).
* Cung cấp tùy chọn cấu hình:
  * **Strict Mode**: Tuyệt đối không thể hủy trước hạn cho đến khi timer hoàn tất.
  * **Penalty Mode**: Yêu cầu gõ chuỗi ký tự ngẫu nhiên 100-200 ký tự (Terminal Captcha) nếu người dùng thực sự muốn hủy khẩn cấp.
* Bảo vệ Daemon qua `systemd` (`Restart=always`) và phục hồi quy tắc mạng an toàn nếu hệ thống tắt máy (`graceful cleanup handler`).

---

## 📦 Nguồn Blocklists & Quản lý danh mục

Hỗ trợ nạp blocklist từ các nguồn uy tín chuẩn cộng đồng:

* **StevenBlack Hosts**:
  * Unified base + extensions: `porn`, `gambling`, `social`, `fakenews`.
  * Repo: [github.com/StevenBlack/hosts](https://github.com/StevenBlack/hosts)
* **OISD Lists**:
  * Danh sách phân loại domain nhẹ, chất lượng cao.
  * Trang chủ: [oisd.nl](https://oisd.nl)
* **Cấu trúc lưu trữ & Tìm kiếm nhanh**:
  * Sử dụng cấu trúc **Domain Suffix Trie (Cây tiền tố đảo ngược)** trên RAM: Tra cứu $O(L)$ với $L$ là số phân đoạn tên miền (ví dụ: `m.facebook.com` $\rightarrow$ `com` $\rightarrow$ `facebook` $\rightarrow$ `m`).
  * Danh sách domain tùy chỉnh và cấu hình category được lưu trữ lâu dài trong SQLite.

---

## 📁 Cấu trúc mã nguồn đề xuất

```text
.
├── cmd/
│   ├── focusctl/               # CLI Client
│   │   └── main.go
│   └── focus-guardd/           # Background Daemon
│       └── main.go
├── internal/
│   ├── config/                 # Đọc/Ghi file cấu hình (YAML)
│   ├── daemon/                 # Vòng đời daemon, Signal handling, Systemd integration
│   ├── dns/                    # DNS Server, Sinkhole, Upstream forwarder
│   ├── firewall/               # nftables controller (rules, DoH/DoT block)
│   ├── ipc/                    # Unix Domain Socket Server & Client (JSON-RPC)
│   ├── policy/                 # Quản lý category, parse hosts/blocklist, Trie matcher
│   ├── scheduler/              # Timer engine, Cron/Time-window scheduler
│   └── storage/                # SQLite repository (sessions, metrics, logs)
├── configs/
│   └── focus-guard.yaml        # File cấu hình mẫu
├── scripts/
│   ├── focus-guard.service     # Systemd service unit file
│   └── install.sh              # Script cài đặt nhanh
├── CONTEXT.md                  # Yêu cầu và bài toán cốt lõi
├── README.md                   # Tài liệu kỹ thuật tổng quan
└── go.mod
```

---

## 🗺️ Lộ trình phát triển (Roadmap)

### 🔹 Phase 1: MVP (Minimal Viable Product)
- [ ] Khởi tạo Go module và cấu trúc project `cmd/` & `internal/`.
- [ ] Cài đặt Local DNS Proxy (`miekg/dns`) trả về `0.0.0.0` cho domain trong blacklist đơn giản.
- [ ] Xây dựng Unix Socket Server (`focus-guardd`) và Client (`focusctl`).
- [ ] Hiện thực lệnh: `focus start <duration>` và tự động unlock sau khi hết giờ.

### 🔹 Phase 2: Category & Blocklist Engine
- [ ] Parser hỗ trợ format file `hosts` và domain list.
- [ ] Cài đặt Suffix Trie matcher để tra cứu hàng trăm nghìn domain siêu nhanh.
- [ ] Tích hợp SQLite (`modernc.org/sqlite`) để lưu sessions và custom domain rules.
- [ ] Hỗ trợ các category có sẵn: `social`, `porn`, `gambling`, `gaming`.

### 🔹 Phase 3: nftables & Chống Bypass
- [ ] Tích hợp `google/nftables` để redirect port 53 và chặn DoH/DoT.
- [ ] Triển khai DoH Canary Domain (`use-application-dns.net`).
- [ ] Hiện thực Scheduler theo khung giờ trong tuần (`08:00 - 12:00 BLOCK`, ...).
- [ ] Thêm các lệnh CLI: `focus status`, `focus schedule`, `focus history`.

### 🔹 Phase 4: Hardening & Analytics
- [ ] Strict Mode & Terminal Captcha khi muốn hủy phiên.
- [ ] Thu thập thống kê số lượt truy vấn bị chặn (Blocked counter per domain/category).
- [ ] Đóng gói `systemd` service và hoàn thiện script cài đặt.

---

## 🚀 Hướng dẫn cài đặt & Phát triển

### Yêu cầu môi trường
* Go 1.22 trở lên.
* Linux Kernel có hỗ trợ `nftables`.

### Thao tác cơ bản (Dự kiến)
```bash
# 1. Build các binary
go build -o bin/focusctl ./cmd/focusctl
go build -o bin/focus-guardd ./cmd/focus-guardd

# 2. Khởi chạy Daemon với quyền quản trị mạng
sudo ./bin/focus-guardd

# 3. Sử dụng CLI
./bin/focusctl start 45m
./bin/focusctl status
```
