---
layout: post
title: Kernel Facilities and Helper Functions (Part 2)  
subtitle:  Những công cụ và hàm hay dùng trong kernel (Phần 2)
gh-repo: 
gh-badge: [star, fork, follow]
tags: [kernel, embedded, device driver]
---

_Tham khảo từ Linux Device Drivers Development_

Ở phần 1 ta đã tìm hiểu về 2 thứ hữu ích được sử dụng rất nhiều trong kernel đó là marco ```container_of``` và cấu trúc dữ liệu ```list_head```. Ở phần này chúng ta sẽ tìm hiểu tiếp 2 cơ chế cũng rất quan trọng đó là cơ chể ngủ và cơ chế timer trong Linux. Let's get start!!!

# 1. Cơ chế ngủ của kernel
Ngủ là gì? Ai ngủ? 
Ngủ là cơ chế mà theo đó, process sẽ tha cho bộ xử lý, để bộ xử lý dành thời gian xử lý process khác. Lý do mà một tiến trình hay process cần ngủ là để xem dữ liệu đã available hay chưa, hoặc đợi tài nguyên được giải phóng. 

Bộ lập lịch của kernel quản lí một danh sách những tác vụ đang chạy, gọi là một run queue. Các process cần được ngủ sẽ bị xóa khỏi run queue và không được lập lịch nữa. Trừ khi có một thứ gì đấy tác động, đánh thức chúng dậy, nếu không thì chúng cứ ngủ vậy mãi. Chúng ta có thể cho một process đi ngủ để đợi tài nguyên giải phóng, rồi tạo ra một điều kiện gì đó để đánh thức nó dậy. Để implement cơ chế này thì kernel cung cấp một tập các function cũng như data structure.

## 1.2. Wait queue.
Khi bị loại bỏ khỏi run queue, một process ngủ sẽ đi vào wait queue. Wait queue làm một thành phần không thể thiếu trong các process blocked I/O. Để hiểu cách hoạt động của wait queu, xem source code thôi:
```c
struct __wait_queue {
  unsigned int flag;
  #define WQ_FLAG_EXCLUSIVE 0x01
  void *private;
  wait_queue_func_t func;
  struct list_head task_list;
}
```

Wow, ta lại thấy một thứ khá quen thuộc, là ```list_head```. Đó là một linked list. Tất cả những process mà cần đi ngủ thì đêu được đưa vào hàng đợi này (do đó nó có tên là wait queue) và được đặt vào trạng thái ngủ cho đến khi một điều kiện trở thành true và đánh thức nó.

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/gbmo0f4km6_LNXqH.gif)
Một Wait queue sẽ gồm các thành phần chính: Một phần tử chỉ có nhiệm vụ đứng đầu danh sách và đại diện cho cả danh sách là wait_queue_head_t ,và các phần tử đứng sau đại diện cho các task được đưa vào danh sách là wait_queue_t. Như vậy việc khai báo và vận hành một wait queue sẽ bao gồm việc tạo phần tử đầu danh sách và thêm các phần tử chứa task vào danh sách.  

Khai báo một struct, như thường lệ, ta có cả 2 cách khai báo là static và dynamic:
- Khai báo static: 

```c
DECLARE_WAIT_QUEUE_HEAD(name)
```

- Khai báo dynamic: 

```c
wait_queue_head_t my_wait_queue;
init_waitqueue_head(&my_wait_queue);
```

- Đưa process đi ngủ:

```c
/*
* block the current task (process) in the wait queue if
* CONDITION is false
*/
int wait_event_interruptible(wait_queue_head_t q, CONDITION);
```

- Đánh thức process:

```c
/*
* wake up one process sleeping in the wait queue if
* CONDITION above has become true
*/
void wake_up_interruptible(wait_queue_head_t *q);
```

Hàm ```wait_even_interruptible``` không hỏi dò mà đơn giản chỉ đánh giá điều kiện, nếu điều kiện sai thì process sẽ được chuyển thành trạng thái ```TASK_INTERRUPTIBLE``` và xóa khỏi run queue. Khi gọi hàm ```wake_up_interruptible``` thì điều kiện này lại được check lại, nếu đúng thì process đang ở trong wait queue sẽ được đánh thức, chuyển thành trạng thái ```TASK_RUNNING```. Để đánh thức tất cả các process trong queue thì dùng hàm ```wake_up_interruptible_all```;

_Note: Thực ra những hàm chính thức là ```wait_event```, ```wake_up``` và ```wake_up_all```, tuy nhiên những hàm hày lại chỉ được sử dụng với những process mà không thể bị ngắt bởi một signal. Nên chúng chỉ nên được sử dụng trong các critical task (những task không cho phép ngắt). Do những process trên không bị ngắt bởi signal, ta nên kiểm tra giá trị trả về của các hàm. Giá trị trả vể khác 0 nghĩa là việc ngủ của process đã bị ngắt bởi một signal nào đó, và driver nên trả về lỗi ```ERESTARTSYS```_ 

Nếu ta gọi ```wake_up``` hoặc ```wake_up_interruptible``` mà điều kiện vẫn ```FALSE```, thì chẳng có gì xảy ra cả. Nếu không dùng 2 hàm này thì process cũng chẳng bao giờ được thức.

Xem một ví dụ:
```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/sched.h>
#include <linux/time.h>
#include <linux/delay.h>
#include <linux/workqueue.h>

static DECLARE_WAIT_QUEUE_HEAD(my_wq);
static int condition = 0;

/*declare a work queue cái này sẽ được bàn đến sau ở phần khác */
static struct work_struct wrk;

static void work_handler(struct work_struct *work)
{
  printk("Waitqueue module handler %s\n", __FUNCTION__);
  msleep(5000);
  printk("Wake up the sleeping module\n");
  condition = 1;
  wake_up_interruptible(&my_wq);
}

static int __init my_init(void)
{
  printk("Wait queue example\n");
  INIT_WORK(&wrk, work_handler);
  schedule_work(&wrk);

  printk("Going to sleep %s\n", __FUNCTION__);
  wait_event_interruptible(my_wq, condition != 0);

  pr_info("woken up by the work job\n");
  return 0;
}

void my_exit(void)
{
printk("waitqueue example cleanup\n");
}
module_init(my_init);
module_exit(my_exit);
MODULE_AUTHOR("John Madieu <john.madieu@foobar.com>");
MODULE_LICENSE("GPL");
```
Và kết quả là:

**[342081.385491] Wait queue example**

**[342081.385505] Going to sleep my_init**

**[342081.385515] Waitqueue module handler work_handler**

**[342086.387017] Wake up the sleeping module**

**[342086.387096] woken up by the work job**

**[342092.912033] waitqueue example cleanup**

P/s: Có thể các bạn nên tìm hiểu qua work_queue một chút thì sẽ hiểu cái này hơn.

# 2. Quản lý timer và delay

Thời gian là một trong những tài nguyên được sử dụng nhiều nhất, chỉ sau memory. Người ta dùng thời gian vào hầu hết những việc như: phân bổ công việc, ngủ, lập lịch, timeout, bla bla....

Có 2 loại thời gian là thời gian tuyệt đối và thời gian tương đối. Kernel sử dụng thời gian tuyệt đối để để biết xem bây giờ là mấy giờ, sử dụng thời gian tương đối để lập lịch. Với thời gian tuyệt đối, có một con chip thật tên là real-time clock (RTC) đảm nhiệm vai trò. Với thời gian tương đối, kernel dựa vào các bộ timer nằm trên CPU. Chúng ta sẽ chủ yếu nó về kernel timer.

Ta chia kernel timer thành 2 phần loại:
- Standard timers, hay system timers
- High-resolution timers.

## 2.1 Standard timers

### 2.1.1 Jiffies và HZ

Jiffy là một đơn vị thời gian của kernel, khai báo ở trong ```<linux/jiffies.h>```. ```HZ``` là một macro hằng số, biểu thị số lần ```jiffies``` tăng lên trong 1 giây. Mỗi lần tăng như vậy gọi là 1 ```tick```. ```HZ``` phụ thuộc vào phần cứng cũng như phiên bản của kernel. ```HZ``` có thể config trên một số kiến trúc phần cứng, nhưng ở một số kiến trúc khác thì nó bị fix cứng.

Điều này có nghĩa là ```jiffies``` sẽ tăng ```HZ``` lần trong 1 giây. Nếu ```HZ``` = 1000, nghĩa là ```jiffies``` tăng 1000 lần (1 tick mỗi 1/1000 giây). Khi đã được định nghĩa, sẽ có một bộ ngắt timer là progammable interrupt timer (PIT, một thành phần phần cứng), được lập trình để  jiffies tăng khi ngắt của PIT báo đến. 

Tùy thuộc vào platform mà ```jiffies``` có thể bị đếm tràn. Với hệ 32-bit mà HZ = 1000 thì mất khoảng 50 ngày để nó đếm tràn, con nố này là 600 triệu năm đối với hệ 64-bit :v. Để sợ không bị đếm tràn thì ta sử dụng biến ```jiffies``` 64-bit, trên cả hệ 32-bit. Nghĩa là có một biến được sử dụng thêm, được định nghĩa trong ```<linux/jiffies.h>```:
```c
extern u64 jiffies_64;
```
Ở hệ 32-bit thì ```jiffies``` là 32 bit thấp, ```jiffies_64``` là các bit cao. Ở hệ 64-bit thì ```jiffies``` = ```jiffies_64```.

### 2.1.2 Timers API

Timer hiện diện trong kernel dưới dạng một đối tượng struct timer_list:
```c
#include <linux/timer.h>
struct timer_list {
  struct list_head entry;
  unsigned long expires;
  struct tvec_t_base_s *base;
  void (*function)(unsigned long);
  unsigned long data;
);
```
trong đó thì ```expires``` là một giá trị tuyệt đối ở trong ```jiffies```. ```entry``` là một danh sách liên kết đôi. ```data``` là một tham số không bắt buộc, nó là tham số truyền vào của con trỏ hàm ```function```.

#### 2.1.2.1 Khởi tạo timer

Các bước sau dùng để khởi tạo và hủy timer:
1. Cài đặt timer bằng hàm:
```c
void setup_timer( struct timer_list *timer, \
void (*function)(unsigned long), \
unsigned long data);
```
Ta cũng có thể sử dụng hàm 
```c 
void init_timer(struct timer_list *timer);
```
vì thực ra ```setup_timer``` là một wrapper của ```init_timer```.

2. Cài đặt thời gian hết hạn timer:
```c
int mod_timer( struct timer_list *timer, unsigned long expires);
```

3. Giải phóng timer: khi đã xong việc timer thì cần phải giải phóng nó:
```c
void del_timer(struct timer_list *timer);
int del_timer_sync(struct timer_list *timer);
```
Hàm ```del_timer``` này có thể hủy cả kernel timer đã được kích hoạt hoặc chưa. Nó sẽ trả về 0 nếu tác động lên kernel timer chưa được kích hoạt, ngược lại nó sẽ trả về 1. Dùng hàm ```del_timer_sync``` để chờ cho hàm của kernel timer thực hiện xong (nếu đang chạy) rồi mới hủy timer này. Ta có thể check độc lập xem timer có còn chạy nữa hay không:
```c
int timer_pending( const struct timer_list *timer);
```

#### 2.1.2.2 Ví dụ
```c
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/timer.h>

static struct timer_list my_timer;
void my_timer_callback(unsigned long data)
{
  printk("%s called (%ld).\n", __FUNCTION__, jiffies);
}

static int __init my_init(void)
{
  int retval;
  printk("Timer module loaded\n");
  setup_timer(&my_timer, my_timer_callback, 0);
  printk("Setup timer to fire in 300ms (%ld)\n", jiffies);
  retval = mod_timer( &my_timer, jiffies + msecs_to_jiffies(300) );
  if (retval)
    printk("Timer firing failed\n");
  return 0;
}

static void my_exit(void)
{
  int retval;
  retval = del_timer(&my_timer);
  /* Is timer still active (1) or no (0) */
  if (retval)
    printk("The timer is still in use...\n");

  pr_info("Timer module unloaded\n");
}
module_init(my_init);
module_exit(my_exit);
MODULE_AUTHOR("John Madieu <john.madieu@gmail.com>");
MODULE_DESCRIPTION("Standard timer example");
MODULE_LICENSE("GPL");
```

## 2.2 High resolution timers (HRTs)

Về cơ bản thì standard timers vẫn còn thiếu sự chính xác và không phù hợp với các ứng dụng realtime. Vì thế high resolution timers, được giới thiệu từ kernel v2.6.16 (bằng cách enable ```CONFIG_HIGH_RES_TIMERS``` option khi build kernel) có độ chính xác micro giây ( có khi là nano giây trên một số platform). Như vậy HRTs có độ chính xác cao hơn nhiều so với standard timers (chỉ là milli giây). Trong khi standard timer phụ thuộc vào ```HZ``` thì HRT phụ thuộc vào ```ktime```.

Và để sử dụng được HRT thì kernel lẫn hardware đều phải hỗ trợ. Nghĩa là không phải platform nào cũng chạy được HRT.

### 2.2.1 HRT API
Để sử dụng được các API của HRT thì cần ```#include <linux/hrtimer.h>```.
Một HRT trong kernel là một đối tượng của struct ```hrtimer```:
```c
struct hrtimer {
  struct timerqueue_node node;
  ktime_t _softexpires;
  enum hrtimer_restart (*function)(struct hrtimer *);
  struct hrtimer_clock_base *base;
  u8 state;
  u8 is_rel;
};
```

#### 2.2.1.1 Khởi tạo và hủy HRT

1. Khởi tạo: Trước khi khởi tạo thì cần phải setup ```ktime```. Sau đó dùng hàm:
```c
void hrtimer_init( struct hrtimer *time, clockid_t which_clock, enum hrtimer_mode mode);
```

2. Khởi động:
```c
int hrtimer_start( struct hrtimer *timer, ktime_t time, const enum hrtimer_mode mode);
```

Tham số ```mode``` đóng vai trò là expiry mode. Nó có thể là ```HRTIMER_MODE_ABS``` với giá trị thời gian tuyệt đối hoặc ```HRTIMER_MODE_REL``` với giá trị thời gian tương đối so với thời điểm bây giờ.

3. Hủy: Có thể sử dụng 2 hàm:
```c
int hrtimer_cancel( struct hrtimer *timer);
int hrtimer_try_to_cancel(struct hrtimer *timer);
```
Cả hai hàm đều return 0 nếu timer chưa kích hoạt và return 1 nếu timer đã được kích hoạt. Sự khác biệt là ```hrtimer_try_to_cancel``` sẽ fail nếu timer đang kích hoạt hoặc hàm callback đang chạy, nó sẽ trả về -1. Trong khi ```hrtimer_cancel``` sẽ đợi cho đến khi hàm callback hoàn tất.

Ta có thể check xem hàm callback có còn chạy nữa hay không:
```c
int hrtimer_callback_running(struct hrtimer *timer);
```
Thực ra hàm ```hrtimer_try_to_cancel``` cũng gọi hàm ```hrtimer_callback_running``` để kiểm tra đấy.

P/s: Để trách việc timer tự động restart thì hàm callback của hrtimer  phải trả về ```HRTIMER_NORESTART```

Ta có thể check xem HRT có ở trên system hay không bằng cách:
- Xem config file, đại loại như ```CONFIG_HIGH_RES_TIMERS=y: zcat /proc/configs.gz | grep CONFIG_HIGH_RES_TIMERS .```
- Xem ở trong ```cat /proc/timer_list``` hoặc ```cat /proc/timer_list | grep resolution```. Nếu ```.resolution``` hiện ra 1 nsecs và ```event_handler``` hiện ```hrtimer_interrupts``` thì ok.
- Dùng system call ```clock_getres```.
- Trong kernel code dùng ```#ifdef CONFIG_HIGH_RES_TIMERS ```.

## 2.3 Delay và sleep trong kernel

Để code delay được thì đầu tiên phải include ```<linux/delay>``` đã. Về cơ bản thì có 2 loại delay, tùy thuộc vào context mà code chúng ta đang chạy: atomic và nonatomic.

### 2.3.1 Atomic context

Atomic dịch ra có nghĩa là nguyên tử. Hàm ý nguyên tử nghĩa là không thể chia nhỏ ra hơn được nữa. Theo lý đó, task chạy trong atomic context không được phép ngủ, không được lập lịch, đã chạy là chạy một mạch đến khi xong. Vậy muốn delay một task như vậy không ngoài cách nào khác là dùng một vòng lặp busy-wait. Kernel cung cấp cho chúng ta một họ hàm Xdelay, về cơ bản các hàm này sẽ tạo một vòng lặp và tiêu tốn thời gian vào đó:
- ```ndelay(unsigned long nsecs)```
- ```udelay(unsigned long usecs)```
- ```mdelay(unsigned long msecs)```

Ta chỉ nên sử dụng hàm ```udelay()``` vì hàm ```ndelay()``` có chuẩn hay không phụ thuộc nhiều vào hardware nữa. 

Timer handlers (hàm callbacks mà ta bàn ở phần timer phía trên) được chạy trong atomic context, nghĩa là nó không được phép ngủ. 

### 2.3.2 Nonatomic context

Nonatomic context thì ngược lại, task được phép ngủ. Kernel cung cấp họ hàm cho việc ngủ ở context này, phụ thuộc vào nhu cầu mà mình muốn delay nó bao lâu:
- ```udelay(unsigned long usecs)```: có vẻ hàm này là hàm nhắc lại phía bên trên, là một vòng lặp busy-wait. Ta nên sử dụng những hàm này vào việc ngủ trong ít µsecs ( < ~10 us ).
- ```usleep_range(unsigned long min, unsigned long max)```: Hàm này dùng hrtimers, và dùng cho việc delay trong tầm  ~µsecs hoặc msecs (10
us - 20 ms). Ở vùng này nên tránh dùng ```udelay()```
- ```msleep(unsigned long msecs)```: Dùng jiffies/legacy_timers. TA nên dùng với nhu cầu trên 10ms+.

Nhắc lại vẫn có thể dùng các hàm ở atomic context ở đây nhưng như thế rất phí phạm, tại sao được ngủ thì lại thức để delay làm gì. Chỉ khi delay với độ dài nhỏ thì mới nên dùng, như ở trường hợp hàm ```udelay()```.

Ta dừng lại phần 2 này ở đây, những thứ hay ho tiếp theo mình sẽ viết trong phần 3.


