
## Source NAT (SNAT) - Chi tiết toàn diện

### Định nghĩa và mục đích

**Source NAT (SNAT)** là quá trình thay đổi địa chỉ IP nguồn (source IP address) và có thể cả port nguồn của gói tin khi nó đi qua NAT device. SNAT được thiết kế chủ yếu để cho phép các thiết bị trong mạng private truy cập ra Internet.

### Cơ chế hoạt động chi tiết

**Giai đoạn 1: Outbound Traffic Processing**

1. **Client khởi tạo kết nối:**

```
PC nội bộ: 192.168.1.100:3000 → Internet: 8.8.8.8:53
```

2. **NAT device nhận gói tin:**
    - Kiểm tra source IP: `192.168.1.100` (IP private)
    - Kiểm tra source port: `3000` (port ngẫu nhiên của client)
    - Kiểm tra destination: `8.8.8.8:53` (Google DNS)
3. **Thực hiện SNAT translation:**
    - **Thay đổi Source IP**: `192.168.1.100` → `203.113.45.1` (IP public của router)
    - **Thay đổi Source Port**: `3000` → `50001` (port mới do NAT gán)
    - **Giữ nguyên Destination**: `8.8.8.8:53`
4. **Tạo entry trong NAT Table:**

```
Inside Local: 192.168.1.100:3000
Inside Global: 203.113.45.1:50001
Outside Global: 8.8.8.8:53
Protocol: UDP
Timeout: 300 seconds
```

5. **Gửi gói tin đã được translate:**

```
Router: 203.113.45.1:50001 → Internet: 8.8.8.8:53
```


**Giai đoạn 2: Return Traffic Processing**

1. **Server Internet phản hồi:**

```
Internet: 8.8.8.8:53 → Router: 203.113.45.1:50001
```

2. **NAT device tra cứu bảng:**
    - Tìm entry có `Inside Global: 203.113.45.1:50001`
    - Lấy thông tin `Inside Local: 192.168.1.100:3000`
3. **Thực hiện reverse SNAT:**
    - **Thay đổi Destination IP**: `203.113.45.1` → `192.168.1.100`
    - **Thay đổi Destination Port**: `50001` → `3000`
    - **Giữ nguyên Source**: `8.8.8.8:53`
4. **Chuyển tiếp đến client:**

```
Internet: 8.8.8.8:53 → PC nội bộ: 192.168.1.100:3000
```


### Các loại SNAT

**1. Static SNAT**

- Ánh xạ cố định một IP private với một IP public
- Sử dụng khi cần địa chỉ IP public cố định

```
192.168.1.10 → 203.113.45.10 (luôn cố định)
192.168.1.20 → 203.113.45.20 (luôn cố định)
```

**2. Dynamic SNAT (Pool SNAT)**

- Sử dụng pool IP public để gán động
- IP được chọn từ pool khi có kết nối

```
Pool: 203.113.45.10 - 203.113.45.15
192.168.1.x → chọn IP có sẵn từ pool
```

**3. MASQUERADE SNAT**

- Sử dụng IP của interface external làm source IP
- Thường dùng khi IP public thay đổi (DHCP)

```
192.168.1.x → IP của interface eth0 (động)
```


### Bảng SNAT Table chi tiết

| Inside Local | Inside Global | Outside Local | Outside Global | Protocol | State | Timeout |
| :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| 192.168.1.10:3000 | 203.113.45.1:50001 | 8.8.8.8:53 | 8.8.8.8:53 | UDP | ACTIVE | 300s |
| 192.168.1.20:4000 | 203.113.45.1:50002 | 1.1.1.1:443 | 1.1.1.1:443 | TCP | ESTABLISHED | 3600s |
| 192.168.1.30:5000 | 203.113.45.1:50003 | 208.67.222.222:53 | 208.67.222.222:53 | UDP | ACTIVE | 300s |

### Ứng dụng thực tế của SNAT

**1. Internet Gateway cho mạng doanh nghiệp**

- Cho phép hàng trăm/nghìn PC truy cập Internet qua một IP public
- Tiết kiệm chi phí IP public đáng kể

**2. Home Router**

- Router gia đình sử dụng SNAT để chia sẻ kết nối Internet
- Tất cả thiết bị trong nhà (PC, smartphone, tablet) đều truy cập qua một IP public

**3. VPN Gateway**

- Client VPN kết nối vào mạng công ty
- SNAT che giấu IP thực của VPN client khi truy cập tài nguyên nội bộ

**4. Load Balancer (Backend Protection)**

- Load balancer sử dụng SNAT để che giấu IP của backend server
- Client chỉ nhìn thấy IP của load balancer


## Destination NAT (DNAT) - Chi tiết toàn diện

### Định nghĩa và mục đích

**Destination NAT (DNAT)** là quá trình thay đổi địa chỉ IP đích (destination IP address) và có thể cả port đích của gói tin khi nó đi qua NAT device. DNAT được sử dụng để "publish" các dịch vụ từ mạng private ra Internet.

### Cơ chế hoạt động chi tiết

**Giai đoạn 1: Inbound Traffic Processing**

1. **Client Internet khởi tạo kết nối:**

```
Internet Client: 8.8.8.8:45000 → Public Server: 203.113.45.1:80
```

2. **NAT device nhận gói tin:**
    - Kiểm tra destination IP: `203.113.45.1` (IP public)
    - Kiểm tra destination port: `80` (HTTP service)
    - Tra cứu DNAT rules
3. **Tìm DNAT rule phù hợp:**

```
Rule: 203.113.45.1:80 → 192.168.1.10:80
```

4. **Thực hiện DNAT translation:**
    - **Thay đổi Destination IP**: `203.113.45.1` → `192.168.1.10`
    - **Giữ nguyên Destination Port**: `80` (hoặc thay đổi nếu có rule)
    - **Giữ nguyên Source**: `8.8.8.8:45000`
5. **Tạo entry trong Connection Tracking:**

```
Outside Source: 8.8.8.8:45000
Inside Destination: 192.168.1.10:80
Public Destination: 203.113.45.1:80
State: NEW → ESTABLISHED
```

6. **Chuyển tiếp gói tin:**

```
Internet Client: 8.8.8.8:45000 → Internal Server: 192.168.1.10:80
```


**Giai đoạn 2: Response Traffic Processing**

1. **Server nội bộ phản hồi:**

```
Internal Server: 192.168.1.10:80 → Internet Client: 8.8.8.8:45000
```

2. **NAT device nhận response:**
    - Tra cứu connection tracking table
    - Tìm session tương ứng
3. **Thực hiện reverse DNAT:**
    - **Thay đổi Source IP**: `192.168.1.10` → `203.113.45.1`
    - **Giữ nguyên Source Port**: `80`
    - **Giữ nguyên Destination**: `8.8.8.8:45000`
4. **Gửi response về client:**

```
Public Server: 203.113.45.1:80 → Internet Client: 8.8.8.8:45000
```


### Các loại DNAT

**1. Simple Port Forwarding**

- Chuyển tiếp một port cụ thể từ IP public đến IP private

```
203.113.45.1:22  → 192.168.1.10:22  (SSH)
203.113.45.1:80  → 192.168.1.20:80  (HTTP)
203.113.45.1:443 → 192.168.1.20:443 (HTTPS)
```

**2. Port Translation DNAT**

- Thay đổi cả IP và port trong quá trình translation

```
203.113.45.1:8080 → 192.168.1.10:80   (Web admin)
203.113.45.1:2222 → 192.168.1.20:22   (SSH non-standard)
```

**3. Load Balancing DNAT**

- Phân phối traffic đến nhiều backend server

```
203.113.45.1:80 → 192.168.1.10:80 (Server 1 - 33%)
203.113.45.1:80 → 192.168.1.20:80 (Server 2 - 33%)
203.113.45.1:80 → 192.168.1.30:80 (Server 3 - 34%)
```

**4. Full Static DNAT (1:1 NAT)**

- Ánh xạ toàn bộ IP public đến một IP private

```
203.113.45.10:* → 192.168.1.100:* (tất cả port)
```


### Bảng DNAT Configuration và Tracking

**DNAT Rules Table:**


| Public IP:Port | Private IP:Port | Protocol | Service Type | Status |
| :-- | :-- | :-- | :-- | :-- |
| 203.113.45.1:80 | 192.168.1.10:80 | TCP | Web Server | Active |
| 203.113.45.1:443 | 192.168.1.10:443 | TCP | HTTPS Server | Active |
| 203.113.45.1:22 | 192.168.1.20:22 | TCP | SSH Server | Active |
| 203.113.45.1:25 | 192.168.1.30:25 | TCP | SMTP Server | Active |

**Active Connections Table:**


| Outside Source | Inside Destination | Connection ID | State | Established Time |
| :-- | :-- | :-- | :-- | :-- |
| 8.8.8.8:45000 | 192.168.1.10:80 | CONN001 | ESTABLISHED | 14:30:25 |
| 1.1.1.1:55000 | 192.168.1.20:22 | CONN002 | ESTABLISHED | 14:25:10 |
| 9.9.9.9:60000 | 192.168.1.30:25 | CONN003 | TIME_WAIT | 14:35:45 |

### Ứng dụng thực tế của DNAT

**1. Web Server Hosting**

```
Scenario: Công ty có web server nội bộ cần publish ra Internet
Configuration: 203.113.45.1:80 → 192.168.1.10:80
Benefit: Client Internet có thể truy cập website qua IP public
```

**2. Remote Access Services**

```
SSH Access: 203.113.45.1:22 → 192.168.1.20:22
RDP Access: 203.113.45.1:3389 → 192.168.1.30:3389
VNC Access: 203.113.45.1:5900 → 192.168.1.40:5900
```

**3. Mail Server Publishing**

```
SMTP: 203.113.45.1:25 → 192.168.1.50:25
IMAP: 203.113.45.1:143 → 192.168.1.50:143
POP3: 203.113.45.1:110 → 192.168.1.50:110
```

**4. Load Balancing và High Availability**

```
Primary: 203.113.45.1:80 → 192.168.1.10:80
Secondary: 203.113.45.1:80 → 192.168.1.20:80 (failover)
```


## So sánh SNAT vs DNAT - Bảng chi tiết

| Tiêu chí | Source NAT (SNAT) | Destination NAT (DNAT) |
| :-- | :-- | :-- |
| **Hướng traffic chính** | Outbound (từ LAN ra Internet) | Inbound (từ Internet vào LAN) |
| **Thành phần được thay đổi** | Source IP + Source Port | Destination IP + Destination Port |
| **Mục đích sử dụng** | Internet access cho mạng private | Service publishing ra public |
| **Ai khởi tạo kết nối** | Client nội bộ | Client từ Internet |
| **Cấu hình** | Thường automatic/dynamic | Cần manual configuration |
| **Tính bảo mật** | Cao (ẩn internal topology) | Thấp hơn (expose services) |
| **IP public cần thiết** | 1 IP cho nhiều client | 1 IP cho các service được publish |
| **Phụ thuộc firewall** | Thường tự động allow outbound | Cần explicit firewall rules |
| **Troubleshooting** | Dễ hơn (traffic từ trong ra) | Khó hơn (cần kiểm tra nhiều layer) |
| **Scaling** | Tốt (1 IP phục vụ nhiều client) | Giới hạn (phụ thuộc số port/IP) |

## Kết hợp SNAT và DNAT

Trong thực tế, **SNAT và DNAT thường được sử dụng kết hợp** trên cùng một NAT device:

**Ví dụ kết hợp:**

```
Router có IP public: 203.113.45.1

SNAT Rules (Outbound):
- 192.168.1.0/24 → 203.113.45.1 (tất cả client ra Internet)

DNAT Rules (Inbound):  
- 203.113.45.1:80 → 192.168.1.10:80 (Web server)
- 203.113.45.1:22 → 192.168.1.20:22 (SSH server)
```

**Workflow hoàn chỉnh:**

1. **Client nội bộ truy cập Internet**: Sử dụng SNAT
2. **Internet client truy cập web server**: Sử dụng DNAT
3. **Web server phản hồi**: Sử dụng reverse DNAT
4. **Traffic trở về client nội bộ**: Sử dụng reverse SNAT
