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
    - **staging_dir_<arch>**: Thư mục cài đặt cuối cùng. Bộ toolchain hoàn chỉnh sau biên dịch đặt ở đây. Các bước build sau này (như biên dịch kernel, package) sẽ sử dụng các công cụ từ thư mục này.
- **/tools - BS**: chứa các Makefile để build các công cụ cần thiết trên máy build như các tiện ích để xây dựng **toolchain** và các **package**, các công cụ để tạo **image firmware**. Bằng cách tự build các công cụ này, **OpenWRT** không phụ thuộc vào các ver sẵn có trên các **Linux distro** khác nhau => tránh lỗi không tương thích.

Các thư mục tạo ra trong quá trình build:
    - **/dl**: là thư mục cache chauws tất các cả các tệp mã nguồn nén được tải về trong quá trình build.
    - **/build_<arch>**: thư mục làm việc tạm thời trong quá trình compile các thành phần.





