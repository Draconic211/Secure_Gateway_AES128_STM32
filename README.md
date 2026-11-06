# Sơ đồ khối phần cứng

```mermaid
graph LR
    subgraph PC["Máy tính Windows 11 (Kiểm thử & Giám sát)"]
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
