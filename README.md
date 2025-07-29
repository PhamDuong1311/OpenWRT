# OpenWRT

## 1. Build system essentials
**Build system** là tập hợp các Makefile và các Patch được nhà sản xuất thiết kế để tự động hóa toàn bộ quá trình build 1 firmware hoàn chỉnh cho router. Hệ thống thực hiện các bước:
- Tạo 1 **cross-compiler toolchain**
- Dùng toolchain đó để compile **Linux kernel**, **root filesystem**...

### 1.1 Cross-compiler toolchain
Thông thường, khi lập trình trên máy tính, việc build và run đều chạy trên máy tính đó. Tuy nhiên với router hoặc các thiết bị nhúng khác sử dụng kiến trúc CPU hoàn toàn khác.

=> Sử dụng **cross-compiler** để build firmware trên máy tính và run trên router. Để làm được thì cần 1 bộ công cụ đặc biệt gọi là **cross-compiler toolchain**, bao gồm:
- Compiler (GCC)
- Các tiện ích binary (Assembler, Linker)
- Thư viện C chuẩn (glibc)

### 1.2 Directory structure
Source code để build từ repo chính thức của OpenWRT gồm 7 folder, trong đó có 4 folder phục vụ cho **build system**:
- **/config**: chứa các file cấu hình cho **menuconfig**
- **/include**: chứa các tệp dạng **.mk** định nghĩa các hàm và quy chắc chung. Các Makefile của toàn bộ hệ thống sử dụng các **.mk** này để sử dụng lại các logic chung. (giống các header file trong source C file)
- **/package - BS**: chứa các **recipe nấu ăn** để nấu các gói phần mềm như **luci, htop, openssl, wget...** Mỗi  thư mục con tương ứng với 1 gói phần mềm. Hầu hết mọi thành phần trong firmware OpenWRT đều đóng gói dạng **.ipk**
- **/script**: chứa các script dạng **.sh, .py, .per** để cho phép người dùng tải hoặc cập nhật các package
- **/target - BS**: đây là thư mục cực kỳ quan trọng, chứa mọi thứ liên quan đến phần cứng cụ thể (router) mà bạn muốn build firmware:
    - **/linux**: chứa các cấu hình kernel và các bản vá **.patch** cần thiết để kernel Linux có thể hoạt động trên phần cứng đó.
    - **/image**: chứa các Makefile mô tả quy trình đống gói các thành phần đã biên dịch (kernel, rootfs) thành 1 tệp firmware hoàn chỉnh với định dạng đúng cho thiết bị.
- **/toolchain - BS**: chứa các Makefile để build bộ **cross-compiler toolchain** bao gồm Compiler, C library, Binary utilities. Quá trình này tạo ra 2 folder mới quan trọng:
    - **/toolchain_build_<arch>**: Thư mục làm việc tạm thời, diễn ra quá trình compile các thành phần trong toolchain.
    - **staging_dir**: Thư mục cài đặt cuối cùng. Bộ toolchain hoàn chỉnh sau biên dịch đặt ở đây. Các bước build sau này (như biên dịch kernel, package) sẽ sử dụng các công cụ từ thư mục này.
- **/tools - BS**: chứa các Makefile để build các công cụ cần thiết trên máy build như các tiện ích để xây dựng **toolchain** và các **package**, các công cụ để tạo **image firmware**. Bằng cách tự build các công cụ này, **OpenWRT** không phụ thuộc vào các ver sẵn có trên các **Linux distro** khác nhau => tránh lỗi không tương thích.

Các thư mục tạo ra trong quá trình build:
    - **/dl**: là thư mục cache chauws tất các cả các tệp mã nguồn nén được tải về trong quá trình build.
    - **/build_<arch>**: thư mục làm việc tạm thời trong quá trình compile các thành phần.

## 2. OpenWrt Feeds
Trong OpenWrt, **feeds** là kho lưu trữ các công thức build (**build recipes**) cho các package được định nghĩa trước cho **OpenWrt Buildroot**, cung cấp các package bổ sung cho **Build system** của OpenWrt.
### 2.1 Feed Configuration
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

### 2.2 Working with Feeds
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

Các folder được tạo ra sau bước này:
- `/feeds`
- `staging_dir`
- `tmp`

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

### 3. Created Directory within Build system
#### 3.1 Thư mục `build_dir/`

#### 3.2 Thư mục `staging_dir/`

#### 3.3 Thư mục `dl/`

#### 3.4 Thư mục `tmp/`



### 4. Target for Build system by Makefile
Dưới đây là 1 số cmd được sử dụng trong việc build:
- `make menuconfig`: Được định nghĩa trong `include/toplevel.mk`, khởi tạo **text-UI** để cấu hình build. Ở đây `tmp/` được tạo.
- `make defconfig`: Được định nghĩa trong `include/toplevel.mk`, phục vụ cho việc tạo hoặc update `.config` file dựa trên **default configuration (defconfig)** cho **target** đã chọn (điền tùy chọn còn thiếu và gỡ bỏ mục đã lỗi thời). Ở đây `tmp/` được mở rộng.
- `make download`: Được định nghĩa trong `include/toplevel.mk`, dùng để tải tất cả source (**kernel, toolchain, packages**) về thư mục `dl/` trước khi compile => giúp mình có thể build ngoại tuyến. Tại đây `dl/` được tạo.
- `make clean`: Được định nghĩa trong **root Makefile**, sử dụng để xóa toàn bộ sản phẩm đã compile (`bin/, build_dir/...`) nhưng **giữ lại toolchain** => là bước dọn dẹp để đảm bảo build mới không dùng nhầm cái cũ.
- `make world`: Được định nghĩa trong **root Makefile**, sử dụng để compile trọn vẹn firmware. Dưới đây là các bước thực hiện:
	- `prepare`: Chuẩn bị môi trường build:
		- Build **host tools** => tạo `staging_dir/host/`
		- Build **cross-compilation toolchain** => tạo `staging_dir/toolchain-*/`
	- `$(target/stamp-compile)`: Compile **Linux kernel** và config cho **target device** 
	- `$(package/stamp-compile)`: Compile **packages**
	- `$(package/stamp-install)`: Install **packages** đã compile vào `staging_dir`
	- `$(target/stamp-install)`: Assemble firmware
