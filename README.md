graph LR
    subgraph PC ["Máy tính Windows 11 (Kiểm thử & Giám sát)"]
        NIC_A["Card mạng #1 (Mạng A)<br/>IP: 192.168.10.10/24<br/>(Python Sender / Receiver)"]
        NIC_B["Card mạng #2 (Mạng B)<br/>IP: 192.168.20.10/24<br/>(Python Receiver / Sender)"]
        USB_DBG["Cổng USB<br/>(ST-Link V2 & USB-to-TTL)"]
    end

    subgraph HW_NET_A ["Phía Mạng A"]
        W5500_A["Module W5500 #1<br/>IP: 192.168.10.2/24<br/>3 Sockets (TCP Main, TCP Sub, UDP)"]
    end

    subgraph MCU ["STM32F103C8T6 (Blue Pill - 72 MHz)"]
        SPI1_BUS["SPI1 + DMA1 (Ch2/Ch3)<br/>CS: PA4 | RST: PA3 | INT: PA2"]
        CORE["FreeRTOS Kernel + Tasks<br/>AES-128-CBC + Static Buffer Pool<br/>IWDG (500 ms)"]
        SPI2_BUS["SPI2 + DMA1 (Ch4/Ch5)<br/>CS: PB12 | RST: PB11 | INT: PB10"]
        SWD_UART["SWD (PA13, PA14)<br/>USART1 (PA9, PA10)"]
    end

    subgraph HW_NET_B ["Phía Mạng B"]
        W5500_B["Module W5500 #2<br/>IP: 192.168.20.2/24<br/>3 Sockets (TCP Main, TCP Sub, UDP)"]
    end

    PWR["Module Nguồn rời 3.3V<br/>(Chung GND toàn hệ thống)"]

    NIC_A <==>|"Cáp RJ45 #1"| W5500_A
    W5500_A <==>|"Bus SPI1"| SPI1_BUS
    SPI1_BUS <==> CORE
    CORE <==> SPI2_BUS
    SPI2_BUS <==>|"Bus SPI2"| W5500_B
    W5500_B <==>|"Cáp RJ45 #2"| NIC_B

    USB_DBG -.->|"Nạp code & Log UART1"| SWD_UART
    PWR ===>|"Cấp 3.3V & GND"| W5500_A
    PWR ===>|"Cấp 3.3V & GND"| W5500_B
    PWR -.->|"Chung GND"| MCU
