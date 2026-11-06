# Sơ đồ khối phần cứng

```mermaid
graph LR
    subgraph PC["Máy tính (Kiểm thử & Giám sát)"]
        NIC_A["Card mạng #1 (Mạng A)<br/>IP: 192.168.10.10/24"]
        NIC_B["Card mạng #2 (Mạng B)<br/>IP: 192.168.20.10/24"]
        USB_DBG["Cổng USB<br/>(ST-Link V2 & USB-to-TTL)"]
    end

    subgraph HW_NET_A["Phía Mạng A"]
        W5500_A["Module W5500 #1<br/>IP: 192.168.10.2/24<br/>TCP Main, TCP Sub, UDP"]
    end

    subgraph MCU["STM32F103C8T6 — Blue Pill (72 MHz)"]
        SPI1_BUS["SPI1 + DMA1<br/>CS: PA4 · RST: PA3 · INT: PA2"]
        CORE["FreeRTOS + AES-128-CBC<br/>Static Buffer Pool + IWDG"]
        SPI2_BUS["SPI2 + DMA1<br/>CS: PB12 · RST: PB11 · INT: PB10"]
        SWD_UART["SWD: PA13, PA14<br/>USART1: PA9, PA10"]
    end

    subgraph HW_NET_B["Phía Mạng B"]
        W5500_B["Module W5500 #2<br/>IP: 192.168.20.2/24<br/>TCP Main, TCP Sub, UDP"]
    end

    PWR["Nguồn rời 3.3V<br/>Chung GND toàn hệ thống"]

    NIC_A <==>|RJ45 #1| W5500_A
    W5500_A <==>|SPI1| SPI1_BUS
    SPI1_BUS <==> CORE
    CORE <==> SPI2_BUS
    SPI2_BUS <==>|SPI2| W5500_B
    W5500_B <==>|RJ45 #2| NIC_B

    USB_DBG -.->|Nạp code và log UART1| SWD_UART
    PWR ==>|3.3V và GND| W5500_A
    PWR ==>|3.3V và GND| W5500_B
    PWR -.->|Chung GND| MCU
```

## Lý do thiết kế

- Dùng nguồn 3.3 V rời cho hai W5500 vì khi cả hai truyền/nhận, dòng tổng có thể khoảng 300–350 mA.
- Dùng SPI1 và SPI2 riêng để tách Mạng A và Mạng B ở mức phần cứng.
- Toàn bộ hệ thống phải dùng chung GND.
graph TB
    subgraph L1 ["TẦNG 1: APPLICATION & TASKS (App/ & Tasks/)"]
        APP["app_main.c<br/>Khởi tạo hệ thống & Static RTOS Objects"]
        T_RX1["rx_spi1_task<br/>(Priority: 4)"]
        T_RX2["rx_spi2_task<br/>(Priority: 4)"]
        T_PROC["packet_proc_task<br/>(Priority: 3)"]
        T_TX["tx_task<br/>(Priority: 3)"]
        T_HB["heartbeat_task<br/>(Priority: 2 - Chu kỳ 200ms)"]
        T_DIAG["diagnostic_task<br/>(Priority: 1)"]
    end

    subgraph IPC ["CƠ CHẾ ĐỒNG BỘ & HÀNG ĐỢI TĨNH (FreeRTOS Native IPC)"]
        NOTIFY1(["Task Notify (EXTI2)"])
        NOTIFY2(["Task Notify (EXTI10)"])
        Q_RX[["s_rx_queue<br/>(10 phần tử - Truyền buffer_index)"]]
        Q_TX[["s_tx_queue<br/>(10 phần tử - Truyền buffer_index)"]]
        POOL[("Static Buffer Pool<br/>10 x 550 Bytes")]
    end

    subgraph L2 ["TẦNG 2: SERVICES (Services/)"]
        S_RELAY["relay_service<br/>Quản lý 3 Sockets/Chip, Timeout 5000ms,<br/>Đóng gói khung & Static Buffer Pool"]
        S_CRYPTO["crypto_service<br/>PKCS#7 Padding + xorshift32 IV<br/>+ AES-128-CBC (tiny-AES)"]
        S_LOG["log_service<br/>Xuất log RELAY DEBUG & Đo chu kỳ DWT"]
    end

    subgraph L3 ["TẦNG 3: BOARD SUPPORT PACKAGE (BSP/)"]
        B_W5500["bsp_w5500 (wiz_port)<br/>Đa hợp 2 chip W5500 + s_wiz_mutex"]
        B_SPI["bsp_spi<br/>SPI1/SPI2 + DMA1 Transmit/Receive"]
        B_GPIO["bsp_gpio<br/>CS, RST, EXTI Callbacks"]
        B_WDG["bsp_watchdog<br/>IWDG (Timeout 500ms)"]
        B_UART["bsp_uart<br/>UART1 Wrapper"]
    end

    subgraph L4 ["TẦNG 4: VENDOR / HAL / HARDWARE (Core/, Drivers/, Middlewares/)"]
        HAL["STM32F1xx HAL + FreeRTOS Kernel + WIZnet ioLibrary"]
    end

    NOTIFY1 --> T_RX1
    NOTIFY2 --> T_RX2
    T_RX1 -->|"Cấp phát & Ghi bản rõ"| POOL
    T_RX2 -->|"Cấp phát & Ghi bản rõ"| POOL
    T_RX1 -->|"Đẩy index"| Q_RX
    T_RX2 -->|"Đẩy index"| Q_RX
    Q_RX --> T_PROC
    T_PROC <-->|"Đọc bản rõ / Ghi bản mã"| POOL
    T_PROC --> S_CRYPTO
    T_PROC -->|"Đẩy index"| Q_TX
    Q_TX --> T_TX
    T_TX -->|"Đọc bản mã & Giải phóng"| POOL

    T_RX1 & T_RX2 & T_TX & T_HB --> S_RELAY
    T_HB --> B_WDG
    T_DIAG --> S_LOG

    S_RELAY --> B_W5500
    B_W5500 --> B_SPI & B_GPIO
    S_LOG --> B_UART
    B_SPI & B_GPIO & B_WDG & B_UART --> HAL
