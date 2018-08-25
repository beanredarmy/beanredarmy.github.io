---
layout: post
title: Tổng quan về Linux kernel SPI 
subtitle: Hỗ trợ giao thức SPI trên Linux 
gh-repo: 
gh-badge: [star, fork, follow]
tags: [kernel, embedded]
---

_Tham khảo từ document của Linux_

## 1. SPI là gì?

SPI (Serial Peripheral Interface) là một giao thức sử dụng 4 đường kết nối nối tiếp đồng bộ, dùng để giao tiếp giữa vi điều khiển tới cảm biến, bộ nhớ hoặc các ngoại vi. SPI sử dụng hình thức kết nối chủ/tớ (Master/Slave).

3 đường kết nối đầu bao gồm đường clock (SCK, thường ở 10MHz) và 2 đường dữ liệu song song (MOSI (Master Out, Slave In) và MISO (Master In, Slave Out)). Có 4 mode clock để điều khiểu trao đổi dữ liệu, trong đó mode 0 và mode 3 được sử dụng nhiều nhất. Mỗi clock cycle dịch dữ liệu vào và ra. Clock sẽ không cycle nếu không có dữ liệu bit để dịch. Và lưu ý không phải tất cả dữ liệu bit sẽ được sử dụng và cũng không phải lúc nào cũng sử dụng truyền dữ liệu song công.

Đường kết nối thứ 4 là đường "chip select" dùng để chọn thiết bị SPI Slave, do đó, 3 đường đã nhắc ở trên thì sẽ được kết nối song song tới các chip Slave. Tất cả SPI Slave đề hỗ trợ chipselect và chúng thường nhận tín hiệu tích cực thấp. Đường tín hiệu này thường có tên là nCSx với Slave 'x' (VD nCS0 với Slave 0). Một số thiết bị có những đường tín hiệu khác, thường là một đường interrupt tới Master.

Không giống như những đường serial bus trên USB hay SMBus, ngay cả những protocol ở mức low level với các SPI Slave thường không tương thích giữa những hãng khác nhau.
  - SPI có thể được sử dụng theo kiểu giao thức yêu cầu/ phản hồi, như với cảm biến màn hình cảm ứng hay là chip nhớ.
  - Có thể sử dụng để truyền dữ liệu chỉ trên 1 đường ( bán song công) hoặc là cả 2 (song công).
  - Một số thiết bị sử dụng word = 8 bit. Một số sử dụng độ dài word khác như là 12-bit hoặc 20-bit.
  - Word thường được truyền MSB trước, LSB sau nhưng thỉnh thoảng có thể là ngược lại.
  - Thỉnh thoảng SPI được sử dụng cho daisy-chain device, nó giống như thanh ghi dịch. Tham khảo ở https://www.maximintegrated.com/en/app-notes/index.mvp/id/3947

Các SPI Slave thường chỉ hỗ trợ cho một số giao thức tự động tìm/liệt kê. Một cây các thiết bị Slave có thể truy cập từ SPI Master thường sẽ được setup thủ công, với một bảng config.

Một số chip có thể loại bỏ 1 đường tín hiệu bằng cách kết hợp cả 2 đường MOSI và MISO vào một, và đương nhiên thì nó chỉ có thể truyền bán song công. Bạn có thể tìm thấy những chip được mô tả sử dụng 3 đường tín hiệu: SCK, data, nCSx. Đường data thình thoảng được gọi là MOMI hoặc SISO.

## 2. Ai sử dụng SPI? Sử dụng trên những hệ thống nào?

Các Linux developer sử dụng SPI thông thường để viết những device driver cho board nhúng. SPI được dùng để điều khiển chip ngoại vi và nó cũng là một giao thức hỗ trợ bởi tất cả các MMC hoặc thẻ nhớ SD. Một số PC còn sử dụng SPI flash cho BIOS code.

Những chip SPI Slave có thể là các bộ chuyển đổi số/tương tự được dùng cho các cảm biến tương tự, bộ giải mã, bộ nhớ, hoặc các ngoại vi như USB controller, Ethernet adapter, ...

Những hệ thống sử dụng SPI thường được tích hợp những thiết bị ngay trên board. Một số thì thiết bị được kết nối ra ngoài board bằng các connector mở rộng, trong trường hợp không có một SPI controller chuyên dụng thì các chân GPIO được dùng để giao tiếp với Slave. Một số rất ít sử dụng "hotplug" cho một SPI controller. Lí do chọn SPI tập trung vào việc giá thành rẻ và đơn giản. Nếu việc config tự động là quan trọng thì USB thường được coi là phù hợp hơn.

Nhiều vi điều khiển có thể chạy Linux tích hợp một hoặc nhiều I/O có SPI mode. Với việc hỗ trợ SPI, chúng có thể sử dụng MMC hoặc SD card mà không cần một MMC/SD/SDIO controller đặc biệt cho nó.

## 3. 4 mode clock của SPI là gì?

Việc sử dụng 2 bit là CPOL và CPHA hình thành nên 4 mode của SPI:
  - SPOL = 0 : clock bắt đầu ở mức thấp, SPOL = 1: clock bắt đầu ở mức cao
  - CPHA = 0: lấy mẫu ở sườn lên, CPHA = 1: lấy mẫu sườn xuống

![Tổ hợp các mode của SPI](/img/Clock Mode SPI.png)

![Clock Timing](/img/Clock Mode SPI Timing.png)

Chú ý rằng clock mode sẽ được áp dụng ngay khi chipselect được active. Do đó Master phải set clock về inactive trước khi chọn Slave và Slave sẽ được báo về mode được chọn khi chipslect được active. Lí do mode 0 và 3 được dùng nhiều là do chúng không quan tâm đến pha mà chỉ việc lấy mẫu ở sườn lên.

## 4. Những driver SPI hoạt động như thế nào?

Những SPI request luôn được đưa vào hàng đợi I/O, được thực thi theo thứ tự FIFO và kết thúc một cách không đồng bộ. Có một số hàm wrapper đồng bộ, bao gồm một số việc như viết một command hay đọc phản hồi.

Có 2 loại SPI driver:
  - Controller driver: Controller được built vào vi xử lý SOC và thường được hỗ trợ cả vai trò Master cũng như Slave. Những driver này liên quan đến các thanh ghi trong hardware vi xử lý và có thể sử dụng DMA. Đối với mỗi con CPU hay SOC nếu có hỗ trợ giao thức SPI thì có cần một controller(nó có thể đóng vai trò là master hoặc slave trong việc giao tiếp spi).Do đó muốn cho kernel thấy và sử dụng được, mình cần có device driver cho phần controller này
  - Protocol driver: Đóng vai trò chuyển những thông tin từ controller driver để giao tiếp với Slave hoặc Master. Khi mình kết nối các spi device với con CPU hay con SOC để sử dụng dùng SPI interface như các flash device(spi nor, spi nand hay cái sensor hỗ trợ spi) thì mình cấn viết cái protocol driver để kernel có thể hiểu các chức năng hoạt động của các device này.
    
"struct spi_device" đóng gói phần controller driver trong 2 loại driver. 

Driver model để kết nối controller và protocol driver sử dụng device table, bảng này được cung cấp bởi code init chi tiết cho board (có thể là device tree).
SPI xuât hiện trong sysfs ở một vài nơi như:
  - /sys/devices/.../CTLR ... node vật lý cho một SPI controller
  - /sys/devices/.../CTLR/spiB.C ... spi_device trên bus "B", chipselect C, truy cập thông qua CTLR
  - /sys/bus/spi/devices/spiB.C ..symlink tới spiB.C ở bên trên
  - /sys/devices/.../CTLR/spiB.C/modalias ... định danh cho driver được sử dụng với device.
  - /sys/bus/spi/drivers/D ... driver cho một hoặc nhiều spi*.* device.
  - /sys/class/spi_master/spiB ... class cho SPI master dùng bus "B". Tất cả spiB.* device chia sẻ chung một đường bus SPI vật lý, với SCLK, MOSI, MISO
  - /sys/devices/.../CTLR/slave ... virtual file cho một slave device (có thể chưa hoặc đã đăng kí). Việc viết tên của driver của một SPI Slave handler vào file này sẽ đăng kí cho slave device. Viết "(null)" sẽ hủy đăng kí cho slave device. Đọc từ file này sẽ được  tên của slave device (sẽ là "(null)" nếu chưa được đăng kí).
  - /sys/class/spi_slave/spiB ... class cho SPI slave dùng bus "B". Nếu được đăng kí, một file spiB.* device sẽ xuất hiện ở đây và chia sẻ chung đường bus SPI vật lý với các SPI slave device khác.
  

