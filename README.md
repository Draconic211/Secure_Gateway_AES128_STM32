# 1. Sơ đồ khối phần cứng

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

## 2. Lý do thiết kế

- Dùng nguồn 3.3 V rời cho hai W5500 vì khi cả hai truyền/nhận, dòng tổng có thể khoảng 300–350 mA.
- Dùng SPI1 và SPI2 riêng để tách Mạng A và Mạng B ở mức phần cứng.
- Toàn bộ hệ thống phải dùng chung GND.
## 3. Kiến trúc phần mềm, FreeRTOS và bộ nhớ

Phần mềm tuân thủ luồng phân lớp một chiều:

```text
Application & Tasks → Services → BSP → STM32 HAL / Vendor
```

```mermaid
graph TB
    subgraph L1["TẦNG 1: APPLICATION & TASKS"]
        APP["app_main.c<br/>Khởi tạo hệ thống và Static RTOS Objects"]
        T_RX1["rx_spi1_task<br/>Priority 4"]
        T_RX2["rx_spi2_task<br/>Priority 4"]
        T_PROC["packet_proc_task<br/>Priority 3"]
        T_TX["tx_task<br/>Priority 3"]
        T_HB["heartbeat_task<br/>Priority 2, chu kỳ 200 ms"]
        T_DIAG["diagnostic_task<br/>Priority 1"]
    end

    subgraph IPC["FREE RTOS IPC VÀ STATIC BUFFER"]
        NOTIFY1(["Task Notify từ EXTI2"])
        NOTIFY2(["Task Notify từ EXTI10"])
        Q_RX[["s_rx_queue<br/>10 buffer index"]]
        Q_TX[["s_tx_queue<br/>10 buffer index"]]
        POOL[("Static Buffer Pool<br/>10 buffers")]
    end

    subgraph L2["TẦNG 2: SERVICES"]
        S_RELAY["relay_service<br/>Socket, timeout, buffer ownership"]
        S_CRYPTO["crypto_service<br/>PKCS#7 + IV + AES-128-CBC"]
        S_LOG["log_service<br/>RELAY DEBUG và DWT"]
    end

    subgraph L3["TẦNG 3: BSP"]
        B_W5500["bsp_w5500 / wiz_port<br/>2 W5500 và wiz_mutex"]
        B_SPI["bsp_spi<br/>SPI1/SPI2 + DMA"]
        B_GPIO["bsp_gpio<br/>CS, RST, EXTI callback"]
        B_WDG["bsp_watchdog<br/>IWDG timeout 500 ms"]
        B_UART["bsp_uart<br/>UART1 wrapper"]
    end

    subgraph L4["TẦNG 4: HAL / VENDOR / HARDWARE"]
        HAL["STM32F1 HAL<br/>FreeRTOS Kernel<br/>WIZnet ioLibrary"]
    end

    NOTIFY1 --> T_RX1
    NOTIFY2 --> T_RX2

    T_RX1 --> POOL
    T_RX2 --> POOL
    T_RX1 -->|Đẩy buffer index| Q_RX
    T_RX2 -->|Đẩy buffer index| Q_RX
    Q_RX --> T_PROC

    T_PROC <-->|Đọc/ghi buffer| POOL
    T_PROC --> S_CRYPTO
    T_PROC -->|Đẩy buffer index| Q_TX
    Q_TX --> T_TX
    T_TX -->|Giải phóng buffer| POOL

    T_RX1 --> S_RELAY
    T_RX2 --> S_RELAY
    T_TX --> S_RELAY
    T_HB --> S_RELAY
    T_HB --> B_WDG
    T_DIAG --> S_LOG

    S_RELAY --> B_W5500
    B_W5500 --> B_SPI
    B_W5500 --> B_GPIO
    S_LOG --> B_UART

    B_SPI --> HAL
    B_GPIO --> HAL
    B_WDG --> HAL
    B_UART --> HAL
```

### 4. Luồng ownership của buffer

```text
FREE → RX task → RX queue → packet processing → TX queue → TX task → FREE
```

Queue chỉ chuyển `uint8_t buffer_idx`, không copy toàn bộ payload. Điều này giảm RAM, giảm tải CPU và xác định rõ task đang sở hữu buffer.

### 5. Ước tính SRAM

| Thành phần | Ước tính |
|---|---:|
| 6 task ứng dụng, stack và TCB | 6,672 B |
| Idle task và Timer task | ~1,200 B |
| Static Buffer Pool | 5,600 B |
| Queue, mutex, AES context, HAL | ~2,200 B |
| Main/ISR stack | 1,536 B |
| **Tổng** | **~17,208 B / 20 KB (~86%)** |
| **Còn lại** | **~3,272 B** |

> Toàn bộ task, queue, mutex và buffer sử dụng cấp phát tĩnh. Cần bật `configCHECK_FOR_STACK_OVERFLOW = 2` và theo dõi `uxTaskGetStackHighWaterMark()` khi kiểm thử tải cao.
## 6. Luồng dữ liệu và cấu trúc khung

```mermaid
flowchart LR
    subgraph SENDER["1. Máy gửi — Mạng A hoặc B"]
        P_IN["Plaintext<br/>1 đến 512 B<br/>Kèm CRC kiểm thử"]
    end

    subgraph STM32["2. STM32 Gateway"]
        direction TB
        PAD["1. PKCS#7 padding<br/>16 đến 528 B"]
        IV_GEN["2. Sinh IV 16 B mới<br/>cho mỗi frame"]
        AES_ENC["3. AES-128-CBC<br/>Mỗi chiều dùng khóa riêng<br/>Key_AB hoặc Key_BA"]
        PACK["4. Đóng gói frame, tối đa 546 B<br/>[len_hi][len_lo][IV 16 B][ciphertext]<br/>len = 16 + ciphertext length"]

        PAD --> AES_ENC
        IV_GEN --> AES_ENC
        AES_ENC --> PACK
    end

    subgraph RECEIVER["3. Máy nhận — Mạng B hoặc A"]
        direction TB
        UNPACK["5. TCP reassembly và tách frame<br/>Đọc length 2 B, IV và ciphertext"]
        AES_DEC["6. AES-128-CBC decrypt<br/>Gỡ PKCS#7 padding<br/>Kiểm tra CRC"]
        P_OUT["Khôi phục plaintext gốc"]

        UNPACK --> AES_DEC --> P_OUT
    end

    P_IN ==>|TCP/UDP plaintext| PAD
    PACK ==>|W5500 đích: ciphertext + IV| UNPACK
```

### PKCS#7 padding

AES-128-CBC chỉ xử lý dữ liệu có độ dài là bội số của 16 byte.

```text
P = 16 - (L mod 16)
```

Mỗi byte padding có giá trị bằng `P`.

Ví dụ, với plaintext dài 512 B:

```text
512 mod 16 = 0
P = 16

Ciphertext = 512 + 16 = 528 B
Frame = 2 B length + 16 B IV + 528 B ciphertext = 546 B
```

Dù plaintext đã chia hết cho 16, PKCS#7 vẫn phải thêm cả một block `0x10` gồm 16 byte. Điều này giúp đầu nhận luôn xác định được padding khi giải mã.

### Cấu trúc frame

| Thành phần | Kích thước | Mục đích |
|---|---:|---|
| `len_hi`, `len_lo` | 2 B | Độ dài Big-Endian của `IV + ciphertext`. |
| `IV` | 16 B | IV riêng của frame hiện tại. |
| `ciphertext` | 16–528 B | Plaintext đã được AES-128-CBC và PKCS#7 xử lý. |

Với frame lớn nhất:

```text
len = 16 + 528 = 544 = 0x0220
Bytes đầu frame: 0x02 0x20
Tổng số byte truyền: 2 + 544 = 546 B
```

Với TCP, dữ liệu nhận được có thể bị chia hoặc gộp tùy ý, nên phải tích lũy dữ liệu cho đến khi nhận đủ `2 + len` byte rồi mới giải mã. Nếu quá 5000 ms vẫn không đủ, hủy frame dở dang.

> Lưu ý bảo mật: `xorshift32` với seed từ tick chỉ phù hợp minh họa/kiểm thử, không đủ an toàn để sinh IV cho hệ thống thật vì IV có thể dự đoán hoặc lặp sau reset. Triển khai thực tế nên dùng nguồn ngẫu nhiên mật mã, hoặc tạo IV bằng một PRF/AES với khóa riêng và bộ đếm không lặp.
