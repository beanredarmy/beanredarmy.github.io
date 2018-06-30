---
layout: post
title: Giới thiệu Linux Kernel
subtitle: Cùng tìm hiểu về Linux Kernel
gh-repo: 
gh-badge: [star, fork, follow]
tags: [test]
---

_Bài viết tham khảo từ tài liệu của tổ chức Bootlin_

## 1. Một chút lịch sử
  * Linux kernel (Nhân Linux) là một thành phần của hệ hiều hành, cùng với những thư viện và ứng dụng cung cấp các tính năng cho user.
  * Linux kernel được tạo ra năm 1991 bởi một học sinh, **Linus Torvalds**, như một thú vui của ông.
  * Linus Torvalds đã tạo ra một cộng động rộng lớn cũng như mạnh mẽ các developer và user xung quanh Linux.
  * Ngày nay, hơn hàng ngàn người đóng góp cho mỗi bản release của Linux kernel, bao gồm các cá nhân, công ty, tổ chức lớn nhỏ.

## 2. Những tính năng chủ chốt của Linux kernel
  * Sự linh động và hỗ trợ hardware mạnh mẽ. Có thể chạy trên hầu hết các kiến trúc.
  * Khả năng mở rộng. Có thể chạy trên các siêu máy tính cũng như các thiết bị tí hon (4Mb RAM là đủ).
  * Tuân thủ các chuẩn cũng như tính tương thích.
  * Bảo mật. Code của linux được review bởi rất nhiều chuyên gia, do đó có thể rà soát được hầu hết những lỗi.
  * Ổn định và tin cậy.
  * Tính module hóa. Có thể chỉ include những thứ mà hệ thống cần ở thời điểm chạy.
  * Dễ  dàng lập trình. CÓ thể học từ những source code có sẵn. 

## 3. Vai trò của Linux kernel trong hệ điều hành
  ![Linux kernel trong hệ thống](/img/Linux kernel in the system.png)

  * Quản lý tất cả các tài nguyên: CPU, memory, I/O
  * Cung cấp một tập các APIs mà không phụ thuộc vào kiến trúc cũng như phần cứng, để cho phép ứng dụng user và các libraries sử dụng các tài nguyên.
  * Xử lý các tương tranh của tài nguyên từ các ứng dụng khác nhau:
    * VD: Một interface mạng được sử dụng bởi nhiều ứng dụng user bởi nhiều connection. Trách nhiệm của kernel là giải quyết vấn đề phân phối tài nguyên cho các ứng dụng đó.

## 4. Lời gọi hệ thống (System calls)
  * Interface giữa kernel và user space là một tập các lời gọi hệ thống.
  * Khoảng 400 các lời gọi hệ thống được cung cấp cho các dịch vụ trên kernel:
    * VD: File và device operations, networking, giao tiếp bên trong process, quản lý process, memory mapping, timers, threads, đồng bộ hóa, etc.
  * Interface này luôn rất ổn định: những lời gọi hệ thống mới chỉ được thêm vào bởi kernel developers.
  * Những lời gọi hệ thống này được "bọc" trong thư viện C, user thường không bao giờ gọi thẳng vào lời gọi hệ thống mà phải gọi qua các hàm trong thư viện C tương ứng.

## 5. Hệ thống pseudo filesystems
  * Linux làm cho hệ thống và thông tin kernel hiện hữu đối với user thông qua pseudo filesystems, thỉnh thoảng còn được gọi là hệ thống file ảo.
  * Pseudo filesystems cho phép các ứng dụng có thể nhìn thấy các đường dẫn, file mà không thực sự tồn tại: chúng được tạo và cập nhật nhanh chóng bởi kernel.
  * Có 2 pseudo filesystems quan trọng nhất:
    * _proc_, thường được mount ở _/proc_
    Các thông tin liên quan đến hệ điều hành (tiến trình, bộ nhớ,...)
    * _sysfs_ , thường được mount ở _/sys_
    Một tập các device và bus đại diện cho hệ thống và những thông tin về những device đó.

## 6. Bên trong linux kernel là gì
  Hình sau sẽ mô tả bên trong Linux kernel:
   ![Inside the Linux kernel](/img/Inside the Linux kernel.png)

## 7. Hỗ trợ các kiến trúc hardware
  * Thư mục _arch/_ trong kernel source:
  * Tối thiểu là processor 32 bit, có hoặc không có MMU, và hỗ trợ gcc.
  * Với kiến trúc 32 bit có arm, arc, c6x, m68k, microblaze.
  * Với kiến trúc 64 bit có alpha, arm64, ia64, tile
  * Với kiến trúc 32/64 bit có mips, powerpc, sh, sparc, x86
  * Có thể tìm thông tin chi tiết về kernel source ở: arch/<arch>/Kconfig, arch/<arch>/README, hoặc Documentation/<arch>/

## 8. Lấy kernel source ở đâu?
  * Phiên bản chính thức của Linux kernel được phát hành bởi chính Linus Torvalds, trên trang web: [http://www.kernel.org](http://www.kernel.org)
  * Một số nhà phát triển chip cung cấp kernel cho chip của họ, nhằm mục đích:
    * Tập trung vào hỗ trợ phần cứng
    * Hữu dụng khi mainline kernel (kernel chính thức) chưa hỗ trợ
  * Một số cộng đồng con của kernel tự duy trì kernel của họ, thông thường sẽ cập nhật hơn tuy nhiên sẽ ít ổn định hơn.

## 9. Cách lấy Linux source
  * Kernel source hiện có ở [https://mirrors.edge.kernel.org/pub/linux/kernel/](https://mirrors.edge.kernel.org/pub/linux/kernel/)
  * Tuy nhiên thì nhiều người thích sử dụng _git_ version control. Điều này là rất cần thiết cho việc phát triển kernel.
    * Có thể lấy toàn bộ source kernel:
      ~~~
      git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
      ~~~
    * Tạo một branch tại một phiên bản kernel cụ thể:
      ~~~
      git checkout -b <name-of-branch> v3.11
      ~~~
    * Thông qua giao diện web: [http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/.](http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/.)
    * Tìm hiểu thêm về Git tại [http://git-scm.com/](http://git-scm.com/)



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
