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
  * Khả năng mở rộng. Có thể chạy trên các siêu máy tính cũng như các thiết bị tí hơn (4Mb RAM là đủ).
  * Tuân thủ các chuẩn cũng như tính tương thích,
  * Bảo mật. Code của linux được review bởi rất nhiều chuyên gia, do đó có thể rà soát được hầu hết những lỗi.
  * Ổn định và tin cậy.
  * Tính module hóa. Có thể chỉ include những thứ mà hệ thống cần ở thời điểm chạy.
  * Dễ  dàng lập trình. CÓ thể học từ những source code có sẵn. 
## 3. Vai trò của Linux kernel trong hệ điều hành
  ![Linux kernel trong hệ thống](/img/Linux kernel in the system.png)

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
