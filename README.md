# Secure Gateway Project

## 1. Development Workflow

Quy trình phát triển và hoàn thiện hệ thống gồm 5 giai đoạn tuần tự[cite: 1]: chuẩn hóa cấu trúc[cite: 1], kiểm thử dịch vụ trên PC[cite: 1], tích hợp BSP & FreeRTOS[cite: 1], đo kiểm thực nghiệm[cite: 1] và nghiệm thu mã nguồn[cite: 1].

```mermaid
flowchart TD
    classDef stepClass      fill:#F8FAFC,stroke:#334155,stroke-width:1.2px,color:#0F172A;
    classDef highlightClass fill:#EFF6FF,stroke:#2563EB,stroke-width:1.5px,color:#1E3A8A;
    classDef finalizeClass  fill:#ECFDF5,stroke:#059669,stroke-width:1.5px,color:#064E3B;

    subgraph G1["Phase 1: Khởi tạo & Chuẩn hóa cấu trúc"]
        direction TB
        A1["Cấu hình CubeMX (.ioc)<br/>• SPI1/2, DMA, EXTI, IWDG, FreeRTOS"] --> A2["Dựng cây thư mục SW-STD-001<br/>• App, BSP, Services, Tasks, Common, Config"]
        A2 --> A3["Thiết lập công cụ chuẩn hóa<br/>• .clang-format, .gitignore, GCC Warnings"]
    end

    subgraph G2["Phase 2: Tầng Services & Unit Test trên PC"]
        direction TB
        B1["Triển khai buffer_pool.c/.h<br/>• 10 Static Buffers x 550B"] --> B2["Triển khai crypto_service & frame_codec<br/>• PKCS#7, xorshift32 IV, AES-128-CBC"]
        B2 --> B3["Chạy Unit Test trên PC (MinGW)<br/>• Đối chiếu tính đúng đắn với PyCryptodome"]
    end

    subgraph G3["Phase 3: Tầng BSP & Tích hợp FreeRTOS"]
        direction TB
        C1["Triển khai bsp_spi, bsp_w5500, bsp_dwt<br/>• SPI DMA, s_wiz_mutex, Backup Socket"] --> C2["Tạo 5 Tasks cấp phát tĩnh<br/>• RX_A, RX_B, Crypto, TX, Heartbeat"]
        C2 --> C3["Ghép nối 2 module W5500<br/>• Kiểm thử truyền nhận mã hóa 2 chiều"]
    end

    subgraph G4["Phase 4: Thực nghiệm & Đo kiểm"]
        direction TB
        D1["Đo thời gian mã hóa DWT<br/>• Khảo sát Payload 64B, 128B, 256B, 512B"] --> D2["Đo thông lượng TCP & UDP<br/>• Tốc độ 50 gói/giây + Giám sát Wireshark"]
        D2 --> D3["Đo độ tin cậy đường truyền<br/>• Phục hồi TCP (n=20), Mất khung 5-50%, Bền 1h"]
    end

    subgraph G5["Phase 5: Nghiệm thu mã nguồn"]
        direction TB
        E1["Nghiệm thu 16 tiêu chí SW-STD-001<br/>• Static Analysis (cppcheck)<br/>• Formatting (clang-format)<br/>• Danh mục giải trình ngoại lệ (Deviations)"]
    end

    G1 -->|"Hoàn tất Base Project"| G2
    G2 -->|"Services được kiểm chứng"| G3
    G3 -->|"Hệ thống tích hợp ổn định"| G4
    G4 -->|"Số liệu đo đạt chuẩn"| G5

    class A1,A2,A3,B1,B2,C1,C2 stepClass;
    class B3,C3,D1,D2,D3 highlightClass;
    class E1 finalizeClass;
```

---

## 2. System Architecture

Kiến trúc hệ thống được phân lớp nghiêm ngặt, sử dụng 100% bộ nhớ tĩnh và giao tiếp luồng dữ liệu an toàn qua hàng đợi FreeRTOS giữa 2 giao diện mạng W5500[cite: 1].

```mermaid
flowchart TB
    classDef taskClass    fill:#EBF3FB,stroke:#1E40AF,stroke-width:1.5px,color:#0F172A;
    classDef serviceClass fill:#ECFDF5,stroke:#047857,stroke-width:1.5px,color:#064E3B;
    classDef bspClass     fill:#FFFBEB,stroke:#B45309,stroke-width:1.5px,color:#78350F;

    subgraph L1["Layer 1: FreeRTOS Tasks & IPC (App/ & Tasks/)"]
        RXA["rx_net_a_task<br/>(Prio 4, Stack 256W)"]
        RXB["rx_net_b_task<br/>(Prio 4, Stack 256W)"]
        QRX[["s_rx_queue<br/>(Static Queue)"]]
        CRYPTO["crypto_task<br/>(Prio 2, Stack 512W)"]
        QTX[["s_tx_queue<br/>(Static Queue)"]]
        TX["tx_task<br/>(Prio 3, Stack 256W)"]
        HB["heartbeat_task<br/>(Prio 1, Stack 192W)"]
    end

    subgraph L2["Layer 2: Functional Services (Services/ & Common/)"]
        POOL[("buffer_pool<br/>(10 x 550B Static Pool)")]
        CS["crypto_service<br/>(PKCS#7, xorshift32, AES-128-CBC)"]
        CODEC["frame_codec<br/>([len][IV 16B][ciphertext])"]
    end

    subgraph L3["Layer 3: Board Support Package (BSP/)"]
        WIZ["bsp_w5500<br/>(Dual W5500, s_wiz_mutex, Backup Socket)"]
        SPI["bsp_spi<br/>(SPI1 / SPI2 DMA Wrapper)"]
        DIAG["bsp_dwt & bsp_watchdog<br/>(DWT Cycle Counter, IWDG 500ms)"]
    end

    subgraph L4["Layer 4: Vendor Drivers & Hardware"]
        HAL["Vendor HAL & Middlewares<br/>(STM32F1xx HAL, FreeRTOS, ioLibrary, tiny-AES)"]
        HW["Hardware Layer<br/>(STM32F103C8T6 MCU, Dual W5500 Modules)"]
    end

    RXA & RXB -->|"Post buffer desc"| QRX
    QRX -->|"Pop packet"| CRYPTO
    CRYPTO -->|"Post framed msg"| QTX
    QTX -->|"Pop frame"| TX

    RXA & RXB -.->|"Acquire buffer"| POOL
    TX -.->|"Release buffer"| POOL

    CRYPTO -->|"AES Encrypt/Decrypt"| CS
    CRYPTO -->|"Pack/Unpack"| CODEC

    RXA & RXB -->|"Read socket"| WIZ
    TX -->|"Write socket"| WIZ
    HB -->|"Kick dog / DWT read"| DIAG

    WIZ -->|"SPI DMA Transfer"| SPI
    SPI & DIAG -->|"HAL Drivers"| HAL
    HAL -->|"Physical Interface"| HW

    class RXA,RXB,QRX,CRYPTO,QTX,TX,HB taskClass;
    class POOL,CS,CODEC serviceClass;
    class WIZ,SPI,DIAG,HAL,HW bspClass;
```

---

## 3. Project Structure

Cấu trúc cây thư mục:

```text
├── App/                                  # Tầng ứng dụng cấp cao và điểm khởi chạy hệ thống
│   ├── app_main.c                        # Khởi tạo các static queue, task và phân phối tài nguyên
│   └── app_main.h                        # Interface điểm vào (entry point) của ứng dụng FreeRTOS
├── BSP/                                  # Board Support Package - Quản lý phần cứng ngoại vi
│   ├── bsp_dwt.c                         # Đo thời gian thực thi cấp chu kỳ xung nhịp CPU (DWT Cycle Counter)
│   ├── bsp_dwt.h                         # Interface đếm chu kỳ DWT phục vụ đo benchmark mã hóa
│   ├── bsp_spi.c                         # Driver SPI1/SPI2 chế độ DMA truyền nhận tốc độ cao
│   ├── bsp_spi.h                         # Interface điều khiển và ngắt DMA cho các kênh SPI
│   ├── bsp_w5500.c                       # Driver 2 chip W5500, socket dự phòng và s_wiz_mutex
│   ├── bsp_w5500.h                       # Interface khởi tạo socket và truyền nhận gói tin Ethernet
│   ├── bsp_watchdog.c                    # Driver phần cứng Independent Watchdog (IWDG chu kỳ 500ms)
│   └── bsp_watchdog.h                    # Interface làm mới watchdog (kick dog) định kỳ
├── Common/                               # Thư viện dùng chung, kiểu dữ liệu chuẩn và cấu hình tĩnh
│   ├── app_config.h                      # Định nghĩa kích thước buffer, độ dài queue, timeout
│   └── app_types.h                       # Định nghĩa kiểu dữ liệu chuẩn, cấu trúc bản tin và mã lỗi
├── Config/                               # File cấu hình hạt nhân hệ điều hành thời gian thực
│   └── FreeRTOSConfig.h                  # Cấu hình tiền định FreeRTOS: cấp phát tĩnh, tick rate, heap 0
├── Core/                                 # Mã nguồn khởi tạo vi điều khiển do STM32CubeMX tự sinh
│   ├── Inc/                              # Thư mục chứa header khởi tạo clock và ngoại vi STM32
│   └── Src/                              # Thư mục chứa main.c và cấu hình vector ngắt stm32f1xx_it.c
├── Drivers/                              # Thư viện driver phần cứng từ nhà sản xuất (ST)
│   ├── CMSIS/                            # Thư viện định nghĩa nhân ARM Cortex-M3 và thanh ghi
│   └── STM32F1xx_HAL_Driver/             # Thư viện ngoại vi STM32F1 HAL (SPI, GPIO, DMA, IWDG)
├── Middlewares/                          # Thư viện trung gian và phần mềm bên thứ ba
│   ├── FreeRTOS/                         # Mã nguồn nhân FreeRTOS Kernel (chế độ Static Allocation)
│   ├── tiny-AES/                         # Thư viện mã hóa AES-128-CBC tối ưu RAM/Flash cho vi điều khiển
│   └── WIZnet/                           # Bộ thư viện ioLibrary điều khiển chip mạng W5500
├── Services/                             # Tầng dịch vụ xử lý dữ liệu nghiệp vụ (Business Logic)
│   ├── buffer_pool.c                     # Quản lý 10 buffer tĩnh x 550B, cấp phát an toàn chống phân mảnh
│   ├── buffer_pool.h                     # Interface acquire/release buffer tĩnh an toàn đa luồng
│   ├── crypto_service.c                  # Thuật toán đệm PKCS#7, sinh IV xorshift32 và AES-128-CBC
│   ├── crypto_service.h                  # Interface dịch vụ mã hóa/giải mã dữ liệu
│   ├── frame_codec.c                     # Đóng gói và bóc tách cấu trúc khung tin [len][IV 16B][ciphertext]
│   └── frame_codec.h                     # Interface bộ mã hóa/giải mã khung truyền thông
├── Tasks/                                # Các tác vụ FreeRTOS độc lập (100% cấp phát tĩnh)
│   ├── crypto_task.c                     # Task xử lý mã hóa/giải mã dữ liệu (Prio 2, Stack 512W)
│   ├── crypto_task.h                     # Interface và hàm khởi tạo tĩnh crypto_task
│   ├── heartbeat_task.c                  # Task giám sát sống còn hệ thống & làm mới IWDG (Prio 1, Stack 192W)
│   ├── heartbeat_task.h                  # Interface và hàm khởi tạo tĩnh heartbeat_task
│   ├── rx_net_a_task.c                   # Task nhận dữ liệu từ giao diện mạng A (Prio 4, Stack 256W)
│   ├── rx_net_a_task.h                   # Interface và hàm khởi tạo tĩnh rx_net_a_task
│   ├── rx_net_b_task.c                   # Task nhận dữ liệu từ giao diện mạng B (Prio 4, Stack 256W)
│   ├── rx_net_b_task.h                   # Interface và hàm khởi tạo tĩnh rx_net_b_task
│   ├── tx_task.c                         # Task truyền khung tin ra giao diện mạng đích (Prio 3, Stack 256W)
│   └── tx_task.h                         # Interface và hàm khởi tạo tĩnh tx_task
├── tests/                                # Môi trường và kịch bản kiểm thử tự động
│   ├── pc_unit_tests/                    # Bộ Unit Test chạy trên PC (MinGW) kiểm thử logic Services
│   └── verify_pycryptodome.py            # Script Python đối chiếu tính đúng đắn với PyCryptodome
├── .clang-format                         # Tệp quy tắc định dạng code tuân thủ chuẩn SW-STD-001
├── .gitignore                            # Cấu hình loại trừ file build tạm (.o, .d) và workspace IDE
└── SecureGateway.ioc                     # Tệp cấu hình chân, clock, DMA, SPI trong STM32CubeMX
```
