## 1. Sơ đồ khối Lộ trình triển khai (Project Workflow)

```mermaid
flowchart TD
    %% =============================================================
    %% STYLING / PALETTE (Tối đa 3 màu, phong cách Open-source)
    %% =============================================================
    classDef stepClass      fill:#F8FAFC,stroke:#334155,stroke-width:1.2px,color:#0F172A;
    classDef highlightClass fill:#EFF6FF,stroke:#2563EB,stroke-width:1.5px,color:#1E3A8A;
    classDef finalizeClass  fill:#ECFDF5,stroke:#059669,stroke-width:1.5px,color:#064E3B;

    %% =============================================================
    %% CÁC GIAI ĐOẠN TRIỂN KHAI (5 PHASES)
    %% =============================================================
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

    %% =============================================================
    %% LIÊN KẾT TUẦN TỰ GIỮA CÁC GIAI ĐOẠN
    %% =============================================================
    G1 -->|"Hoàn tất Base Project"| G2
    G2 -->|"Services được kiểm chứng"| G3
    G3 -->|"Hệ thống tích hợp ổn định"| G4
    G4 -->|"Số liệu đo đạt chuẩn"| G5

    %% =============================================================
    %% GÁN CLASS STYLE
    %% =============================================================
    class A1,A2,A3,B1,B2,C1,C2 stepClass;
    class B3,C3,D1,D2,D3 highlightClass;
    class E1 finalizeClass;
```

---

## 2. Sơ đồ khối Kiến trúc phân lớp & Luồng dữ liệu (System Architecture)

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#F1F5F9', 'primaryBorderColor': '#64748B', 'primaryTextColor': '#0F172A', 'lineColor': '#334155', 'tertiaryColor': '#FFFFFF'}}}%%
flowchart TB
    classDef taskNode fill:#E0F2FE,stroke:#0284C7,stroke-width:1.5px,color:#0369A1,font-weight:600,font-size:12px;
    classDef queueNode fill:#FEF3C7,stroke:#D97706,stroke-width:1.5px,color:#92400E,font-weight:600,font-size:12px;
    classDef srvNode fill:#ECFDF5,stroke:#059669,stroke-width:1.5px,color:#065F46,font-weight:600,font-size:12px;
    classDef bspNode fill:#F5F3FF,stroke:#7C3AED,stroke-width:1.5px,color:#5B21B6,font-weight:600,font-size:12px;
    classDef halNode fill:#F1F5F9,stroke:#475569,stroke-width:2px,color:#1E293B,font-weight:bold,font-size:12px;
    classDef groupTitle fill:#FFFFFF,stroke:#94A3B8,stroke-width:1.5px,stroke-dasharray: 4 2,color:#334155,font-weight:bold,font-size:13px;

    subgraph L1["Tầng 1: Application & FreeRTOS Tasks"]
        direction TB
        subgraph INGRESS["Thu nhận gói tin"]
            RXA["rx_net_a_task<br/>(Prio 4, 256W)"]
            RXB["rx_net_b_task<br/>(Prio 4, 256W)"]
        end
        Q_RX[["s_rx_queue (Static)"]]
        CRYPTO["crypto_task<br/>(Prio 2, 512W)"]
        Q_TX[["s_tx_queue (Static)"]]
        TX["tx_task<br/>(Prio 3, 256W)"]
        HB["heartbeat_task<br/>(Prio 1, 192W)"]
    end

    subgraph L2["Tầng 2: Functional Services & Utilities"]
        POOL["buffer_pool<br/>(10 x 550B Static Pool)"]
        CS["crypto_service<br/>(AES-128-CBC + PKCS#7)"]
        CODEC["frame_codec<br/>([len][IV 16B][cipher])"]
    end

    subgraph L3["Tầng 3: Board Support Package (BSP)"]
        WIZ["bsp_w5500<br/>(Dual W5500 + Mutex)"]
        SPI["bsp_spi<br/>(SPI1/2 DMA Wrapper)"]
        DIAG["bsp_dwt & bsp_watchdog<br/>(Cycle Count + IWDG 500ms)"]
    end

    subgraph L4["Tầng 4: Vendor Drivers & OS Kernel"]
        HAL["STM32F1xx HAL  |  FreeRTOS Kernel  |  WIZnet ioLibrary  |  tiny-AES"]
    end

    %% Luồng xử lý dữ liệu chính (Main Pipeline)
    INGRESS -->|"1. Push Frame"| Q_RX
    Q_RX -->|"2. Pop"| CRYPTO
    CRYPTO -->|"3. Encrypt & Push"| Q_TX
    Q_TX -->|"4. Pop & Send"| TX

    %% Gọi tầng Services
    CRYPTO --> CS
    CRYPTO --> CODEC
    INGRESS -.->|"Cấp phát buffer"| POOL
    TX -.->|"Giải phóng buffer"| POOL

    %% Giao tiếp ngoại vi (BSP & Hardware)
    INGRESS & TX --> WIZ
    HB --> DIAG
    WIZ --> SPI
    SPI & DIAG & CS --> HAL

    class RXA,RXB,CRYPTO,TX,HB taskNode;
    class Q_RX,Q_TX queueNode;
    class POOL,CS,CODEC srvNode;
    class WIZ,SPI,DIAG bspNode;
    class HAL halNode;
    class L1,L2,L3,L4,INGRESS groupTitle;
```
