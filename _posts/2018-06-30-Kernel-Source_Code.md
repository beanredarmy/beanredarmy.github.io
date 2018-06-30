---
layout: post
title: Bài 2. Kernel Source Code
subtitle: Nó là cái gì vậy???
gh-repo: 
gh-badge: [star, fork, follow]
tags: [test]
---

_Bài viết tham khảo từ tài liệu của tổ chức Bootlin_
# I. Linux Code và Device Driver

## 1. Nó dùng ngôn ngữ lập trình nào?
  * Ngôn ngữ C được sử dụng để xây dựng tất cả các hệ điều hành Unix ( C được tạo nhằm mục đích viết ra hệ điều hành Unix đầu tiên).
  * Một chút ngôn ngữ Assembly để:
    * Khởi tạo CPU
    * Một số các thư viện khác
  * Không dùng một chút C++ nào
  * Tất cả code đều được compile bằng _gcc_

## 2. Không dùng một tí thư viện C nào???
  * Kernel phải độc lập và không thể sử dụng code từ user space.
  * Nguyên nhân về kiến trúc: User space được thực hiện trên đỉnh của kiến trúc kernel, không phải bên dưới.
  * Nguyên nhân kỹ thuật: Kernel bắt đầu chạy từ quá trình boot, trước khi nó truy cập được vào root filesystem (nơi chứa thư viện C).
  * Do đó, kernel code phải tự có cho riêng nó một thư viện khác (dùng để xử lý string, mã hóa, ...)
  * Và vì thế, không thể sử dụng những hàm trong thư viện C chuẩn như là _printf(), memset(), malloc(),.._
  * May nay kernel cung cấp những hàm tương tự như là _printk(), memset(), kmalloc(),.._

## 2. Những tính năng chủ chốt của Linux kernel
  * Sự linh động và hỗ trợ hardware mạnh mẽ. Có thể chạy trên hầu hết các kiến trúc.
  * Khả năng mở rộng. Có thể chạy trên các siêu máy tính cũng như các thiết bị tí hon (4Mb RAM là đủ).
  * Tuân thủ các chuẩn cũng như tính tương thích.
  * Bảo mật. Code của linux được review bởi rất nhiều chuyên gia, do đó có thể rà soát được hầu hết những lỗi.
  * Ổn định và tin cậy.
  * Tính module hóa. Có thể chỉ include những thứ mà hệ thống cần ở thời điểm chạy.
  * Dễ  dàng lập trình. CÓ thể học từ những source code có sẵn. 

## 3. Tính di động (Portability)
  * Linux kernel code được thiết kế để có thể portable
  * Tất cả những code nằm ngoài _arch/_ đều có thể portable
  * Với yêu cầu trên, kernel cung cấp những marco và hàm để trừu tượng hóa những kiến trúc cụ thể:
    * Endianness:
      * cpu_to_be32()
      * cpu_to_le32()
      * be32_to_cpu()
      * le32_to_cpu()
    * Truy cập bộ nhớ vào ra
    * Bảo vệ bộ nhớ nếu cần
    * DMA (Direct memory access) API để flush hoặc vô hiệu hóa các cache nếu cần.

## 4.  Không có tính toán dấu phẩy động
  * Không bao giờ sử dụng số dấu phẩy động trong kernel code. Code của bạn có thể phải chạy trên một vi xử lý mà ko có đơn vị dấu phẩy động ( ví dụ trên một số CPU ARM).
  * Đừng bối rối với những config option liên quan đến dấu phẩy động:
    * Chúng liên quan đến việc bắt chước phép tính dấu phẩy động trên ứng dụng ở user space .
    * Hãy sử dụng soft-float.
  
## 5. Không có API nào hoàn toàn ổn định trong Linux
  * Những API để thi hành kernel code có thể bị thay đổi giữa 2 bản phát hành.
  * Những In-tree driver được cập nhật bởi nhà phát triển để thay đổi được API cho bản kernel chính.
  * Những out-tree driver được compile cho một số các version có thể sẽ không còn hoạt động về sau nữa.
  * Xem [process/stable-api-nonsense](https://www.kernel.org/doc/html/latest/process/stable-api-nonsense.html) trong kernel source để hiểu tại sao.
  * Dĩ nhiên, những API từ kernel đến user space không bị thay đổi (những lời gọi hệ thống, /proc, /sys).

## 6. Ràng buộc bộ nhớ kernel
  * Kernel không có sự bảo vệ bộ nhớ!
  * Kernel không cố gắng để khôi phục để truy cập vào vùng nhớ bất hợp pháp. Việc này sẽ taọ ra một oops message trên console.
  * Dung lượng stack cố định (8 hoặc 4Kb). Không giống như user space, không có cơ chế nào có thể làm tăng dung lượng này được.
  * Swap bộ nhớ cũng không có trong kernel.

## 7. Ràng buộc giấy phép của Linux Kernel
  * Linux kernel được cấp giấy phép GNU General Public License version 2.
    * Giấy phép này cho phép người dùng có quyền để sử dụng, học tập, thay đổi và chia sẻ phần mềm một cách tự do.
  * Tuy nhiên, khi phần mềm được tái phân phối, dù là đã chỉnh sửa hay chưa thì GPL yêu cầu phần mềm phải có cùng giấy phép với source code.

## 8. Lợi thế của GPL drivers
  * Bạn sẽ không phải tự viết lấy driver của bạn mà có thể sử dụng lại code từ những driver tương tự.
  * Bạn có thể nhận được hỗ trỡ từ cộng đồng, code review và test, mặc dù điều này chỉ thường xảy ra với những code được summit cho mainline kernel.
  * Những driver được biên dịch trước chỉ hoạt động trên một phiên bản kernel và một config cụ thể, điều này làm khó user trong việc thay đổi version của kernel.

## 9. Lợi thế của in-tree kernel driver
  Một khi source của bạn được chấp nhận vào mainline tree
  * Có rất nhiều người có sẽ review code của bạn, fix code cũng như cải thiện code miễn phí.
  * Bạn cũng có thể nhận được thay đổi từ những người chỉnh sửa API kernel.
  * Truy cập code của bạn rất là dễ dàng.
  * Bạn có thể lấy được bản phân phối từ chính khách hàng của bạn.

  Điều này làm làm giảm chi phí bảo trì code.

## 10. Device Driver ở user space.
  * Sẽ là khả thi nếu muôn implement device driver ở user space.
  * Điều này xảy ra khi:
    * Kernel cung cấp cơ chế cho phép ứng dụng ở user space truy cập trực tiếp vào hardware.
    * Không cần thiế để kernel hoạt động như một multiplexer cho device: chỉ 1 ứng dụng truy cập device.
  * Những user space device driver:
      * USB với [libusb](http://www.libusb.info/)
      * SPI với [spidev](https://kernel.org/doc/Documentation/spi/spidev)
      * I2C với [i2cdev](https://kernel.org/doc/Documentation/i2c/dev-interface)
      * Memory-mapped device với [UIO](https://www.kernel.org/doc/html/latest/driver-api/uio-howto.html), có bao gồm xử lý ngắt.
  * Một số lớp các device (máy in, máy scan, ...) thường có một phần ở kernel space, một phần ở user space.
  * Ưu điểm:
    * Không cần biết code ở lớp kernek. Dễ dàng sử dụng lại code giữa những device.
    * Drivers có thể viết bằng bất kì ngôn ngữ nào, kể cả Perl.
    * Driver code có thể được debugged mà ko crash kernel.
    * Có thể tính toán dấu phẩy động.
    * Ít phức tạp hơn code ở kernel
    * Có thể có hiệu năng cao hơn, đặc biệt ở vụ memory-mapped device, vì tránh ko sử dụng các lời gọi hệ thống.
  * Nhược điểm:
    * Ít đơn giản hơn để xử lý các ngắt.
    * Tăng độ trễ ngắt so với kernel code.

You can write regular [markdown](http://markdowntutorial.com/) here and Jekyll will automatically convert it to a nice webpage.  I strongly encourage you to [take 5 minutes to learn how to write in markdown](http://markdowntutorial.com/) - it'll teach you how to transform regular text into bold/italics/headings/tables/etc.

**Here is some bold text**

## Here is a secondary heading

Here's a useless table:

| Number | Next number | Previous number |
| :------ |:--- | :--- |
| Five | Six | Four |
| Ten | Eleven | Nine |
| Seven | Eight | Six |
| Two | Three | One |


How about a yummy crepe?

![Crepe](http://s3-media3.fl.yelpcdn.com/bphoto/cQ1Yoa75m2yUFFbY2xwuqw/348s.jpg)

Here's a code chunk:

~~~
var foo = function(x) {
  return(x + 5);
}
foo(3)
~~~

And here is the same code with syntax highlighting:

```javascript
var foo = function(x) {
  return(x + 5);
}
foo(3)
```

And here is the same code yet again but with line numbers:

{% highlight javascript linenos %}
var foo = function(x) {
  return(x + 5);
}
foo(3)
{% endhighlight %}

## Boxes
You can add notification, warning and error boxes like this:

### Notification

{: .box-note}
**Note:** This is a notification box.

### Warning

{: .box-warning}
**Warning:** This is a warning box.

### Error

{: .box-error}
**Error:** This is an error box.
