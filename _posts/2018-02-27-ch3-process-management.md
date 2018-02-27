---
layout: post
title: Chapter 3. Process Management
subtitle: Chương 3. Quản lý tiến trình
gh-repo: daattali/beautiful-jekyll
gh-badge: [star, fork, follow]
tags: [test]
---

#### **Bài viết được dịch từ cuốn Linux Kernel Development**

Chương này sẽ giới thiệu về khái niệm của **tiến trình**. Bộ quản lý tiến trình là một yếu tố cốt lõi của bất kì hệ điều hành nào, trong đó có cả Linux.

## Tiến trình 

Tiền trình là một chương trình (object code stored on some media) .
Bên cạnh việc thi hành đoạn mã chương trình (text section in Unix), các tiến trình còn bao gồm:
- Mở các file.
- Chờ các tín hiệu.
- Dữ liệu bên trong kernel.
- Trạng thái bộ xử lý.
- Không gian địa chỉ trong bộ nhớ (with one or more memory mappings).
- **Các thread**.
- Vùng dữ liệu chứa biến toàn cục.

<span style="color:red">Các thread</span>.

Các thread là các hoạt động bên trong một tiến trình. Mỗi thread chứa:
- Bộ đếm chương trình (program counter)
- Ngăn nhớ tiến trình (process stack)
- Tập các thanh ghi bộ xử lý.

Kernel lập lịch cho các thread, chứ không phải là các tiến trình (process). Linux không phân biệt giữa thread và process. Với Linux, một thread là một loại tiến trình đặc biệt.


<span style="color:red">Bộ xử lý ảo và bộ nhớ ảo</span>.




Dễ dàng nhất vẫn là:
### Sử dụng git
Sử dụng lệnh git để clone một phiên bản của Linux kernel (cụ thể mình sẽ clone version 2.6):
~~~
$ git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux-2.6.git
~~~
Và sau đó update:
~~~
$ git pull
~~~
Với hai lệnh này thì về cơ bản bạn đã có được bản cập nhật mới của linux 2.6

## Kernel Source Tree
Kernel Source Tree được chia thành các thư mục, đa số chúng chứa nhiều các thư mục con khác. 
Các thư mục gốc trong source tree được mô tả bằng bảng dưới đây:


| Thư mục | Mô tả|
| :----| :----------------|
| arch| Kiến trúc chi tiết |
| block| Các tầng vào ra|
| crypto| Mã hóa API|
| Documentation| Tài liệu của Kernel Source|
| drivers| Device drivers|
| firmware| Device firmware cần thiết cho driver nào đó|
| fs| VFS và các file hệ thống đơn lẻ|
| include| Kernel headers|
| init| Khởi tạo Kernel|
| ipc| Mã giao tiếp liên tiến trình|
| kernel| Các hệ thống con lõi, vd như bộ lập lịch|
| lib| Helper routines|
| mm| Quản lí bộ nhớ và máy ảo|
| net| Hệ thống network|
| samples| Các sample code|
| scripts| Scripts để build kernel|
| security| Linux Security Module|
| usr| Không gian người dùng|
| tools| Các công cụ để phát triển Linux|
| virt| Virtualization infrastructure|



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
