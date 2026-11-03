# Secure Gateway Project

## 1. Development Workflow

Quy trình phát triển và hoàn thiện hệ thống gồm 5 giai đoạn tuần tự: chuẩn hóa cấu trúc, kiểm thử dịch vụ trên PC, tích hợp BSP & FreeRTOS, đo kiểm thực nghiệm và nghiệm thu mã nguồn.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, sans-serif', 'fontSize': '13px' }}}%%
flowchart TD
    %% =============================================================
    %% BẢNG MÀU TƯƠNG PHẢN CAO - DỄ ĐỌC TRÊN GITHUB
    %% =============================================================
    classDef default font-size:12px,color:#0F172A;
    classDef stepClass      fill:#F1F5F9,stroke:#475569,stroke-width:2px,color:#0F172A,font-weight:500;
    classDef highlightClass fill:#DBEAFE,stroke:#1D4ED8,stroke-width:2px,color:#1E3A8A,font-weight:600;
    classDef finalizeClass  fill:#DCFCE7,stroke:#15803D,stroke-width:2px,color:#14532D,font-weight:600;

    %% =============================================================
    %% CÁC GIAI ĐOẠN (HÀNG NGANG)
    %% =============================================================
    subgraph G1[" Phase 1: Khởi tạo & Chuẩn hóa cấu trúc "]
        direction LR
        A1["<b>Cấu hình CubeMX (.ioc)</b><br/>SPI1/2, DMA, EXTI, IWDG, FreeRTOS"] --> A2["<b>Dựng cây thư mục SW-STD-001</b><br/>App, BSP, Services, Tasks, Common, Config"]
        A2 --> A3["<b>Thiết lập công cụ</b><br/>.clang-format, .gitignore, GCC Warnings"]
    end

    subgraph G2[" Phase 2: Tầng Services & Unit Test PC "]
        direction LR
        B1["<b>Triển khai buffer_pool</b><br/>10 Static Buffers x 550B"] --> B2["<b>Triển khai crypto & frame</b><br/>PKCS#7, xorshift32 IV, AES-128-CBC"]
        B2 --> B3["<b>Unit Test trên PC (MinGW)</b><br/>Đối chiếu PyCryptodome"]
    end

    subgraph G3[" Phase 3: Tầng BSP & Tích hợp FreeRTOS "]
        direction LR
        C1["<b>Triển khai bsp ngoại vi</b><br/>SPI DMA, s_wiz_mutex, Backup Socket"] --> C2["<b>Tạo 5 Tasks cấp phát tĩnh</b><br/>RX_A, RX_B, Crypto, TX, Heartbeat"]
        C2 --> C3["<b>Ghép nối 2x W5500</b><br/>Kiểm thử mã hóa 2 chiều"]
    end

    subgraph G4[" Phase 4: Thực nghiệm & Đo kiểm "]
        direction LR
        D1["<b>Đo thời gian mã hóa DWT</b><br/>Payload 64B, 128B, 256B, 512B"] --> D2["<b>Đo thông lượng TCP & UDP</b><br/>50 gói/s + Wireshark"]
        D2 --> D3["<b>Đo độ tin cậy kết nối</b><br/>Mất khung 5-50%, Bền 1h"]
    end

    subgraph G5[" Phase 5: Nghiệm thu mã nguồn "]
        direction LR
        E1["<b>Nghiệm thu 16 tiêu chí SW-STD-001</b><br/>cppcheck (Static Analysis) • clang-format • Giải trình sai lệch (Deviations)"]
    end

    %% =============================================================
    %% LIÊN KẾT TUẦN TỰ
    %% =============================================================
    G1 ==> G2 ==> G3 ==> G4 ==> G5

    %% =============================================================
    %% ÁP DỤNG STYLE
    %% =============================================================
    class A1,A2,A3,B1,B2,C1,C2 stepClass;
    class B3,C3,D1,D2,D3 highlightClass;
    class E1 finalizeClass;
```

---

## 2. System Architecture

Kiến trúc hệ thống được phân lớp nghiêm ngặt, sử dụng 100% bộ nhớ tĩnh và giao tiếp luồng dữ liệu an toàn qua hàng đợi FreeRTOS giữa 2 giao diện mạng W5500.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontSize': '16px', 'fontFamily': 'ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, sans-serif' }}}%%
flowchart TB
    classDef default font-size:16px;
    classDef taskClass    fill:#1E293B,stroke:#38BDF8,stroke-width:2px,color:#F8FAFC,font-size:16px;
    classDef queueClass   fill:#0F172A,stroke:#94A3B8,stroke-width:2px,color:#38BDF8,font-size:16px;
    classDef serviceClass fill:#064E3B,stroke:#34D399,stroke-width:2px,color:#F0FDF4,font-size:16px;
    classDef bspClass     fill:#451A03,stroke:#FBBF24,stroke-width:2px,color:#FFFBEB,font-size:16px;
    classDef hwClass      fill:#18181B,stroke:#A1A1AA,stroke-width:2px,color:#F4F4F5,font-size:16px;

    subgraph L1[" Layer 1: FreeRTOS Tasks & Queues "]
        direction TB
        RXA["rx_net_a_task<br/>(Prio 4, Stack 256W)"]
        RXB["rx_net_b_task<br/>(Prio 4, Stack 256W)"]
        QRX[["s_rx_queue (Static)"]]
        CRYPTO["crypto_task<br/>(Prio 2, Stack 512W)"]
        QTX[["s_tx_queue (Static)"]]
        TX["tx_task<br/>(Prio 3, Stack 256W)"]
        HB["heartbeat_task<br/>(Prio 1, Stack 192W)"]

        RXA & RXB --> QRX
        QRX --> CRYPTO
        CRYPTO --> QTX
        QTX --> TX
    end

    subgraph L2[" Layer 2: Functional Services "]
        direction LR
        POOL[("buffer_pool<br/>(10 x 550B Static)")]
        CS["crypto_service<br/>(AES-128-CBC)"]
        CODEC["frame_codec<br/>([len][IV][data])"]
    end

    subgraph L3[" Layer 3: Board Support Package (BSP) "]
        direction LR
        WIZ["bsp_w5500<br/>(Dual W5500, Mutex)"]
        SPI["bsp_spi<br/>(SPI1/2 DMA)"]
        DIAG["bsp_dwt & bsp_watchdog<br/>(IWDG 500ms, DWT)"]

        WIZ --> SPI
    end

    subgraph L4[" Layer 4: Drivers & Hardware "]
        direction LR
        HAL["Vendor HAL & FreeRTOS Kernel"]
        HW["STM32F103 MCU + 2x W5500"]
        HAL --> HW
    end

    %% Giao tiếp giữa các tầng (gọn gàng, không bị chồng chéo)
    L1 -.->|"Acquire / Release"| POOL
    CRYPTO -->|"Encrypt / Pack"| CS & CODEC
    RXA & RXB & TX <-->|"Read / Write Socket"| WIZ
    HB -->|"Kick dog / DWT"| DIAG

    SPI & DIAG --> HAL

    class RXA,RXB,CRYPTO,TX,HB taskClass;
    class QRX,QTX queueClass;
    class POOL,CS,CODEC serviceClass;
    class WIZ,SPI,DIAG bspClass;
    class HAL,HW hwClass;
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
