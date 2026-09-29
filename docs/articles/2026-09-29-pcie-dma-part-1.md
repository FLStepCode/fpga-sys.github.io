---
title: Разработка 8-канального DMA для PCIe своими руками
type: article
date: 2026-09-29
author: stargazer
tags:
  - article
  - FPGA
  - PCIe
  - DMA
  - walkthrough
---

# Разработка 8-канального DMA для PCIe своими руками

> автор: *Талибов Сэрхан, Junior RTL developer, играю с FPGA в универе*

Проект реализовывает 8-канальный (в максимальной конфигурации) DMA от ПЛИС к ПК через PCIe 2.0 x4 и доступен в публичном репозитории https://github.com/apoj-inc/Kintex-7-PCIe-DMA. В нем содержится RTL-код, констрейны, tcl-скрипт для репродьюсера IP-ядра, билд-система для автоматизированной сборки и прошивки ПЛИС, софт (драйвер и пара примеров использования).

Особенности:
- IP-ядро - 7 Series FPGAs Integrated Block for PCI Express (UG477) - использует трансиверы и оборачивает их логикой таким образом, чтобы передавать и принимать PCIe TLP (Transaction Layer Packet) от юзер-логики на ПЛИС через AXI4-Stream;
- Логика, преобразующая PCIe TLP в обе стороны в AXI4 транзакции, реализована кодом на SystemVerilog (топлевел в файле rtl/src/pcie_axi_bridge/kdma_pcie_axi_bridge.sv);
- DMA-Engine, работающий на AXI4, реализован кодом на SystemVerilog (топлевел в файле rtl/src/dma_controller/kdma_top.sv);
- Модуль kdma_toplevel (файл rtl/src/kdma_toplevel.sv) является топлевелом всего проекта и соединяет DMA-Engine + Bridge и IP-ядро. В данном примере DMA-Engine реализует эходевайс, то есть при DMA-записи на ПЛИС и последующем DMA-чтении с ПЛИС прочитанные данные будут совпадать с записанными данными.

## Сборка и использование из репозитория

Оборудование, на котором проводилось тестирование:
- ПК с Ubuntu 22.04 - драйвер написан для Linux, границы совместимости я не проверял;
- Отладочная плата STLV7325 с ПЛИС Kintex 7 xc7k325t (part xc7k325tffg676-2L). В теории проект можно завести на любой ПЛИС 7-Series с соответствующей периферией, но за работоспособность не отвечаю + в любом случае придется ковырять констрейны хотя бы для коррекции локации пинов.

### Компиляция и прошивка

1. Скачивание репозитория:
    ```bash
    $ git clone https://github.com/apoj-inc/Kintex-7-PCIe-DMA

    $ cd <корневая директория репозитория>

    $ git submodule update --init --recursive --checkout
    ```
    Дальнейшие команды будут выполняться из корневой папки репозитория.

2. Компиляция/синтез проекта:
    ```bash
    $ make -f build_system/vivado/makefile TOPLEVEL=kdma_toplevel clean compile
    # После завершения работы должен появиться битстрим
    # .cache/vivado/kdma_toplevel/kdma_toplevel.runs/impl_1/kdma_toplevel.bit
    ```

3. Прошивка ПЛИС:
    ```bash
    $ make -f build_system/vivado/makefile TOPLEVEL=kdma_toplevel DEVICE_NUMBER=<N> program
    # N - это номер целевого ПЛИС в JTAG-цепочке
    ```
    Если прошить не получается, всегда можно открыть Vivado и через Hardware Manager найти нужный ПЛИС и прошить указанный битстрим

4. После успешной прошивки необходимо обновить PCIe, чтобы ПК увидел карточку, и проверить список PCIe устройств:
    ```bash
    $ echo 1 | sudo tee /sys/bus/pci/rescan

    $ sudo lspci -vvd 10ee:7024
    # -vv - very verbose (очень подробно), -d 10ee:7024 - фильтр по ID вендора и ID девайса
    # VendorID = 10ee и Device ID = 7024 - дефолтные настройки IP-ядра

    04:00.0 Memory controller: Xilinx Corporation Device 7024
            Subsystem: Xilinx Corporation Device 0007
            Control: I/O- Mem- BusMaster- SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
            Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
            IOMMU group: 1
            Region 0: Memory at 91204000 (64-bit, non-prefetchable) [disabled] [size=4K]
            Region 2: Memory at 91200000 (64-bit, non-prefetchable) [disabled] [size=16K]
            Capabilities: [40] Power Management version 3
    ...
    ```

5. Компиляция и активация драйвера:
    ```bash
    $ cd ./sw/dma_driver

    $ make

    $ sudo insmod kdma_driver.ko
    ```
    При просмотре сообщений ядра будет выведено похожее сообщение:
    ```bash
    $ sudo dmesg -w

    [  405.636751] hdlnocgen_c5p_driver: Device vid: 0x10EE
    [  405.636753] hdlnocgen_c5p_driver: Device pid: 0x7024
    [  405.636757] hdlnocgen_c5p_dma 0000:04:00.0: enabling device (0000 -> 0002)
    [  405.636763] hdlnocgen_c5p_driver: BAR[0]: 0x91204000-0x91204fff
    [  405.636764] hdlnocgen_c5p_driver: BAR[2]: 0x91200000-0x91203fff
    [  405.636778] hdlnocgen_c5p_driver: Extracting configuration info...
    [  405.636780] hdlnocgen_c5p_driver: Extracting done. This DMA has 8 channels
    [  405.636780] hdlnocgen_c5p_driver: Allocating 8 DMA and 8 user interrupts
    [  405.636924] hdlnocgen_c5p_driver: Allocated 16 interrupts using MSIXs
    [  405.636926] hdlnocgen_c5p_driver: DMA IRQ for channel 0 is 181
    [  405.636936] hdlnocgen_c5p_driver: Registered IRQ handler for DMA channel 0
    ......
    [  405.639688] hdlnocgen_c5p_driver: Created 4194304 bytes of dma bytes. Channel - 7, CPU addr - 0x0000000009a5b5bd, DMA addr - 0x1d0000000
    [  405.639689] hdlnocgen_c5p_driver: Wrote DMA addr for channel 7
    [  405.639690] hdlnocgen_c5p_driver: Register read data - DMA addr low d0000000
    [  405.639692] hdlnocgen_c5p_driver: Register read data - DMA addr high 1
    [  405.639693] hdlnocgen_c5p_driver: Channel 8 struct addr is 0x240
    [  405.639695] hdlnocgen_c5p_driver: Registered cdev with Major 510 starting with Minor 0
    [  405.639714] hdlnocgen_c5p_driver: Created class hdlnocgen_c5p_class
    [  405.639783] hdlnocgen_c5p_driver: Created device file hdlnocgen_c5p0
    [  405.639817] hdlnocgen_c5p_driver: Created device file hdlnocgen_c5p1
    [  405.639838] hdlnocgen_c5p_driver: Created device file hdlnocgen_c5p2
    [  405.639858] hdlnocgen_c5p_driver: Created device file hdlnocgen_c5p3
    [  405.639878] hdlnocgen_c5p_driver: Created device file hdlnocgen_c5p4
    [  405.639898] hdlnocgen_c5p_driver: Created device file hdlnocgen_c5p5
    [  405.639918] hdlnocgen_c5p_driver: Created device file hdlnocgen_c5p6
    [  405.639937] hdlnocgen_c5p_driver: Created device file hdlnocgen_c5p7
    [  405.639957] hdlnocgen_c5p_driver: Created device file hdlnocgen_c5p_user_irq
    [  405.640009] hdlnocgen_c5p_driver: Created device file hdlnocgen_c5p_env_csr
    [  405.640031] hdlnocgen_c5p_driver: Created device file hdlnocgen_c5p_dma_csr
    [  405.640036] hdlnocgen_c5p_driver: Bus mastered by PCIe device
    ```
    Теперь устройство готово к DMA трансферам!

6. Компиляция и запуск примеров:

    Интерактивный эходевайс:
    ```bash
    $ cd ./sw/example

    $ gcc -o echodevice_interactive echodevice_interactive.c

    $ ./echodevice_interactive 0
    # Число - это номер канала. Для 8-канального DMA возможные значения - от 0 до 7 включительно

    DMA channel 0 echodevice demonstration. Write something: sldkfjlkdfj
    You entered sldkfjlkdfj

    Resetting the DMA controller...
    DMA controller is reset
    Writing to DMA channel 0... Done
    Reading from DMA channel 0... Done
    DMA says: sldkfjlkdfj
    ```
    Данный пример считывает строку, отправляет ее в выбранный канал через DMA-запись, далее читает ее с этого же канала через DMA-чтение. При корректной работе строки должны совпадать.

    Стресс-тест:
    ```bash
    $ cd ./sw/example

    $ gcc -o echodevice_test echodevice_test.c

    $ ./echodevice_test 100 0
    # Аргумент 1 - количество итераций теста. Аргумент 2 - тип теста (0 - последовательный, 1 - параллельный).

    All channels initialized data
    DMA controller reset
    All channels read from dma
    Fail array: 0 0 0 0 0 0 0 0 
    Speed: 16.161805 Gbit/sec


    $ ./echodevice_test 100 1

    All channels initialized data
    DMA controller reset
    All channels read from dma
    Fail array: 0 0 0 0 0 0 0 0 
    Speed: 14.570905 Gbit/sec
    ```
    Данный пример запускает 8 параллельных потоков, каждый из которых множество раз генерирует транзакцию DMA-записи в ПЛИС и DMA-чтения из ПЛИСа, которые в итоге перекладывают данные из одного массива в другой. В последовательном тесте отдельно взятый поток ожидает окончания записи перед началом чтения, в параллельном тесте чтение и запись запускаются параллельно в отдельных потоках. Далее происходит проверка на расхождение данных в массиве-источнике и массиве-приемнике, а также вычисляется скорость работы на основе временных замеров, проводимых для каждой итерации теста.


!!! Warning
    Используемый пример хранит данные в очереди глубины 1024 слов с шириной слова 128 бит. Таким образом, максимальный размер транзакции, который не приведет к застреванию DMA-Engine - 16384 байт.

!!! Warning
    При DMA-чтении из ПЛИС важно, чтобы было что читать. Для эходевайса это значит, что перед DMA-чтением с ПЛИС необходимо сначала произвести DMA-запись со стольким же или большим количеством данных.

7. Деактивция драйвера и перезагрузка PCIe:
    ```bash
    $ sudo rmmod kdma_driver          # Деактивация драйвера

    $ cd /sys/devices/pci0000:00/0000:00:01.0/0000:01:00.0/0000:02:10.0/0000:04:00.0
    # Для вас это может быть другая директория в /sys/devices/pci0000:00/. Название целевой папки должно совпадать 
    # с идентификатором устройства в lspci (у меня 04:00.0). Команды tree и grep могут помочь с поиском

    $ cd ../

    $ echo 1 | sudo tee ./0000\:04\:00.0/remove
    ```
    Эти команды могут пригодиться при перепрограммировании ПЛИС, поскольку для этого необходимо безопасно отключить устройство от PCIe.

## Траблшутинг

На шаге 4 столкнулись со следующим (`Memory at <unassigned>` для обоих регионов MMIO):
```bash
$ echo 1 | sudo tee /sys/bus/pci/rescan

$ sudo lspci -vvd 10ee:7024

04:00.0 Memory controller: Xilinx Corporation Device 7024
        Subsystem: Xilinx Corporation Device 0007
        Control: I/O- Mem- BusMaster- SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        IOMMU group: 1
        Region 0: Memory at <unassigned> (64-bit, non-prefetchable) [disabled]
        Region 2: Memory at <unassigned> (64-bit, non-prefetchable) [disabled]
        Capabilities: [40] Power Management version 3
...
```

При просмотре сообщений ядра будет выведено следующее:
```bash
$ sudo dmesg -w

...
[  195.858358] pci 0000:04:00.0: [10ee:7024] type 00 class 0x058000 PCIe Endpoint
[  195.858397] pci 0000:04:00.0: BAR 0 [mem 0x00000000-0x00000fff 64bit]
[  195.858401] pci 0000:04:00.0: BAR 2 [mem 0x00000000-0x00003fff 64bit]
[  195.858465] pci 0000:04:00.0: PME# supported from D0
[  195.858536] pci 0000:04:00.0: Adding to iommu group 1
[  195.870262] pcieport 0000:02:10.0: bridge window [mem size 0x00100000]: can't assign; no space
[  195.870265] pcieport 0000:02:10.0: bridge window [mem size 0x00100000]: failed to assign
[  195.870267] pci 0000:04:00.0: BAR 2 [mem size 0x00004000 64bit]: can't assign; no space
[  195.870269] pci 0000:04:00.0: BAR 2 [mem size 0x00004000 64bit]: failed to assign
[  195.870270] pci 0000:04:00.0: BAR 0 [mem size 0x00001000 64bit]: can't assign; no space
[  195.870271] pci 0000:04:00.0: BAR 0 [mem size 0x00001000 64bit]: failed to assign
```
Откройте `/etc/default/grub` с правами sudo и добавьте `pci=realloc` в переменную `GRUB_CMDLINE_LINUX_DEFAULT` (пример - `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on iommu=pt video=efifb:off pci=realloc"`). Эта опция позволит ядру реконфигурировать адреса MMIO во время загрузки, не полагаясь на то, что назначил BIOS.

Далее обновите grub:
```bash
$ sudo update-grub
```
И перезагрузите ПК. Если в ПЛИС зашит чисто битстрим без программирования Flash-памяти, то нужно именно перезагрузить, а не выключить и включить, иначе ПЛИС потеряет питание и его надо будет заново программировать, что может привести опять к этой проблеме.

Если теперь проверить инфу о девайсе, то все адреса должны быть назначены:
```bash
$ sudo lspci -vvd 10ee:7024

04:00.0 Memory controller: Xilinx Corporation Device 7024
        Subsystem: Xilinx Corporation Device 0007
        Control: I/O- Mem- BusMaster- SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        IOMMU group: 1
        Region 0: Memory at 91204000 (64-bit, non-prefetchable) [disabled] [size=4K]
        Region 2: Memory at 91200000 (64-bit, non-prefetchable) [disabled] [size=16K]
        Capabilities: [40] Power Management version 3
...
```

## Процесс разработки

### Создание инстанса IP-ядра

Используется ядро 7 Series FPGAs Integrated Block for PCI Express (UG477), которое является оберткой для трансиверов, к площадкам которого выводятся контакты PCIe разъема. Он реализовывает карту памяти PCIe-устройства, кредиты Flow Control и прочие детали стандарта. Интеграция этого IP-ядра будет разделена на 2 части - правильное подключение тактовых сигналов и реализация декодера PCIe TLP.

#### Часть 1. Тактование IP-ядра