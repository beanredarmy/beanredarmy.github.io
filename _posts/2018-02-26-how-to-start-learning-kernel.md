---
layout: post
title: How To Start Learning Kernel.
subtitle: Bắt đầu với Linux Kernel như thế nào
gh-repo: daattali/beautiful-jekyll
gh-badge: [star, fork, follow]
tags: [test]
---

#### **Bài viết được dịch từ trả lời của manav m-n trên stackoverflow**

{: .box-note}
**Note:** . "Con người ko thiểu hiểu được một cuốn sách cho đến khi họ dành một phần của cuộc đời họ cho nó, dù gì đi nữa, chẳng có ai hiểu được sâu một cuốn sách
đến khi a ta đọc và "sống trong những phần nhỏ nhất của nó. - Ezra Pound.

Một hành trình của hàng ngàn dặm dòng code luôn phải khởi đầu từ một bước đi. Nếu bạn đang bối rối về việc không biết chọn cuốn sách nào để bắt đầu học thì đừng lo, hãy chọn lấy một cuốn. Không phải cứ đi lang thang có nghĩa là đang lạc lối. Như những con suối cuối cùng cũng đổ về sông cả, bạn sẽ luôn khám phá ra những điều mới trong hành trình học kernel mà chẳng có hồi kết, và rồi kết nối chúng thành code-set. Hãy đọc và luôn nhớ răng: Code đéo phải là văn.:)))


## Lấy Kernel Source 
Phiên bản Linux source code hiện tại thì luôn để ở 2 dạng là: dạng nén (.tar) và dạng patch đặt ở trang chủ của Linux kernel: [http://www.kernel.org](http://www.kernel.org).
Có 3 cách để lấy source code là:
- Sử dụng git.
- Tải về phải nén vài giải nén.
- Sử cụng các patch.


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
