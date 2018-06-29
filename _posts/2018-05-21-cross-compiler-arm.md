---
layout: post
title: Cross-compiler cho ARM (PCduino3)
subtitle: Cách biên dịch chương trình cho kiến trúc ARM

---

Một bộ cross compiler là một bộ compiler có khả năng biên dịch source code từ một nền tảng sang một nền tảng khác để chạy chúng. Ví dụ ở đây source code sẽ được biên dịch trên máy tính có vi xử lý Intel, nhờ có cross compiler, mã nguồn được biên dịch có thể chạy được trên nền tảng khác có kiến trúc ARM. Một lợi thế lớn của cross compiler là tốc độ viết ứng dụng và biên dịch ứng dụng sẽ được cải thiện rõ rệt do tốc độ của máy tính là cao hơn nhiều so với hầu hết các nền tảng nhúng.



**Here is some bold text**

### Cài đặt GCC cho cross compiler

~~~
sudo apt-get install gcc-arm-linux-gnueabihf
~~~

### Cài đặt G++ cho cross compiler

~~~
sudo apt-get install g++-arm-linux-gnueabihf
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
