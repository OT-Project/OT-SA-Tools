# OT-SA Tools

Bộ công cụ build (toolchain) cho **OT-SA** (OT Security Appliance) — thiết bị bảo mật mạng công nghiệp (OT/ICS). Dự án được fork từ [OPNsense/tools](https://github.com/opnsense/tools) và tùy biến cho mục đích xây dựng appliance bảo mật chuyên dụng.

Toolchain này tạo ra các **image khởi động** (bootable image) ở nhiều định dạng: ISO (DVD), memstick (USB), ổ đĩa máy ảo (VM), flash card (nano), và image cho thiết bị ARM. Quá trình build được điều phối từ mã nguồn FreeBSD, ports (gói phần mềm bên thứ ba), core (lõi OT-SA) và plugins (phần mở rộng).

## Mục lục

- [Yêu cầu hệ thống](#yêu-cầu-hệ-thống)
- [Bắt đầu nhanh](#bắt-đầu-nhanh)
- [Kiến trúc build](#kiến-trúc-build)
- [Các lệnh build theo giai đoạn](#các-lệnh-build-theo-giai-đoạn)
- [Tạo image](#tạo-image)
- [Định dạng image máy ảo (VM)](#định-dạng-image-máy-ảo-vm)
- [Tuỳ chọn build](#tuỳ-chọn-build)
- [Cấu trúc thư mục cấu hình](#cấu-trúc-thư-mục-cấu-hình)
- [Quản lý repository](#quản-lý-repository)
- [Quản lý gói phần mềm](#quản-lý-gói-phần-mềm)
- [Ký số và xác minh](#ký-số-và-xác-minh)
- [Dọn dẹp (cleanup)](#dọn-dẹp-cleanup)
- [Build chéo cho ARM](#build-chéo-cho-arm)
- [Lệnh tự động hoá (composite)](#lệnh-tự-động-hoá-composite)
- [Thao tác từ xa](#thao-tác-từ-xa)
- [Lệnh tiện ích](#lệnh-tiện-ích)
- [Hệ thống hook thiết bị](#hệ-thống-hook-thiết-bị)
- [Cấu trúc dự án](#cấu-trúc-dự-án)
- [Giấy phép](#giấy-phép)

---

## Yêu cầu hệ thống

| Yêu cầu | Chi tiết |
|----------|----------|
| Hệ điều hành | FreeBSD **14.3-RELEASE** (kiến trúc amd64) |
| Dung lượng ổ đĩa | Tối thiểu **40 GB** trống |
| RAM | Tối thiểu **8 GB** |
| Quyền truy cập | **Root** (toàn bộ quá trình build cần quyền root) |
| Phần mềm cần cài | `git` và `pkg` |

> **Lưu ý:** Toolchain này chỉ chạy trên FreeBSD. Không hỗ trợ build trên Linux hoặc macOS.

---

## Bắt đầu nhanh

### Cài đặt cơ bản

```sh
pkg install git
cd /usr
git clone git@github.com:OT-Project/OT-SA-Tools.git tools
cd tools
make update    # Tải/cập nhật toàn bộ repo: src, ports, core, plugins
make dvd       # Build image DVD ISO hoàn chỉnh
```

Image đầu ra nằm trong thư mục được in bởi lệnh:

```sh
make print-IMAGESDIR
```

### Sử dụng thư mục gốc khác (không phải `/usr`)

Mặc định, toolchain đặt tất cả repo con trong `/usr/`. Nếu muốn dùng thư mục khác, thiết lập biến `ROOTDIR`:

```sh
mkdir -p /tmp/otsa && cd /tmp/otsa
git clone git@github.com:OT-Project/OT-SA-Tools.git tools
cd tools
env ROOTDIR=/tmp/otsa make update
```

Khi đó cấu trúc sẽ là:

```
/tmp/otsa/
├── tools/     # repo này (OT-SA-Tools)
├── src/       # mã nguồn FreeBSD
├── ports/     # ports bên thứ ba
├── core/      # gói lõi OT-SA
└── plugins/   # plugins mở rộng
```

---

## Kiến trúc build

Hệ thống build được chia thành **các giai đoạn độc lập** (stage). Mỗi giai đoạn có thể chạy riêng lẻ và chạy lại nhiều lần — hệ thống sẽ tự tiếp tục từ điểm dừng trước đó, không cần build lại từ đầu.

### Sơ đồ phụ thuộc

```
base ──→ kernel
  │
  ├────→ ports ──→ plugins ──→ core ──→ packages / test
  │      (gói bên       (phần mở    (lõi     (đóng gói /
  │       thứ ba)        rộng)       OT-SA)   kiểm thử)
  │
  └────→ distfiles
           (tệp nguồn tải sẵn)

kernel + core ──→ dvd / nano / serial / vga / vm / arm
                  (tạo các loại image khởi động)
```

**Giải thích sơ đồ:**

1. **`base`** phải build trước tiên — đây là nền tảng userland (các chương trình cơ bản, thư viện, bootloader) của FreeBSD.
2. **`kernel`**, **`ports`**, và **`distfiles`** đều phụ thuộc vào `base`.
3. **`plugins`** phụ thuộc `ports` (vì plugin có thể dùng thư viện từ ports).
4. **`core`** phụ thuộc `plugins` (core đóng gói toàn bộ sản phẩm).
5. Các image (`dvd`, `nano`, `serial`, `vga`, `vm`, `arm`) cần cả `kernel` và `core` đã sẵn sàng.

> **Quan trọng:** Nếu chạy lại `make ports`, hệ thống sẽ **xoá kết quả** của `plugins` và `core` (vì chúng phụ thuộc ports). Sau đó bạn cần build lại plugins và core.

Tất cả các giai đoạn được gọi qua `make`:

```sh
make <giai_đoạn> [TUỲ_CHỌN="giá_trị"]
```

---

## Các lệnh build theo giai đoạn

### Giai đoạn biên dịch

| Lệnh | Chức năng |
|-------|-----------|
| `make base` | Biên dịch userland: các chương trình hệ thống, thư viện, bootloader, và các tệp quản trị |
| `make kernel` | Biên dịch kernel FreeBSD và các module kernel có thể nạp |
| `make ports` | Biên dịch toàn bộ gói phần mềm bên thứ ba (275+ port) từ cây ports |
| `make plugins` | Biên dịch các plugin mở rộng của OT-SA (107 plugin) |
| `make core` | Đóng gói phần lõi (core) của OT-SA thành package |

### Kiểm tra và xác minh

| Lệnh | Chức năng |
|-------|-----------|
| `make packages` | Xác minh tính toàn vẹn và đóng gói tất cả các package |
| `make test` | Chạy bộ kiểm thử hồi quy (regression test) trên các thay đổi core |
| `make audit` | Kiểm tra các package đã build có lỗ hổng bảo mật đã biết hay không (tra cứu vulnerability database) |
| `make lint` | Kiểm tra cú pháp POSIX `sh` cho toàn bộ script build và composite |

> **Ghi chú:** Lệnh `lint` được Makefile thiết lập là **điều kiện tiên quyết** cho mọi target build. Nghĩa là mỗi lần chạy build, tất cả script đều được kiểm tra cú pháp trước khi thực thi.

---

## Tạo image

Sau khi build xong các giai đoạn biên dịch, bạn có thể tạo image khởi động ở nhiều định dạng:

| Lệnh | Đầu ra | Mô tả |
|-------|--------|-------|
| `make dvd` | Tệp `.iso` | Image ISO live — ghi ra đĩa DVD hoặc mount để cài đặt. Hỗ trợ UEFI và Legacy BIOS (amd64) |
| `make serial` | Tệp `.img` | Image USB memstick với giao tiếp qua **serial console** (dùng cho thiết bị không có màn hình) |
| `make vga` | Tệp `.img` | Image USB memstick với giao tiếp qua **VGA console** (màn hình + bàn phím thông thường) |
| `make nano` | Tệp `.img` | Image full-disk cho **thẻ nhớ flash / SSD** — phù hợp thiết bị nhúng, mặc định 3 GB |
| `make vm` | Tệp `.vmdk` | Image ổ đĩa **máy ảo** — mặc định định dạng VMDK (VMware), 20 GB disk, 1 GB swap |
| `make arm` | Tệp `.img` | Image cho **thiết bị ARM** (Raspberry Pi, NanoPi R4S, v.v.) |
| `make release` | Nhiều tệp | Tạo đồng thời tất cả image phát hành: dvd, nano, serial, vga |
| `make distribution` | Nhiều tệp | Giống `release` nhưng thêm kiểm tra phiên bản và tính nhất quán |

---

## Định dạng image máy ảo (VM)

### Cú pháp

```sh
make vm-<định_dạng>[,<dung_lượng>[,<swap>[,<extras>]]]
```

### Các định dạng được hỗ trợ

| Định dạng | Tương thích với | Ghi chú |
|-----------|-----------------|---------|
| `qcow` | QEMU, KVM | Phiên bản cũ (legacy) |
| `qcow2` | QEMU, KVM | Phiên bản hiện tại, hỗ trợ snapshot |
| `raw` | Mọi hypervisor | Image sector thô, không nén |
| `vhd` | VirtualPC, Hyper-V, Xen | Dynamic (dung lượng tăng dần) |
| `vhdf` | Azure, Hyper-V, Xen | Fixed (dung lượng cố định — bắt buộc cho Azure) |
| `vmdk` | VMware, VirtualBox | Dynamic, định dạng mặc định |

### Giá trị mặc định

- Định dạng: `vmdk`
- Dung lượng disk: `20G`
- Swap: `1G` (đặt `off` để tắt swap)

### Ví dụ

```sh
make vm                           # VMDK 20G, swap 1G (mặc định)
make vm-qcow2                     # QCOW2 20G, swap 1G
make vm-raw,40G                   # RAW 40G, swap 1G
make vm-vhdf,30G,2G               # VHD fixed 30G, swap 2G (cho Azure)
make vm-vmdk,20G,off              # VMDK 20G, không có swap
```

### Tuỳ chỉnh kích thước image nano

```sh
make nano-<kích_thước>
```

Ví dụ: `make nano-4G` tạo image nano 4 GB thay vì mặc định 3 GB.

---

## Tuỳ chọn build

Các tuỳ chọn được truyền qua dòng lệnh hoặc thiết lập trong `config/<ABI>/build.conf`:

| Tuỳ chọn | Mặc định | Mô tả |
|----------|----------|-------|
| `SETTINGS` | (tự động) | Tên profile cấu hình — chọn thư mục `config/<tên>/` tương ứng |
| `CONFIGDIR` | (tự động) | Đường dẫn tuyệt đối tới thư mục cấu hình (ghi đè `SETTINGS`) |
| `ABI` | từ SETTINGS | Chuỗi phiên bản ABI (ví dụ: `26.1`) |
| `ADDITIONS` | (trống) | Danh sách gói/plugin bổ sung thêm vào image |
| `ARCH` | native | Kiến trúc đích: `amd64` hoặc `aarch64` |
| `COMSPEED` | `115200` | Tốc độ baud của serial console |
| `DEBUG` | (trống) | Bật build kernel debug với symbol bổ sung |
| `DEVICE` | `A10` | Profile thiết bị từ thư mục `device/` |
| `KERNEL` | `SMP` | Tên cấu hình kernel (tệp trong `config/<ABI>/`) |
| `MIRRORS` | (upstream) | URL mirror để tải trước (prefetch) các bộ cài sẵn |
| `NAME` | `OPNsense` | Tên sản phẩm hiển thị trong image |
| `PRIVKEY` | (trống) | Đường dẫn tới private key cho ký số package |
| `PUBKEY` | (trống) | Đường dẫn tới public key cho xác minh chữ ký |
| `SUFFIX` | (trống) | Hậu tố tên gói chính (top package) |
| `TYPE` | `opnsense` | Tên cơ sở của gói chính |
| `UEFI` | `arm dvd serial vga vm` | Danh sách loại image sử dụng UEFI hybrid boot |
| `VERSION` | (timestamp) | Nhãn phiên bản cho bản build |
| `ZFS` | (trống) | Tên ZFS pool cho image VM (để trống = dùng UFS) |

### Ví dụ sử dụng

```sh
# Build với tên sản phẩm tuỳ chỉnh
make dvd NAME="OT-SA" TYPE="otsa"

# Build kernel debug
make kernel DEBUG=debug

# Build image VM cho Azure với ZFS
make vm-vhdf,30G ZFS=zroot

# Thêm plugin bổ sung vào image
make dvd ADDITIONS="security/acme-client net/tailscale"
```

---

## Cấu trúc thư mục cấu hình

Mỗi phiên bản ABI có một thư mục cấu hình riêng:

```
config/<ABI>/
├── build.conf          # Cấu hình build chính: phiên bản OS, ngôn ngữ (PHP, Python, ...), SSL
├── build.conf.local    # Ghi đè cục bộ — được đọc TRƯỚC build.conf, KHÔNG commit vào git
├── ports.conf          # Danh sách ports cần build (275+ gói)
├── plugins.conf        # Danh sách plugins cần build (107 plugin)
├── make.conf           # Cấu hình make.conf của FreeBSD cho quá trình build ports
├── extras.conf         # Hàm hook tuỳ chỉnh cho từng loại image
├── SMP                 # Cấu hình kernel SMP (cho amd64)
├── SMP-ARM             # Cấu hình kernel SMP (cho ARM)
├── src.conf            # Cấu hình src.conf của FreeBSD (bật/tắt tính năng hệ thống)
├── base.plist.*        # Danh sách tệp trong base package (25.000+ dòng)
└── base.obsolete.*     # Danh sách tệp lỗi thời cần xoá khi nâng cấp
```

Hiện tại dùng bộ cấu hình **26.1** (FreeBSD 14.3).

### Ghi đè cấu hình cục bộ (build.conf.local)

Tệp `build.conf.local` cho phép ghi đè bất kỳ biến nào trong `build.conf` mà không ảnh hưởng tới repo git.

#### So sánh nhanh `build.conf` vs `build.conf.local`

| Đặc điểm | `build.conf` | `build.conf.local` |
|----------|-------------|---------------------|
| **Mục đích** | Cấu hình **chung** của dự án | Cấu hình **riêng** của từng dev/máy |
| **Commit vào git?** | ✅ Có | ❌ Không (đã trong `.gitignore`) |
| **Bắt buộc tồn tại?** | ✅ Có | ❌ Không (tuỳ chọn) |
| **Cú pháp** | `?=` (set if undefined) | `=` (force set) |
| **Thứ tự load** | Sau (priority thấp) | Trước (priority cao) |

#### Cú pháp khác nhau (QUAN TRỌNG!)

```sh
# build.conf — luôn dùng ?= để cho phép override
PHP?=83                         # Set PHP=83 nếu PHP chưa có giá trị

# build.conf.local — luôn dùng = để force override
PHP=84                          # Luôn set PHP=84 (override mọi nơi khác)
```

> ⚠️ **Lưu ý:** Nếu dùng `?=` trong `build.conf.local`, override sẽ **không có hiệu lực** vì biến trong `build.conf` cũng dùng `?=` và sẽ được load sau. Luôn dùng `=` trong file `.local`.

#### Ví dụ thực tế

**Tạo cấu hình cá nhân:**

```sh
# config/26.1/build.conf.local

# Tôi muốn thử nghiệm Python 3.12
PYTHON=312

# Build cho fork riêng của tôi
GITBASE=https://github.com/my-fork

# Bật verbose để debug build
VERBOSE=1

# Đổi tên sản phẩm
NAME=MyProduct
```

**Kết quả khi build:**

| Biến | Giá trị | Nguồn |
|------|---------|-------|
| `APACHE` | 24 | build.conf |
| `PHP` | 83 | build.conf |
| `PYTHON` | **312** | build.conf.local (override) |
| `GITBASE` | **https://github.com/my-fork** | build.conf.local (override) |
| `VERBOSE` | **1** | build.conf.local |

#### Khi nào dùng file nào?

| Tình huống | Dùng |
|-----------|------|
| Đổi phiên bản PHP/Python cho **cả team** | `build.conf` (commit) |
| Cá nhân thử nghiệm phiên bản mới | `build.conf.local` |
| Cấu hình mặc định của dự án | `build.conf` (commit) |
| Build cho fork riêng | `build.conf.local` |
| Signing keys / URL nội bộ | `build.conf.local` (bí mật) |
| Bật DEBUG/VERBOSE | `build.conf.local` |

#### Workflow điển hình

```sh
# 1. Clone repo
git clone git@github.com:OT-Project/OT-SA-Tools.git tools
cd tools

# 2. Xem các biến có thể tuỳ chỉnh (đã có comment chi tiết)
cat config/26.1/build.conf

# 3. Tạo cấu hình cá nhân (KHÔNG commit)
cat > config/26.1/build.conf.local <<EOF
GITBASE=https://github.com/OT-Project
VERBOSE=1
EOF

# 4. Build với cấu hình của bạn
make update
make base kernel
```

#### Tương tự cho plugins

Tệp `plugins.conf.local` cho phép tuỳ chỉnh danh sách plugin cục bộ (cùng cơ chế).

### Cấu hình Repository URL + Branch (repositories.yaml)

Để thay đổi URL **và branch** của git repositories qua một file YAML duy nhất:

#### Setup ban đầu

```bash
cp config/26.1/repositories.yaml.example config/26.1/repositories.yaml
vim config/26.1/repositories.yaml
```

#### Cấu trúc file (nested format: URL + branch)

```yaml
# config/26.1/repositories.yaml

# Base URL chung (dùng khi url: null)
git_base: https://github.com/OT-Project

repositories:
  core:
    url: null                    # null = git_base/core
    branch: null                 # null = stable/${ABI}
  plugins:
    url: null
    branch: null
  ports:
    url: https://github.com/OT-Project/OT-Ports    # Override URL
    branch: develop                                 # Override branch
  src:
    url: null
    branch: null
  tools:
    url: null
    branch: null
  portsref:
    url: https://git.FreeBSD.org/ports.git
    branch: main
```

#### Ý nghĩa giá trị `null`

| Trường | `null` nghĩa là |
|--------|----------------|
| `url: null` | Tự ghép `${git_base}/<repo_name>` |
| `branch: null` | Dùng default Makefile (xem bảng dưới) |

#### Branch mặc định cho mỗi repo

| Repo | Branch mặc định |
|------|-----------------|
| `core` | `stable/${ABI}` (vd: `stable/26.1`) |
| `plugins` | `stable/${ABI}` |
| `ports` | `master` |
| `src` | `stable/${ABI}` |
| `tools` | `master` |
| `portsref` | `main` |

### Cấu hình Mirrors (mirrors.yaml — file riêng)

Mirror servers dùng cho `prefetch` và `clone` (tách riêng khỏi `repositories.yaml` cho rõ ràng):

```bash
cp config/26.1/mirrors.yaml.example config/26.1/mirrors.yaml
```

```yaml
# config/26.1/mirrors.yaml
mirrors:
  - https://mirror.internal.local/otsa
  - https://mirror.backup.local/otsa
```

### Ưu tiên ghi đè (URL & Branch & Mirrors)

#### URL

```
1. Command line:        make -O "https://custom"      ← cao nhất
2. build.conf.local:    GITBASE=https://custom
3. repositories.yaml:   git_base hoặc per-repo url
4. Makefile default:    https://github.com/opnsense   ← thấp nhất
```

#### Branch

```
1. Command line:        make COREBRANCH=master         ← cao nhất
2. build.conf.local:    COREBRANCH=master
3. repositories.yaml:   per-repo branch
4. Makefile default:    stable/${ABI} | master | main  ← thấp nhất
```

#### Mirrors

```
1. Command line:        make -m "https://custom"       ← cao nhất
2. mirrors.yaml:        list mirrors
3. Makefile default:    6 OPNsense mirrors             ← thấp nhất
```

> ⚠️ **Lưu ý quan trọng**: Cả `repositories.yaml` và `mirrors.yaml` đều **không được commit** vào git (đã trong `.gitignore`) — chỉ file `.example` mới được commit. Lý do: 2 file này có thể chứa URL nội bộ riêng.

> 💡 **Logic override branch**: YAML chỉ override branch khi giá trị hiện tại là default Makefile. Nếu user truyền `make COREBRANCH=...` hoặc set trong `build.conf.local`, command line/build.conf vẫn thắng.

### Ví dụ thực tế

#### Build với fork OT-Project + branch riêng

```yaml
# config/26.1/repositories.yaml
git_base: https://github.com/OT-Project

repositories:
  core:
    url: null
    branch: ot-customizations
  plugins:
    url: null
    branch: ot-customizations
  ports:
    url: null
    branch: develop
  # src, tools, portsref: giữ default
```

```bash
make update    # Tự pull đúng URL + branch
```

#### Build branch experiment (override command line)

```bash
# Dù YAML có gì, command line vẫn thắng
make update COREBRANCH=hotfix-123
```

---

## Quản lý repository

Toolchain quản lý nhiều repo cùng lúc. Lệnh `update` sẽ clone (nếu chưa có) hoặc pull (nếu đã có) từ remote.

```sh
make update                          # Cập nhật TẤT CẢ repo (src, ports, core, plugins, tools)
make update-core,plugins             # Chỉ cập nhật core và plugins
make update VERSION=26.1             # Checkout tag phiên bản cụ thể
```

Các repo có thể quản lý: `core`, `plugins`, `ports`, `portsref`, `src`, `tools`

---

## Quản lý gói phần mềm

### Tải trước từ mirror (prefetch)

Thay vì build từ đầu, bạn có thể tải các bộ (set) đã build sẵn từ mirror:

```sh
make prefetch-base,kernel,packages              # Tải base, kernel, packages từ mirror
make prefetch-base VERSION=26.1                 # Tải phiên bản cụ thể
make clone-base,kernel,packages TO=26.1         # Sao chép từ bản build cục bộ có sẵn
```

### Build lại từng gói riêng lẻ

Không cần build lại toàn bộ — bạn có thể chỉ định build lại một gói cụ thể bằng cú pháp `<giai_đoạn>-<tên_gói>`:

```sh
make ports-curl                                 # Build lại port curl
make plugins-os-some-plugin                     # Build lại một plugin cụ thể
make core-opnsense                              # Build lại core package
```

### Tuỳ chọn build ports

Các tuỳ chọn được truyền qua biến `PORTSENV`:

| Tuỳ chọn | Mặc định | Tác dụng |
|----------|----------|----------|
| `BATCH` | `yes` | `no` = dừng lại và mở shell khi build thất bại (để debug) |
| `DEPEND` | `yes` | `no` = không chạm vào gói plugin/core (giữ nguyên chúng) |
| `MISMATCH` | `yes` | `no` = bỏ qua việc build lại các gói có phiên bản không khớp |
| `PRUNE` | `yes` | `no` = bỏ qua kiểm tra tính toàn vẹn ports |

```sh
# Build lại curl mà không ảnh hưởng plugin/core, bỏ qua kiểm tra ports
make ports-curl PORTSENV="DEPEND=no PRUNE=no"
```

### Ghi đè danh sách gói

Mặc định, `make ports` build tất cả gói trong `ports.conf`. Để chỉ build một số gói cụ thể:

```sh
make ports PORTSLIST="security/openssl"         # Chỉ build openssl
make plugins PLUGINSLIST="devel/debug"          # Chỉ build plugin debug
```

---

## Ký số và xác minh

Hệ thống hỗ trợ ký số các bộ build (set) và xác minh chữ ký để đảm bảo tính toàn vẹn:

```sh
make sign-base,kernel,packages       # Ký (hoặc ký lại) các set
make verify                          # Xác minh chữ ký của tất cả set
make fingerprint                     # Hiển thị fingerprint của khoá ký
```

Khoá ký được lưu tại `config/<ABI>/repo.key` (private) và `config/<ABI>/repo.pub` (public). Các tệp này **không được commit** vào git (đã có trong `.gitignore`).

---

## Dọn dẹp (cleanup)

Xoá kết quả build của các giai đoạn cụ thể:

```sh
make clean-<mục_tiêu>[,<mục_tiêu>,...]
```

### Các mục tiêu có thể dọn dẹp

| Mục tiêu | Xoá cái gì |
|-----------|-------------|
| `base` | Kết quả build userland |
| `kernel` | Kết quả build kernel |
| `ports` | Kết quả build ports (cũng xoá plugins và core) |
| `plugins` | Kết quả build plugins |
| `core` | Kết quả build core |
| `packages` | Gói package đã đóng |
| `distfiles` | Tệp nguồn đã tải |
| `images` | Tất cả image đã tạo |
| `dvd`, `nano`, `serial`, `vga`, `vm`, `arm` | Loại image cụ thể |
| `obj` | Thư mục object (kết quả biên dịch trung gian) |
| `logs` | Log build |
| `sets` | Các bộ (set) đã ký |
| `stage` | Thư mục staging |
| `src` | Mã nguồn FreeBSD đã clone |
| `release` | Tất cả image release |
| `xtools` | Cross-compilation toolchain |

### Ví dụ

```sh
make clean-base,kernel              # Dọn base và kernel để build lại
make clean-images                   # Xoá tất cả image, giữ lại packages
make clean-obj                      # Xoá object files để giải phóng dung lượng
```

---

## Build chéo cho ARM

Toolchain hỗ trợ build chéo (cross-compilation) cho các thiết bị ARM từ máy amd64:

```sh
make base kernel DEVICE=RPI                # Build base + kernel cho Raspberry Pi
make xtools DEVICE=RPI                     # Build toolchain native cho ARM
make packages DEVICE=RPI                   # Build packages (sử dụng xtools)
make arm-<kích_thước> DEVICE=RPI           # Tạo image ARM cuối cùng
```

### Thiết bị ARM được hỗ trợ

| Thiết bị | Tệp cấu hình | Mô tả |
|----------|---------------|-------|
| **A10** | `device/A10.conf` | Deciso NetBoard A10 — thiết bị mặc định |
| **ARM64** | `device/ARM64.conf` | ARM64 chung — tương thích QEMU và ESXi trên ARM |
| **R4S** | `device/R4S.conf` | FriendlyARM NanoPi R4S — router nhỏ gọn, serial 1.5 Mbps |
| **ROCKPRO64** | `device/ROCKPRO64.conf` | Pine64 RockPro64 — SBC hiệu suất cao |
| **RPI** | `device/RPI.conf` | Raspberry Pi 3/4/CM4 — SBC phổ biến nhất |

---

## Lệnh tự động hoá (composite)

Các lệnh composite kết hợp nhiều giai đoạn build thành một quy trình hoàn chỉnh:

### Build hàng đêm (nightly)

| Lệnh | Mô tả |
|-------|-------|
| `make nightly` | Build tự động hoàn chỉnh — dọn dẹp, cập nhật, build, kiểm thử, ghi log |
| `make nightly EXTRABRANCH=master` | Nightly build kèm cả gói từ nhánh dev |

Quy trình nightly gồm 2 giai đoạn:

- **Giai đoạn 1:** Dọn obj → cập nhật repo → lấy thông tin → build base → build kernel → build xtools → tải distfiles → dọn packages
- **Giai đoạn 2:** Xoá tệp lỗi thời → cấu hình ports → build ports → build plugins → build core → audit bảo mật → kiểm thử

### Theo dõi build

| Lệnh | Mô tả |
|-------|-------|
| `make watch` | Hiển thị trạng thái build nightly mới nhất |
| `make watch-<giai_đoạn>` | Xem log realtime của một giai đoạn build cụ thể |

Log nightly được lưu trữ trong 1 tuần. Thư mục `./latest` trỏ tới lần build gần nhất. Xem đường dẫn log:

```sh
make print-LOGSDIR
```

### Các quy trình khác

| Lệnh | Mô tả |
|-------|-------|
| `make distribution` | Build phát hành chính thức — kiểm tra phiên bản, đảm bảo tính nhất quán |
| `make hotfix` | Build lại các plugin/core bị thiếu hoặc hỏng, ký lại |
| `make hotfix-core` | Build lại toàn bộ core |
| `make hotfix-plugins` | Build lại toàn bộ plugins |
| `make hotfix-ports` | Build lại các port bị thiếu hoặc sai phiên bản |
| `make factory` | Tạo image cho thiết bị nhúng (embedded) |
| `make custom-<image> ADDITIONS="..."` | Tạo image tuỳ chỉnh với plugin bổ sung |

### Ví dụ custom image

```sh
# Tạo image DVD với thêm plugin HAProxy và Zabbix agent
make custom-dvd ADDITIONS="net/haproxy net-mgmt/zabbix7-agent"
```

---

## Thao tác từ xa

Đẩy (upload) hoặc kéo (download) các bộ build qua SSH:

```sh
make upload-<set>[,...]  SERVER=user@host     # Đẩy set/image lên server
make download-<set>[,...] SERVER=user@host     # Kéo set/image từ server (thay thế bản cục bộ)
```

- Biến `REMOTEDIR` thiết lập đường dẫn trên server từ xa.
- Lệnh `download` hoạt động giống `prefetch` — nó **xoá bản cục bộ trước** rồi mới tải về.

---

## Lệnh tiện ích

| Lệnh | Mô tả |
|-------|-------|
| `make info` | In trạng thái các repository (nhánh, commit, thay đổi) |
| `make print-<BIẾN>[,<BIẾN>]` | Kiểm tra giá trị biến build |
| `make print-SETSDIR` | Đường dẫn tới thư mục set (kernel, base, packages) |
| `make print-IMAGESDIR` | Đường dẫn tới thư mục chứa image |
| `make print-LOGSDIR` | Đường dẫn tới thư mục log |
| `make rename-<set> VERSION=X` | Đổi nhãn phiên bản của một set |
| `make compress-<image>[,...]` | Nén image bằng bzip2 để phân phối |
| `make make.conf` | Tạo tệp make.conf để build port độc lập (ngoài toolchain) |
| `make chroot` | Vào build jail (môi trường build cách ly) |
| `make chroot-<thư_mục_con>` | Vào thư mục con cụ thể trong chroot |
| `make boot-<image>` | Khởi động image trong bhyve (chỉ serial/nano) |
| `make confirm` | Cổng xác nhận yes/no — hữu ích khi viết script tự động |
| `make skim` | Xem xét và áp dụng thay đổi từ upstream port tree |
| `make sync-cat/port[,...]` | Cherry-pick thay đổi port giữa các nhánh |
| `make rebase` | Tạo lại danh sách tệp base sau khi thay đổi src |
| `make distfiles` | Tải trước tệp nguồn port (cache lại để build nhanh hơn) |

---

## Hệ thống hook thiết bị

Hook là các hàm shell được gọi **tự động** trong quá trình tạo image, cho phép tuỳ chỉnh image cho từng loại thiết bị hoặc định dạng cụ thể.

### Thứ tự thực thi

1. **Hook cấu hình** (`config/<ABI>/extras.conf`) — chạy trước, áp dụng cho mọi thiết bị
2. **Hook thiết bị** (`device/<TÊN>.conf`) — chạy sau, tuỳ chỉnh riêng cho thiết bị

### Các hook có thể định nghĩa

| Hook | Được gọi khi |
|------|-------------|
| `serial_hook()` | Tạo image serial memstick |
| `dvd_hook()` | Tạo image DVD ISO |
| `nano_hook()` | Tạo image nano flash |
| `vga_hook()` | Tạo image VGA memstick |
| `vm_hook()` | Tạo image VM |
| `arm_hook()` | Tạo image ARM |

### Cách viết hook

Tham số `${1}` là đường dẫn **gốc hệ thống tệp** (filesystem root) của image đang được tạo. Bạn có thể thêm, sửa, hoặc xoá tệp trong đó:

```sh
# Ví dụ trong device/CUSTOM.conf
serial_hook() {
    # Thêm tệp cấu hình tuỳ chỉnh vào image serial
    cp /path/to/custom.conf ${1}/etc/custom.conf
    
    # Bật service tuỳ chỉnh
    echo 'custom_service_enable="YES"' >> ${1}/etc/rc.conf.local
}

vm_hook() {
    # Bật tự động mở rộng filesystem cho VM
    echo 'growfs_enable="YES"' >> ${1}/etc/rc.conf.local
}
```

---

## Cấu trúc dự án

```
.
├── Makefile              # Điểm vào chính — điều phối tới build/ và composite/
├── CLAUDE.md             # Hướng dẫn cho Claude Code AI assistant
├── README.md             # Tệp bạn đang đọc
├── LICENSE               # Giấy phép BSD 2-Clause
├── .gitignore            # Loại trừ tệp cục bộ và khoá ký
│
├── build/                # Script build từng giai đoạn (43 script)
│   ├── common.sh         # ★ Tệp quan trọng nhất — hàm dùng chung, parse tuỳ chọn,
│   │                     #   git helpers, setup chroot/base/kernel/packages
│   ├── base.sh           # Build userland FreeBSD
│   ├── kernel.sh         # Build kernel
│   ├── ports.sh          # Build ports
│   ├── plugins.sh        # Build plugins
│   ├── core.sh           # Đóng gói core
│   ├── dvd.sh            # Tạo image ISO
│   ├── nano.sh           # Tạo image flash
│   ├── serial.sh         # Tạo image serial memstick
│   ├── vga.sh            # Tạo image VGA memstick
│   ├── vm.sh             # Tạo image VM
│   ├── arm.sh            # Tạo image ARM
│   ├── audit.sh          # Kiểm tra lỗ hổng bảo mật
│   ├── sign.sh           # Ký số package
│   ├── verify.sh         # Xác minh chữ ký
│   ├── test.sh           # Kiểm thử
│   └── ...               # 27 script phụ trợ khác
│
├── composite/            # Script tự động hoá đa giai đoạn
│   ├── nightly.sh        # Build hàng đêm tự động
│   ├── distribution.sh   # Build phát hành chính thức
│   ├── hotfix.sh         # Build sửa lỗi nhanh
│   ├── factory.sh        # Build image thiết bị nhúng
│   ├── custom.sh         # Build image tuỳ chỉnh
│   ├── util.sh           # Hàm helper: load_core_version(), load_make_vars()
│   ├── watch.sh          # Theo dõi trạng thái build
│   └── pkgver.sh         # Kiểm tra phiên bản package
│
├── config/               # Cấu hình build theo phiên bản ABI
│   └── 26.1/             # Cấu hình cho FreeBSD 14.3, phiên bản 26.1
│
├── device/               # Cấu hình và hook theo thiết bị
│   ├── A10.conf          # Deciso NetBoard A10
│   ├── ARM64.conf        # ARM64 chung (QEMU/ESXi)
│   ├── R4S.conf          # NanoPi R4S
│   ├── ROCKPRO64.conf    # RockPro64
│   └── RPI.conf          # Raspberry Pi 3/4/CM4
│
└── scripts/              # Script tiện ích
    ├── pkg_sign.sh       # Wrapper ký package
    ├── pkg_fingerprint.sh # Hiển thị fingerprint khoá ký
    └── parse_ports_log.py # Phân tích log build ports (Python)
```

---

## Giấy phép

BSD 2-Clause. Xem [LICENSE](LICENSE).
