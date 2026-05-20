# Mạng Máy Tính Nâng Cao - Đồ Án GNS3

**Sinh viên:** Uông Nhật Nam  
**MSSV:** 22120221  
**Email:** 22120221@student.hcmus.edu.vn  

## Giới thiệu

Đồ án mô phỏng hệ thống mạng doanh nghiệp đa vùng (Multi-AS) trên **GNS3**, triển khai các giao thức định tuyến động **OSPF**, **RIPv2** và **BGP** kết hợp **Redistribute** để kết nối liên vùng.

## Sơ đồ mạng (Topology)

```
┌─────────────── AS100 (OSPF) ───────────────┐
│                                             │
│   PC1 ─── Switch1 ─── R1 ─── R2 ─────────┐ │
│              200.200.21.0/28      │       │ │
│                                   │       │ │
│                          R3 ──────┘       │ │
│                           │               │ │
└───────────────────────────┼───────────────┘ │
                            │                 │
                    ┌───────┼───────┐         │
                    │       │       │         │
              ┌─────▼──┐ ┌──▼─────┐│         │
              │  R4    │ │  R5    ││         │
              │ AS200  │ │ AS300  ││         │
              │ RIPv2  │ │ RIPv2  ││         │
              └────┬───┘ └───┬────┘│         │
                   │         │     │         │
                   R7        R6    │         │
                   │         │     │         │
               Switch3    Switch2  │         │
                   │         │     │         │
                  PC3       PC2    │         │
          220.220.21.0/28  210.210.21.0/28   │
                                              │
┌──────────────────────────────────────────────┘
│ BGP (AS100 ←→ AS200 ←→ AS300)
└──────────────────────────────────────────────
```

## Thiết bị sử dụng

| Thiết bị  | Loại                    | Số lượng |
|-----------|-------------------------|----------|
| Router    | Cisco 7200 (c7200)     | 7        |
| Switch    | Ethernet Switch (GNS3) | 3        |
| PC        | VPCS (GNS3)            | 3        |

## Bảng địa chỉ IP

### Phân vùng AS100 (OSPF)

| Thiết bị | Cổng   | Địa chỉ IP         | Subnet Mask        |
|----------|--------|--------------------|--------------------|
| PC1      | e0     | 200.200.21.2/28    | 255.255.255.240    |
| R1       | f0/0   | 200.200.21.1/28    | 255.255.255.240    |
| R1       | f1/0   | 100.100.21.1/30    | 255.255.255.252    |
| R1       | f1/1   | 110.110.21.1/30    | 255.255.255.252    |
| R2       | f0/0   | 100.100.21.2/30    | 255.255.255.252    |
| R2       | f1/0   | 120.120.21.1/30    | 255.255.255.252    |
| R3       | f0/0   | 110.110.21.2/30    | 255.255.255.252    |
| R3       | f1/0   | 160.160.21.1/30    | 255.255.255.252    |

### Phân vùng AS300 (RIPv2)

| Thiết bị | Cổng   | Địa chỉ IP         | Subnet Mask        |
|----------|--------|--------------------|--------------------|
| PC2      | e0     | 210.210.21.2/28    | 255.255.255.240    |
| R6       | f0/0   | 210.210.21.1/28    | 255.255.255.240    |
| R6       | f1/0   | 130.130.21.2/30    | 255.255.255.252    |
| R5       | f0/0   | 120.120.21.2/30    | 255.255.255.252    |
| R5       | f1/0   | 130.130.21.1/30    | 255.255.255.252    |

### Phân vùng AS200 (RIPv2)

| Thiết bị | Cổng   | Địa chỉ IP         | Subnet Mask        |
|----------|--------|--------------------|--------------------|
| PC3      | e0     | 220.220.21.2/28    | 255.255.255.240    |
| R7       | f0/0   | 220.220.21.1/28    | 255.255.255.240    |
| R7       | f1/0   | 140.140.21.2/30    | 255.255.255.252    |
| R4       | f0/0   | 160.160.21.2/30    | 255.255.255.252    |
| R4       | f1/0   | 140.140.21.1/30    | 255.255.255.252    |

## Các giao thức đã triển khai

### 1. Cấu hình địa chỉ IP
Cấu hình IP tĩnh trên tất cả các cổng router và PC theo bảng địa chỉ trên.

### 2. OSPF (AS100)
- Triển khai OSPF Area 0 trên các router R1, R2, R3
- Quảng bá các mạng nội bộ trong AS100
- Redistribute BGP vào OSPF để học các mạng từ AS khác

### 3. RIPv2 (AS200 & AS300)
- Triển khai RIPv2 trên các router R4, R5, R6, R7
- Tắt auto-summary để tránh mất gói tin giữa các major network
- Redistribute BGP vào RIP để phân phối tuyến liên vùng

### 4. BGP (Border Gateway Protocol)
- Thiết lập BGP peering giữa các router biên:
  - **R2 (AS100)** ↔ **R5 (AS300)**: kết nối AS100 và AS300
  - **R3 (AS100)** ↔ **R4 (AS200)**: kết nối AS100 và AS200
  - **R2 (AS100)** ↔ **R3 (AS100)**: iBGP trong AS100
- Quảng bá mạng và redistribute giữa BGP và IGP (OSPF/RIP)

### 5. Redistribute
- **R2**: `redistribute bgp 100 subnets` trong OSPF
- **R3**: `redistribute bgp 100 subnets` trong OSPF
- **R5**: `redistribute bgp 300 metric 3` trong RIP
- **R4**: `redistribute bgp 200 metric 3` trong RIP

## Kết quả

- PC1 (AS100) có thể ping thành công tới PC2 (AS300) và PC3 (AS200)
- PC2 (AS300) có thể ping thành công tới PC1 (AS100) và PC3 (AS200)
- PC3 (AS200) có thể ping thành công tới PC1 (AS100) và PC2 (AS300)
- Tất cả các router có bảng định tuyến đầy đủ, học được mạng từ các AS khác thông qua BGP

## Cấu trúc thư mục

```
.
├── README.md                    # Tài liệu mô tả dự án
├── Report.pdf                   # Báo cáo chi tiết (PDF)
├── topology.gns3                # File topology GNS3 (mở bằng GNS3)
├── .gitignore
└── configs/
    ├── AS100-OSPF/
    │   ├── R1_startup-config.cfg
    │   ├── R2_startup-config.cfg
    │   └── R3_startup-config.cfg
    ├── AS200-RIPv2/
    │   ├── R4_startup-config.cfg
    │   └── R7_startup-config.cfg
    ├── AS300-RIPv2/
    │   ├── R5_startup-config.cfg
    │   └── R6_startup-config.cfg
    └── PCs/
        ├── PC1.vpc
        ├── PC2.vpc
        └── PC3.vpc
```

## Hướng dẫn mở lại project trên GNS3

1. Cài đặt [GNS3](https://www.gns3.com/) + GNS3 VM
2. Import file `topology.gns3` vào GNS3
3. Cấu hình path tới Cisco IOS image `c7200-adventerprisek9-mz.124-24.T5.image` (không được bao gồm trong repo do bản quyền)
4. Start tất cả các thiết bị
5. Các router sẽ tự động load startup-config từ thư mục `configs/`

> **Lưu ý:** Cisco IOS image không được bao gồm trong repository này vì lý do bản quyền. Bạn cần tự cung cấp image hợp lệ.

## Kỹ năng thể hiện

- Thiết kế và triển khai mạng IP với subnetting (/28, /30)
- Cấu hình Cisco IOS (CLI)
- Triển khai OSPF (Single Area)
- Triển khai RIPv2
- Triển khai BGP (eBGP & iBGP)
- Route Redistribution giữa các giao thức định tuyến
- Khắc phục sự cố kết nối mạng (troubleshooting)
- Sử dụng GNS3 để mô phỏng mạng

## Tài liệu tham khảo

1. Hướng dẫn thực hành OSPF
2. Hướng dẫn thực hành RIPv1, RIPv2
3. Hướng dẫn thực hành Router – Static Route
4. Hướng dẫn thực hành BGP
5. [Sơ lược về giao thức định tuyến BGP - VNPro](https://vnpro.vn/thu-vien/so-luoc-ve-giao-thuc-dinh-tuyen-bgp-2061.html)
6. [Cấu hình định tuyến động OSPF - VNPro](https://vnpro.vn/tin-tuc/cau-hinh-dinh-tuyen-dong-ospf-1151.html)
