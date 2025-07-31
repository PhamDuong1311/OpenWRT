# OpenWRT

## 1. Build system cơ bản
**Build system** là tập hợp các Makefile và các Patch được nhà sản xuất thiết kế để tự động hóa toàn bộ quá trình build 1 firmware hoàn chỉnh cho router. Hệ thống thực hiện các bước:
- Tạo 1 **cross-compiler toolchain**
- Dùng toolchain đó để compile **Linux kernel**, **root filesystem**...

### 1.1 Cross-compiler toolchain
Thông thường, khi lập trình trên máy tính, việc build và run đều chạy trên máy tính đó. Tuy nhiên với router hoặc các thiết bị nhúng khác sử dụng kiến trúc CPU hoàn toàn khác.

=> Sử dụng **cross-compiler** để build firmware trên máy tính và run trên router. Để làm được thì cần 1 bộ công cụ đặc biệt gọi là **cross-compiler toolchain**, bao gồm:
- Compiler (GCC)
- Các tiện ích binary (Assembler, Linker)
- Thư viện C chuẩn (glibc)

### 1.2 Cấu trúc thư mục (lúc chưa build)
Source code để build từ repo chính thức của OpenWRT gồm 7 folder, trong đó có 4 folder phục vụ cho **build system**:
- **/config**: chứa các file cấu hình cho **menuconfig**
- **/include**: chứa các tệp dạng **.mk** định nghĩa các hàm và quy chắc chung. Các Makefile của toàn bộ hệ thống sử dụng các **.mk** này để sử dụng lại các logic chung. (giống các header file trong source C file)
- **/package - BS**: chứa các **recipe nấu ăn** để nấu các gói phần mềm như **luci, htop, openssl, wget...** Mỗi  thư mục con tương ứng với 1 gói phần mềm. Hầu hết mọi thành phần trong firmware OpenWRT đều đóng gói dạng **.ipk**
- **/script**: chứa các script dạng **.sh, .py, .per** để cho phép người dùng tải hoặc cập nhật các package
- **/target - BS**: đây là thư mục cực kỳ quan trọng, chứa mọi thứ liên quan đến phần cứng cụ thể (router) mà bạn muốn build firmware:
    - **/linux**: chứa các cấu hình kernel và các bản vá **.patch** cần thiết để kernel Linux có thể hoạt động trên phần cứng đó.
    - **/image**: chứa các Makefile mô tả quy trình đống gói các thành phần đã biên dịch (kernel, rootfs) thành 1 tệp firmware hoàn chỉnh với định dạng đúng cho thiết bị.
- **/toolchain - BS**: chứa các Makefile để build bộ **cross-compiler toolchain** bao gồm Compiler, C library, Binary utilities.
- **/tools - BS**: chứa các Makefile để build các công cụ cần thiết trên máy build như các tiện ích để xây dựng **toolchain** và các **package**, các công cụ để tạo **image firmware**. Bằng cách tự build các công cụ này, **OpenWRT** không phụ thuộc vào các ver sẵn có trên các **Linux distro** khác nhau => tránh lỗi không tương thích.

## 2. OpenWrt Feeds
Trong OpenWrt, **feeds** là kho lưu trữ các công thức build (**build recipes**) cho các package được định nghĩa trước cho **OpenWrt Buildroot**, cung cấp các package bổ sung cho **Build system** của OpenWrt.
### 2.1 Cấu hình bằng feed
OpenWrt sử dụng file cấu hình cho feeds là "**feeds.conf**" và "**feeds.conf.default**". Mỗi feed được định nghĩa trên 1 dòng riêng với 3 thành phần cách nhau bởi khoảng trắng:
```
<phương_thức_feed> <tên_feed> <nguồn_feed>
```

Dưới đây là file "**feeds.conf.default**" trong github repo chính thức của OpenWrt:

```bash
src-git packages https://git.openwrt.org/feed/packages.git
src-git luci https://git.openwrt.org/project/luci.git
src-git routing https://git.openwrt.org/feed/routing.git
src-git telephony https://git.openwrt.org/feed/telephony.git
src-git video https://github.com/openwrt/video.git
#src-git targets https://github.com/openwrt/targets.git
#src-git oldpackages http://git.openwrt.org/packages.git
#src-link custom /usr/src/openwrt/custom-feed
```

#### OpenWrt hỗ trợ nhiều phương thức tải feed:
| Phương thức | Chức năng |
| :-- | :-- |
| `src-bzr` | Tải dữ liệu bằng bzr |
| `src-cpy` | Copy dữ liệu từ đường dẫn (tương đối hoặc tuyệt đối) |
| `src-darcs` | Tải dữ liệu bằng darcs |
| `src-git` | Tải dữ liệu bằng git (shallow clone, depth=1) |
| `src-git-full` | Tải dữ liệu bằng git (full clone) |
| `src-gitsvn` | Hoạt động hai chiều giữa SVN và git |
| `src-hg` | Tải dữ liệu bằng mercurial (hg) |
| `src-link` | Tạo symlink đến đường dẫn tuyệt đối |
| `src-svn` | Tải dữ liệu bằng subversion |

### 2.2 Cách feed làm việc
Feeds được truy xuất và quản lý bởi `scripts/feeds`, nơi tổng hợp các package từ nhiều nguồn trong **OpenWrt Build System**. Có 2 bước để dev xử lý trước khi build image:
#### Step 1: Update
```bash
./scripts/feeds update -a
```

Chức năng:
- Tải feeds từ các nguồn được liệt kê trong feed configuration
- Copy vào thư mục feeds/
- Tạo cấu trúc: feeds/<feed_name>/<package_name>/
- Lưu ý: Gói chưa được tích hợp vào build system (chưa có trong package/)

#### Step 2: Install
```bash
./scripts/feeds install -a
```

Chức năng:
- Tạo symbolic links từ feeds vào package/feeds/<feed_name>/
- Sau bước này có thể thực hiện các thao tác make cụ thể:
```bash
make package/feeds/<feed_name>/<package_name>
```

#### Ngoài ra còn có các lệnh khác trong `scripts/feeds`
```bash
$ ./scripts/feeds 
Usage: ./scripts/feeds <command> [options]

Commands:
	list [options]: List feeds, their content and revisions (if installed)
	Options:
	    -n :            List of feed names.
	    -s :            List of feed names and their URL.
	    -r <feedname>:  List packages of specified feed.
	    -d <delimiter>: Use specified delimiter to distinguish rows (default: spaces)
	    -f :            List feeds in feeds.conf compatible format (when using -s).

	install [options] <package>: Install a package
	Options:
	    -a :           Install all packages from all feeds or from the specified feed using the -p option.
	    -p <feedname>: Prefer this feed when installing packages.
	    -d <y|m|n>:    Set default for newly installed packages.
	    -f :           Install will be forced even if the package exists in core OpenWrt (override)

	search [options] <substring>: Search for a package
	Options:
	    -r <feedname>: Only search in this feed

	uninstall -a|<package>: Uninstall a package
	Options:
	    -a :           Uninstalls all packages.

	update -a|<feedname(s)>: Update packages and lists of feeds in feeds.conf .
	Options:
	    -a :           Update all feeds listed within feeds.conf. Otherwise the specified feeds will be updated.
	    -i :           Recreate the index only. No feed update from repository is performed.
	    -f :           Force updating feeds even if there are changed, uncommitted files.

	clean:             Remove downloaded/generated files.
```

### 3. Các thư mục được tạo ra trong quá trình Build System
#### 3.1 Thư mục `tmp/`
Được tạo ra tự động từ đầu, ngay khi config hoặc build bất kỳ gì. Vai trò của nó để lưu trữ tạm thời các file cache và 1 số info khác => giúp tăng tốc build.

#### 3.2 Thư mục `feeds/`
Trong `package/` chỉ chứa 1 số `core-package` để đảm bảo hệ thống có thể build tối thiểu. Tại `feeds/` mới là nơi chứa toàn bộ hàng nghìn recipe cho các package mở rộng. Ví dụ như `feeds/packages/`, `feeds/luci/`, `feeds/routing/`, `feeds/telepony/` sẽ chứa các recipe được tải/cập nhật về `feeds/` từ các repo được khai báo trong file `feeds.conf.default`. Được cập nhật và tải về qua 2 cmd:

```
./scripts/feeds update -a
./scripts/feeds install -a
```
#### 3.3 Thư mục `dl/`
Được tạo khi chạy `make download`, nơi lưu trữ (cache) tất cả các source code của kernel, toolchain và package mà mình sẽ compile trong quá trình **Build system**. Bên trong `dl` là các file nén (**.taz.gz, .taz.xz**) chứa source code. Khác với `feed/` ở chỗ, `dl/` chứa **nguyên liệu thô** `feed/` chứa **hướng dẫn sử dụng nguyên liệu thô** đó
#### 3.4 Thư mục `build_dir/`
Được tạo ra khi bắt đầu quá trình compile **host/target/toolchain**. Đây là nơi giải nén, patch, config, compile các source code trong OpenWrt (host, toolchain và target). Trong `build_dir` có các thư mục con:
- Thư mục `build_dir/host`: được tạo ra bởi lệnh
```bash
make tools/compile
```
hoặc khi `make world` bước vào giai đoạn `prepare` (build tool) để build các công cụ cho máy host.
- Thư mục `build_dir/toolchain-*`: được tạo bởi lệnh
```bash
make toolchain/compile
```
hoặc khi `make world` bước vào giai đoạn `prepare` (build toolchain) để build cross-toolchain.
- Thư mục `build_dir/target-*`: được tạo bởi lệnh
```bash
make target/compile
make package/compile
```
hoặc khi `make world` bước vào giai đoạn sau `prepare` để build kernel, rootfs, các package cho **target system**.


#### 3.5 Thư mục `staging_dir/`
Được tạo song song với `build_dir/`, nơi tập kết và tổ chức tất cả các sản phẩm đã build xong, sẵn sàng cho các bước tiếp theo trong quá trình build hoặc đóng gói.
#### 3.6 Thư mục `bin/`
Được tạo ra sau cùng, chỉ khi build thành công firmware cuối dùng (`make world`). Chứa **firmware images** và **package repository** (.ipk files)
#### 3.7 Các file key
Gồm các file như `key-build`, `key-build.pub`, `key-build.ucert`, `key-build.ucert.revoke`


### 4. Các bước để Build System
#### Bước 1: Setup và Configuration basic
```bash
# Clone source code
git clone https://git.openwrt.org/openwrt/openwrt.git
cd openwrt
git checkout openwrt-21.02

# Update feeds
./scripts/feeds update -a
./scripts/feeds install -a
```
**Kết quả**: Tạo thư mục `feeds/` chứa metadata các package.

#### Bước 2: Configuration target
```bash
# Configure target
make menuconfig 
```
Được định nghĩa trong `include/toplevel.mk`, khi chạy lệnh sẽ mở ra **Build System Configuration Interface**, một số yêu cầu cấu hình:
- Target System
- Subtarget
- Target Profile
- Package selection
- Build system settings
- Kernel modules

**Kết quả**: File `.config` được tạo với cấu hình build. Tạo thư mục `tmp/`

#### Bước 3: Build Process
```bash
make defconfig download clean world
```
Dưới đây là chi tiết quy trình build
##### Phase 3.1: Update, dowload và clean
- `make defconfig`: Được định nghĩa trong `include/toplevel.mk`, phục vụ cho việc tạo hoặc update `.config` file dựa trên **default configuration (defconfig)** cho **target** đã chọn (điền tùy chọn còn thiếu và gỡ bỏ mục đã lỗi thời). Ở đây `tmp/` được mở rộng. => Hoàn thành việc Configuration.
- `make download`: Được định nghĩa trong `include/toplevel.mk`, dùng để tải tất cả source code (**kernel, toolchain, packages**) về thư mục `dl/` trước khi compile => giúp mình có thể build ngoại tuyến. Tại đây `dl/` được tạo.
- `make clean`: Được định nghĩa trong **root Makefile**, sử dụng để xóa toàn bộ sản phẩm đã compile cũ (`bin/`, `build_dir/`...) nhưng giữ lại toolchain và `dl/` => là bước dọn dẹp để đảm bảo build mới không dùng nhầm cái cũ.

##### Phase 3.2: Build
```bash
make world
``` 
Được định nghĩa trong **root Makefile**, sử dụng để compile trọn vẹn firmware. Trong `make world` cũng bao gồm các giai đoạn:

**3.2.1: Compile tools và toolchain (`prepare`)**:
- Build Host Tools:
	```bash
	make tools/compile
	```
	**Kết quả tạo**:
	- `build_dir/host/`: Thư mục các tools đã compile. Ví dụ như `cmake/`, `python3/`...
	- `staging_dir/host/`: Thư mục các tools đã được install. Ví dụ như `bin/cmake`, `bin/python3/`...

- Build Cross-Toolchain:
	```bash
	make toolchain/compile
	```
	**Kết quả tạo**:
	- `build_dir/toolchain-*/`: Thư mục các cross-compiler đã compile.
	- `staging_dir/toolchain-*/`: Thư mục các cross-compiler đã install.

**3.2.2: Compile target và package**
```bash
make target/compile
make package/compile
```
**Kết quả tạo**:
- `build_dir/target-*/`: Thư mục kernel và package đã compile.
- `staging_dir/target-*/`: Thư mục kernel và package đã install.

**3.2.3: Assembly và Packaging**
```bash
make package/install # Dùng để assemble rootfs từ staged components
make target/install # Dùng để tạo firmware images từ rootfs
make package/index
```

**Kết quả tạo**:
- `bin/target`: Firmware images (.bin files)
- `bin/package`: Package repository (.ipk files + index)

### 5. UCI system (Unified Configuration Interface)
**UCI** là hệ thống cấu hình trung tâm cho tất cả các dịch vụ OpenWrt bao gồm **Network settings, Wireless configuration, Firewall rules, System services, Package configurations...**
#### 5.1 Nguyên tắc chung


### 6. UBUS system (OpenWrt Micro Bus)

### 7. PROCD (Process Management Daemon)