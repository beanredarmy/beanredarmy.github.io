---
layout: post
title: (C/C++) Hành trình từ code C đến file thực thi 
subtitle: 4 Giai đoạn trong quá trình compile mã nguồn C trên Linux 
gh-repo: 
gh-badge: [star, fork, follow]
tags: [C/C++]
---

_Tham khảo từ geekstuff_

Bạn viết một chương trình C, rồi dùng gcc để biên dịch, sau đó nhận được file thực thi. Trông thật đơn giản phải không nào? 
_(Chú ý trong bài viết, file thực thi, file executable hay file nhị phân để chỉ file đầu ra sau khi biên dịch có thể chạy trực tiếp)_

Có bao giờ bạn thắc mắc, chuyện gì xảy ra trong quá trình trình biên dịch và làm cách nào để một chương trình C có thể biến thành một file có thể chạy được?

Có 4 bước chính trong suốt quá trình mà một file source code phải trải qua để biến thành file executable, đó là:
  1. Tiền xử lý (Pre-processing)
  2. Biên dịch (Compilation)
  3. Assembly
  4. Liên kết (Linking)

Trước tiên, hãy thực hiện nhanh lại việc biên dịch một source code C bằng gcc như thông thường

```c
$ vi print.c
#include <stdio.h>
#define STRING "Hello World"
int main(void)
{
/* Using a macro to print 'Hello World'*/
printf(STRING);
return 0;
}
```
Giờ biên dịch với gcc compiler để taọ file thực thi
```
$ gcc -Wall print.c -o print
```
Trong đó thì:
  - gcc - gọi đến GNU C compiler
  - -Wall - cờ của gcc, cho phép bật tất cả các cảnh báo lên. -W viết tắt của warning, và ta sẽ warning all.
  - print.c - Source code C đầu vào
  - -o print - chỉ dẫn cho C compiler tạo ra file thực thi có tên là print. Nếu không dùng option -o thì mặc định C compiler sẽ tạo ra file thực thi là a.out

Cuối cùng file thực thi của chúng ta sẽ hiện thị ra 
```
$ ./print
Hello World
```
Chú ý. Khi làm việc với project lớn thì nên sử dùng Makefile để quản lý việc biên dịch mã nguồn C. Mình sẽ giới thiệu Makefile đến các bạn sau.

Bây giờ khi chúng ta đã có hình dung việc gcc được dùng để chuyển source code sang file nhị phân, hãy bắt đầu nghiên cứu từng bước củ thể nào.

## 1. Tiền xử lý (Pre-processing)
Đây là bước đầu tiên mà source code phải trải qua. Ở bước này, trình biên dịch phải làm những công việc:
  - Thay thế các marco
  - Cắt bỏ comment code
  - Mở rộng các file được include.

Để hiểu hơn về tiền xử lý, ta có thể sử dụng flag -E để in đầu ra của tiền xử lý:
```
$ gcc -Wall -E print.c
```
Đầu ra sẽ là một mớ code dài loằng ngoằng mà chắc hẳn ít người có thể hiểu liền được.
Để dễ hình dung hơn, bạn hãy sử dụng option _-save-temp_, option này chỉ dẫn compiler lưu lại những output tạm thời ở mỗi bước trong 4 bước đã nêu.
```
$ gcc -Wall -save-temps print.c -o print
```
```
$ ls
print.i
print.s
print.o
```
Đầu ra của bước tiền xử lý preprocessing được lưu lại ở file đuôi .i ( trong ví dụ là print.i). Bây giờ thử mở file print.i xem có gì.
```c
$ vi print.i
......
......
......
......
# 846 "/usr/include/stdio.h" 3 4
extern FILE *popen (__const char *__command, __const char *__modes) ;
extern int pclose (FILE *__stream);
extern char *ctermid (char *__s) __attribute__ ((__nothrow__));

# 886 "/usr/include/stdio.h" 3 4
extern void flockfile (FILE *__stream) __attribute__ ((__nothrow__));
extern int ftrylockfile (FILE *__stream) __attribute__ ((__nothrow__)) ;
extern void funlockfile (FILE *__stream) __attribute__ ((__nothrow__));

# 916 "/usr/include/stdio.h" 3 4
# 2 "print.c" 2

int main(void)
{
printf("Hello World");
return 0;
}
```
Wow, đó là output mà chúng ta đã thấy khi dùng option -E lúc nãy đúng không. Nhìn vào đây, ta có thể thấy được rất nhiều thông tin và ở cuối file ta được thấy những dòng code ta đã viết trước đó. Hãy thử phân tích những dòng code này một chút.
  1. Có thể nhìn ra ngay là tham số của hàm printf() giờ không phải là marco STRING nữa mà đã là chuỗi "Hello World" rồi. Như vậy có thể thấy ở bước này thì tất cả các marco sẽ được thay thế.
  2. Thứ hai có thể thấy comment của chúng ta đã biến mất. Như vậy bước này sẽ cắt đi hết những comment, thứ không có ý nghĩa đối với file nhị phân.
  3. Thứ 3 là dòng #include của chúng ta cũng biến mất, thay vào đó thì là rất nhiều dòng code khác. Như vậy có thể hiểu rằng thư viện stdio.h đã được mở rộng include trong source file của chúng ta. Trong đống code mở rộng kia, ta có thể tìm thấy một khai báo của hàm printf()
  ~~~c
  extern int printf (__const char *__restrict __format, ...);
  ~~~
  Từ khóa extern cho biết rẳng printf() không được định nghĩa ở đây mà nằm ngoài file source code ta viết. Ta sẽ bàn luận cách gcc lấy được hàm prinf() như thế nào sau.
  
  Giờ thì hãy chuyển sang bước tiếp theo.

## 2. Biên dịch (Compiling)
  Sau khi compiler hoàn thành tiền xử lý. Bước tiếp theo là lấy file print.i làm đầu vào và tiếp tục xử lý nó. Đầu ra tiếp theo của chúng ta sẽ là print.s. Đầu ra này là một file lệnh assembly.
  ```as
  $ vi print.s
.file "print.c"
.section .rodata
.LC0:
.string "Hello World"
.text
.globl main
.type main, @function
main:
.LFB0:
.cfi_startproc
pushq %rbp
.cfi_def_cfa_offset 16
movq %rsp, %rbp
.cfi_offset 6, -16
.cfi_def_cfa_register 6
movl $.LC0, %eax
movq %rax, %rdi
movl $0, %eax
call printf
movl $0, %eax
leave
ret
.cfi_endproc
.LFE0:
.size main, .-main
.ident "GCC: (Ubuntu 4.4.3-4ubuntu5) 4.4.3"
.section .note.GNU-stack,"",@progbits
```
Mã nguồn assembly này sẽ là những lệnh mà assembler có thể hiểu và chuyển được chúng thành mã máy.

## 3. Assembly
Ở bước này, file print.s lại là đầu vào và file print.o là đầu ra.
Bước này được xử lý bở assembler. Assembler có thể hiểu và chuyển một file '.s' với lệnh assembly thành file '.o' chứa mã máy. Và chắc là trừ máy tính ra thì không có ai đọc được mã máy.
~~~
$ vi print.o
^?ELF^B^A^A^@^@^@^@^@^@^@^@^@^A^@>^@^A^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@0^
^@UH<89>å¸^@^@^@^@H<89>Ç¸Hello World^@^@GCC: (Ubuntu 4.4.3-4ubuntu5) 4.4.3^@^
T^@^@^@^@^@^@^@^AzR^@^Ax^P^A^[^L^G^H<90>^A^@^@^\^@^@]^@^@^@^@A^N^PC<86>^B^M^F
^@^@^@^@^@^@^@^@.symtab^@.strtab^@.shstrtab^@.rela.text^@.data^@.bss^@.rodata
^@.comment^@.note.GNU-stack^@.rela.eh_frame^@^@^@^@^@^@^@^@^@^@^@^
...
...
…
~~~
Nhưng đếu để ý ta có thể thấy được chuỗi ELF, chuỗi này và viết tắt của executable & linking format. 

## 4. Liên kết (Linking)
Đây là bước cuối cùng mà tất cả những liên kết tới hàm được gọi và định nghĩa hàm được xử lý. Như đã nói, trước khi tới bước này, gcc khong biết được định nghĩa của hàm printf(). Ở bước này, định nghĩa của hàm printf() được biết và địa chỉ của hàm printf() cũng vậy. 



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
