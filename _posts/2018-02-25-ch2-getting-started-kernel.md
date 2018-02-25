---
layout: post
title: Getting started with Kernel
subtitle: Xin chào Kernel!
gh-repo: daattali/beautiful-jekyll
gh-badge: [star, fork, follow]
tags: [test]
---

#### **Bài viết được dịch từ cuốn Linux Kernel Development**

Trong chương này, chúng ta sẽ giới thiệu một số thứ cơ bản về  Linux Kernel (nhân Linux): cách lấy source, cách biên dịch và cài đặt một kernel mới. Sau đó ta sẽ lướt qua về sự khác nhau giữa kernel và chương trình user-space (không gian người sử dụng) và một số chương trình trong kernel. Mặc dù chỉ có duy nhất 1 kernel, vẫn có những sự khác biệt nhỏ giữa các kernel trong những project khác nhau. 

## Lấy Kernel Source 
Phiên bản Linux source code hiện tại thì luôn để ở 2 dạng là: dạn nén (.tar) và dạng patch đặt ở trang chủ của Linux kernel: [http://www.kernel.org](http://www.kernel.org).
Dưới đây giới thiệu 3 cách để lấy source code:

### Sử dụng git
Sử dụng lệnh git để clone một phiên bản của Linux kernel (cụ thể mình sẽ clone version 2.6):
~~~
$ git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux-2.6.git
~~~
Và sau đó update:
~~~
$ git pull
~~~



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
