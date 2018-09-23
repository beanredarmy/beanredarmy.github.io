---
layout: post
title: Kernel Facilities and Helper Functions (Part 2)  
subtitle:  Những công cụ và hàm hay dùng trong kernel (Phần 2)
gh-repo: 
gh-badge: [star, fork, follow]
tags: [kernel, embedded]
---

_Tham khảo từ Linux Device Drivers Development_

Ở phần 1 ta đã tìm hiểu về 2 thứ hữu ích được sử dụng rất nhiều trong kernel đó là marco container_of và cấu trúc dữ liệu list_head. Ở phần này chúng ta sẽ tìm hiểu tiếp 2 cơ chế cũng rất quan trọng đó là cơ chể ngủ và cơ chế timer trong Linux. Let's get start!!!

# 1. Cơ chế ngủ của kernel
Ngủ là gì? Ai ngủ? 
Ngủ là cơ chế mà theo đó, process sẽ tha cho bộ xử lý, để bộ xử lý dành thời gian xử lý process khác. Lý do mà một tiến trình hay process cần ngủ là để xem dữ liệu đã available hay chưa, hoặc đợi tài nguyên được giải phóng. 

Bộ lập lịch của kernel quản lí một danh sách những tác vụ đang chạy, gọi là một run queue. Các process cần được ngủ sẽ bị xóa khỏi run queue và không được lập lịch nữa. Trừ khi có một thứ gì đấy tác động, đánh thức chúng dậy, nếu không thì chúng cứ ngủ vậy mãi. Chúng ta có thể cho một process đi ngủ để đợi tài nguyên giải phóng, rồi tạo ra một điều kiện gì đó để đánh thức nó dậy. Để implement cơ chế này thì kernel cung cấp một tập các function cũng như data structure.

## 2. Wait queue.
Khi bị loại bỏ khỏi run queue, một process ngủ sẽ đi vào wait queue. Wait queue làm ột thành phần không thể thiếu trong các process blocked I/O. Để hiểu cách hoạt động của wait queu, xem source code thôi:
```c
struct __wait_queue {
  unsigned int flag;
  #define WQ_FLAG_EXCLUSIVE 0x01
  void *private;
  wait_queue_func_t func;
  struct list_head task_list;
}
```

Wow, ta lại thấy một thứ khá quen thuộc, là list_head. Đó là một linked list. Tất cả những process mà cần đi ngủ thì đêu được đưa vào hàng đợi này (do đó nó có tên là wait queue) và được đặt vào trạng thái ngủ cho đến khi một điều kiện trở thành true và đánh thức nó.

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

Hàm wait_even_interruptible không hỏi dò mà đơn giản chỉ đánh giá điều kiện, nếu điều kiện sai thì process sẽ được chuyển thành trạng thái TASK_INTERRUPTIBLE và xóa khỏi run queue. Khi gọi hàm wake_up_interruptible thì điều kiện này lại được check lại, nếu đúng thì process đang ở trong wait queue sẽ được đánh thức, chuyển thành trạng thái TASK_RUNNING. Để đánh thức tất cả các process trong queue thì dùng hàm wake_up_interruptible_all();

_Note: Thực ra những hàm chính thức là wait_event, wake_up và wake_up_all, tuy nhiên những hàm hày lại chỉ được sử dụng với những process mà không thể bị ngắt bởi một signal. Nên chúng chỉ nên được sử dụng trong các critical task (những task không cho phép ngắt). Do những process trên không bị ngắt bởi signal, ta nên kiểm tra giá trị trả về của các hàm. Giá trị trả vể khác 0 nghĩa là việc ngủ của process đã bị ngắt bởi một signal nào đó, và driver nên trả về lỗi ERESTARTSYS_ 

Nếu ta gọi wake_up hoặc wake_up_interruptible mà điều kiện vẫn FALSE, thì chẳng có gì xảy ra cả. Nếu không dùng 2 hàm này thì process cũng chẳng bao giờ được thức.

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
