
# Secure Gateway AES-128-CBC (STM32 & FreeRTOS)

Hệ thống cổng kết nối mã hóa hai chiều dữ liệu mạng qua TCP/UDP sử dụng STM32F103 và 2 chip W5500.

---

## 1. Sơ đồ khối Lộ trình triển khai (Project Workflow)

```mermaid
flowchart TD
    subgraph G1["Giai đoạn 1: Khởi tạo & Chuẩn hóa"]
        A1["Cấu hình CubeMX (.ioc)<br/>SPI1/2, DMA, EXTI, IWDG, FreeRTOS"] --> A2["Dựng cây thư mục SW-STD-001<br/>App, BSP, Services, Tasks, Common, Config"]
        A2 --> A3["Thiết lập Công cụ<br/>.clang-format, .gitignore, GCC Warnings"]
    end

    subgraph G2["Giai đoạn 2: Tầng Services & Unit Test PC"]
        B1["Viết buffer_pool.c/h<br/>10 Static Buffers x 550B"] --> B2["Viết crypto_service & frame_codec<br/>PKCS#7, xorshift32 IV, AES-128-CBC"]
        B2 --> B3["Chạy Unit Test trên PC (MinGW)<br/>Kiểm chứng với Python PyCryptodome"]
    end

    subgraph G3["Giai đoạn 3: Tầng BSP & Tích hợp FreeRTOS"]
        C1["Viết bsp_spi, bsp_w5500, bsp_dwt<br/>SPI DMA, s_wiz_mutex, Backup Socket"] --> C2["Viết 5 Tasks cấp phát tĩnh<br/>RX_A, RX_B, Crypto, TX, Heartbeat"]
        C2 --> C3["Ghép nối 2 module W5500<br/>Kiểm thử truyền nhận mã hóa 2 chiều"]
    end

    subgraph G4["Giai đoạn 4: Thực nghiệm & Đo kiểm"]
        D1["Đo thời gian mã hóa DWT<br/>Payload 64B, 128B, 256B, 512B"] --> D2["Đo thông lượng TCP & UDP<br/>Tốc độ đích 50 gói/giây + Wireshark"]
        D2 --> D3["Đo khôi phục TCP (n=20),<br/>Mất khung 5-50% & Bền 1 giờ"]
    end

    subgraph G5["Giai đoạn 5: Nghiệm thu mã nguồn"]
        E1["Nghiệm thu 16 mục SW-STD-001<br/>cppcheck + clang-format + deviations"]
    end

    G1 --> G2 --> G3 --> G4 --> G5
```

---

## 2. Sơ đồ khối Kiến trúc phân lớp & Luồng dữ liệu (System Architecture)

```mermaid
flowchart TB
    subgraph L1["Tầng 1: Application & FreeRTOS Tasks (App/ & Tasks/)"]
        RXA["rx_net_a_task<br/>(Prio 4, 256W)"]
        RXB["rx_net_b_task<br/>(Prio 4, 256W)"]
        Q_RX[["s_rx_queue<br/>(Static Queue)"]]
        CRYPTO["crypto_task<br/>(Prio 2, 512W)"]
        Q_TX[["s_tx_queue<br/>(Static Queue)"]]
        TX["tx_task<br/>(Prio 3, 256W)"]
        HB["heartbeat_task<br/>(Prio 1, 192W)"]
    end

    subgraph L2["Tầng 2: Functional Services (Services/ & Common/)"]
        POOL["buffer_pool<br/>(10 x 550B Static Pool)"]
        CS["crypto_service<br/>(PKCS#7 + xorshift32 + AES-128-CBC)"]
        CODEC["frame_codec<br/>([len][IV 16B][ciphertext])"]
    end

    subgraph L3["Tầng 3: Board Support Package (BSP/)"]
        WIZ["bsp_w5500<br/>(Dual W5500 + s_wiz_mutex + Backup Socket)"]
        SPI["bsp_spi<br/>(SPI1/SPI2 DMA Wrapper)"]
        DIAG["bsp_dwt & bsp_watchdog<br/>(DWT Cycle Counter + IWDG 500ms)"]
    end

    subgraph L4["Tầng 4: Vendor HAL & Third-Party (Core/, Drivers/, Middlewares/)"]
        HAL["STM32F1xx HAL + FreeRTOS Kernel + WIZnet ioLibrary + tiny-AES"]
    end

    RXA & RXB -->|"Acquire Buffer & Push"| Q_RX
    Q_RX --> CRYPTO
    CRYPTO -->|"Encrypt & Frame"| Q_TX
    Q_TX --> TX
    RXA & RXB & TX -.-> POOL
    CRYPTO --> CS & CODEC
    RXA & RXB & TX & HB --> WIZ
    HB --> DIAG
    WIZ --> SPI
    CS & SPI & DIAG --> HAL
```
