---
layout: post
title: Kernel Facilities and Helper Functions (Part 3)  
subtitle:  Những công cụ và hàm hay dùng trong kernel (Phần 3)
gh-repo: 
gh-badge: [star, fork, follow]
tags: [kernel, embedded, device driver]
---

_Tham khảo từ Linux Device Drivers Development_

Chào mừng các bạn đã đến với phần 3. Ở phần này, mình sẽ viết tiếp về 2 cơ chế thông dụng nhất của kernel nữa. Đó là cơ chế lock và cơ chế defer

# 1. Cơ chế locking của kernel

Locking là một cơ chế giúp chia sẻ tài nguyên giữa các threads cũng như processes mà tránh được tranh chấp (race condition) giữa chúng. Một tài nguyên chia sẻ có thể là data hoặc 1 device mà có thể truy cập bởi ít nhất 2 user, có thể đồng thời hoặc không. Cơ chế này bảo vệ tài nguyên khỏi bị làm dụng truy cập, ví dụ như một process thì cứ đòi ghi data vào vùng nhớ trong khi một process lại đang đọc data ở vùng nhớ đấy, hoặc ví dụ khác là 2 process cùng truy cập vào 1 device như GPIO. Kernel cung cấp vài cơ chế locking. Quan trọng nhất bao gồm:
- Mutex
- Semaphore
- Spinlock

Phần này mình chỉ đề cập tới Mutex và spinlock vì cách hoạt động khác nhau. Semaphore là dạng tổng quát của mutex nên các bạn có thể tìm hiểu thêm sau.

## 1.1 Mutex
Mutex là viết tắt của Mutual exclusion, trên thực tế nó là cơ chế hay được sử dụng nhất. Để hiểu được cách hoạt động thì nhìn vào source code ở ```include/linux/mutex.h```:
```c
struct mutex {
  /* 1: unlocked, 0: locked, negative: locked, possible waiters */
  atomic_t count;
  spinlock_t wait_lock;
  struct list_head wait_list;
  [...]
}
```
Ta lại được thấy một struct ```list_head``` ở đây, đó là ```wait_list```. Bản chất ở đây sẽ tương tự như với wait_queue (xin nhắc lại là với ```wait_queue```, những task nào cần ngủ sẽ được đưa vào một hàng đợi và đợi có tín hiệu đánh thức).

Hãy tưởng tượng mutex như một cái khóa. Process nào muốn sử dụng tài nguyên thì cần có chìa khóa này. Để đạt được chìa khóa, process A sẽ đi hỏi xem chìa khóa có không. Nếu có, process A sẽ lấy chìa và làm việc với tài nguyên, làm xong thì trả khóa, và nếu trong thời gian này, có một process B khác hỏi khóa, process B sẽ bị đưa vào ```wait_list``` ở trạng thái ngủ và đợi đến khi process A trả khóa thì mới lấy khóa. Nếu process A từ đầu không lấy được chìa có nghĩa là chìa đang bị trưng dụng bởi process nào đó, process A sẽ bị đưa vào ```wait_list``` giống như trường hợp ở process B. 

### 1.1.1 Mutex API
Một số hàm sử dụng với mutex:

#### Khai báo
- Tĩnh:
```c
DEFINE_MUTEX(my_mutex);
```

- Động:
```c
struct mutex my_mutex;
mutex_init(&my_mutex);
```

#### Acquire và release
- Lock:
```c
void mutex_lock(struct mutex *lock);
int mutex_lock_interruptible(struct mutex *lock);
int mutex_lock_killable(struct mutex *lock);
```

- Unlock:
```c
void mutex_unlock(struct mutex *lock);
```

Đôi lúc nếu muốn ta có thể kiểm tra xem mutex có bị lock hay không bằng cách sử dụng hàm:
```c
int mutex_is_locked(struct mutex *lock);
```

Hàm này sẽ kiểm tra xem liệu process chứa mutex có ```NULL``` hay không. Ngoài hàm ```mutex_lock``` còn có hàm ```mutex_trylock```, sự khác biệt là hàm ```mutex_lock``` sẽ làm cho process ngủ đến khi lấy được mutex, còn với ```mutex_trylock``` thì sẽ trả về giá trị ngay, nếu lấy được mutex trả về 1, ngược lại trả về 0:
```c
int mutex_trylock(struct mutex *lock);
```

Cũng giống như họ hàm interruptible của wait queue, hàm ```mutex_lock_interruptible()``` khuyến khích được sử dụng. Khi sử dụng hàm này, những process không lấy được mutex được đưa vào một hàng đợi có thể bị ngắt, nghĩa là có thể đánh thức những process trong hàng đợi mutex bằng một signal nào đó. Đối với ```mutex_lock``` thì việc đợi là không thể bị ngắt. Hàm ```mutex_lock_killable()``` thì những process trong hàng đợi mutex chỉ có thể bị ngắt bởi tín hiệu kill process.

Như vậy nên cẩn thận khi sử dụng hàm ```mutex_lock()```, hãy sử dụng khi đảm bảo khi mutex chắc chắn được giải phóng, và nên giải phóng trong khoảng thời gian ngắn. Ở user context thì nên luôn sử dụng ```mutex_lock_interruptible()``` vì ```mutex_lock()``` cũng không bị ngắt ngay cả khi nhận tín hiệu ```ctrl+c```.

Sau đây là ví dụ:
```c
struct mutex my_mutex;
mutex_init(&my_mutex);
/* inside a work or a thread */
mutex_lock(&my_mutex);
access_shared_memory();
mutex_unlock(&my_mutex);
```

Tổng kết lại, có những sự thật khi dùng mutex:
- Chỉ có 1 task có thể giữ mutex tại một thời điểm
- Phải được khởi tạo bằng các API
- Task giữ mutex có thể exit để lại hậu quả là mutex vẫn bị khóa khiến cho các process đang đợi nó sẽ ngủ mãi mãi
- Vùng nhớ của task nắm giữ lock không được phép giải phóng ??? 
- Task giữ mutex không được khởi tạo lại
- Vì những thao tác sẽ bao gồm việc lập lịch lại, mutex không nên sử dụng trong atomic context, giống như tasklet và timer.

Giống như ```wait_queue```, mutex không dùng cơ chế thăm dò. Mỗi khi ```mutex_unlock``` được gọi, kernel sẽ kiểm tra những process đang đợi trong ```wait_list```. Một trong số chúng được đánh thức theo thứ tự mà chúng được đi ngủ. 

## 1.2 Spinlock

Spin có nghĩa là quay, hàm ý sẽ dùng cơ chế hỏi quay vòng cho đến khi đạt được lock, chứ không phải cho vào hàng đợi như mutex.
Bất cứ thread nào cần spinlock sẽ tạo ra một vòng lặp cho đến khi đạt được spinlock mới thoát ra khỏi vòng lặp nên spinlock sẽ tiêu tốn CPU rất nhiều. Vậy nên chỉ dùng cho những lần hỏi lock thật nhanh, đặc biệt là khi thời gian giữ spinlock nhỏ hơn thời gian lập lịch lại. Spinlock cần được giải phóng ngay khi critical task hoàn thành, không được dây dưa.

Để tránh việc lãng phí CPU (lãng phí như là lập lịch cho một thread có thể đang spin, hoặc là đợi một spinlock đang được giữ một process ngủ @@), kernel sẽ vô hiệu hóa việc chen hàng khi code giữ spinlock đang chạy. Chen hàng là hiện tượng một process A đang chạy thì bị ngắt, sau khi thực hiện ngắt xong thì process B lại được thực hiện thay vì process A. Nghĩa là phần code này sẽ chạy ngay mà không được phép ngủ (dù cho bị ngắt thì thực hiện ngắt xong phải quay lại thực hiện tiếp) để làm những process đang đợi kia không tiêu tốn thời gian và CPU.

Miễn là có một task nào đó đang giữ spinlock, những task khác có thể phải đang spin khi đang đợi nó. Khi sử dụng spinlock, cần phải đảm bảo rằng nó sẽ không bị giữ trong một thời gian dài. Việc spin trên một processor có nghĩa rằng chẳng còn task nào có thể chạy trên processor đó ở thời điểm đó, như vậy việc sử dụng spinlock trên các bộ vi xử lý 1 core là vô nghĩa, và có thể dẫn tới việc làm chậm hệ thống, thậm chí là deadlock. Ở các bộ vxl 1 core, ta nên sử dụng cặp đôi 
```spin_lock_irqsave()``` và ```spin_unlock_irqrestore()```, bộ đôi này tương ứng sẽ vô hiệu hóa ngắt trên CPU, tránh việc tranh chấp khi xảy ra ngắt.

Khi mà ta không biết được ta cần viết driver  cho hệ thống như thế nào, nói chung nên sử dụng ```spin_lock_irqsave(spinlock_t *lock, unsigned long flags)```, hàm này sẽ vô hiệu hóa ngắt trên processor đang chạy nó. Hàm ```spin_lock_irqsave``` sẽ gọi tới ```local_irq_save(flags)```, đây là một hàm phụ thuộc kiến trúc hệ thống, mục đích là lưu giữ trạng thái IRQ, và ```preempt_disable()``` để vô hiệu hóa việc chen hàng trên những CPU liên quan. Sau đó nên dùng hàm ```spin_unlock_irqrestore()``` để khôi phục lại những thao tác trước. 
Ví dụ dưới đây là một irq handler:
```c
/* some where */
spinlock_t my_spinlock;
spin_lock_init(my_spinlock);
static irqreturn_t my_irq_handler(int irq, void *data)
{
  unsigned long status, flags;
  spin_lock_irqsave(&my_spinlock, flags);
  status = access_shared_resources();
  spin_unlock_irqrestore(&gpio->slock, flags);
  return IRQ_HANDLED;
}
```

## 1.3 Spinlock vs Mutex

Vấn đề đối với mutex là đặt những process (hoặc thread) về trạng thái ngủ và đưa vào hàng đợi rồi lại đánh thức chúng khi có tín hiệu, những thao tác này tiêu tốn một lượng thời gian và lệnh CPU nhất định. Vấn đề với spinlock là việc nếu càng muốn bảo vệ tài nguyên càng lâu thì tiêu tốn càng nhiều CPU. Như vậy ta chỉ muốn sử dụng spinlock trong những trường hợp mà xử lý tài nguyên thật nhanh để một process khác không rơi vào vòng lặp spin quá lâu, trong khi những trường hợp muốn bảo vệ tài nguyên dài hơi, nên sử dụng mutex. Ngoài ra với spinlock, thread nắm giữ nó không được phép ngủ, nếu ngủ thì thời gian chờ đợi sẽ rất lâu. Vậy nên sinlock và mutex có hai mục đích khác nhau:
- Mutex bảo vệ tài nguyên critical của process, trong khi spinlock bảo vệ vùng critical của trình phục vụ ngắt (irq handler).

# 2. Cơ chế work deferring 

Defer dịch sang tiếng việt là "hoãn" nhưng có nghĩa có vẻ không chuẩn lắm nên mình cứ để nguyên. Ta hiểu deferring là cơ chế mà ta lập lịch cho một công việc nào đấy được thực hiện trong tương lai. Nghĩa là ta sẽ khai báo một công việc nào đấy, rồi thông báo kernel cho phép việc này chạy trong tương lai gần, còn chính xác là chạy khi nào thì bạn không biết, kernel sẽ lo việc đó. Việc defer work là vô cùng quan trọng, nhất là trong việc xử lý ngắt (sẽ bàn sau). Đương nhiên là kernel sẽ cung cấp cho ta những công cụ để thực hiện cơ chế này. Có 3 cách để thực hiện nhiệm vụ này:
- Tasklets: Chạy trong atomic context.
- Workqueues: Chạy trong process context.
Nhắc lại là trong atomic context thì task không được phép ngủ. 
Tasklets và workqueus đều được triển khai ở bottom-half của một interrupt handler (vấn đề này sẽ được bàn sau)
## 2.1 Tasklet
Tasklet trong kernel được đại diện bằng struct tasklet_struct:
```c
struct tasklet_struct
{
  struct tasklet_struct *next;
  unsigned long state;
  atomic_t count;
  void (*func)(unsigned long);
  unsigned long data;
};
```

Về cơ bản thì tasklet là không reentrant. Một đoạn code được gọi là ```reentrant``` nếu nó có thể bị ngắt khi đang thực thi, rồi sau đó có thể được thực thi tiếp. Tasklet được thiết kế để có thể chạy đồng thời trên chỉ 1 CPU (ngay cả trên SMP system), đó là CPU mà nó được lập lịch, những tasklet khác có thể chạy đồng thời trên những CPUs khác. Những API của tasklet khá đơn giản:
### 2.1.1 Khai báo một tasklet
- Động:
```c
void tasklet_init(struct tasklet_struct *t, void (*func)(unsigned long), unsigned long data);
```

- Tĩnh:
```c
DECLARE_TASKLET( tasklet_example, tasklet_function, tasklet_data );
DECLARE_TASKLET_DISABLED(name, func, data);
```

Chỉ có một điểm khác biệt giữa hai hàm này. Hàm đầu tạo một tasklet đã được enable và sẵn sàng để được lập lịch mà không cần gọi hàm nào nữa (bằng cách set trường ```count``` = 0), trong khi hàm sau thì tạo một tasklet bị disable (bằng cách set trường ```count``` = 1), do đó sau hàm này phải gọi tiếp hàm ```tasklet_enable()``` trước khi tasklet có thể lập lịch:
```c
#define DECLARE_TASKLET(name, func, data) \
struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(0), func, data }

#define DECLARE_TASKLET_DISABLED(name, func, data) \
struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(1), func, data }
```

### 2.1.2 Bật và vô hiệu hóa tasklet
Để kích hoạt một tasklet, ta dùng hàm sau:
```c
void tasklet_enable(struct tasklet_struct *);
```

Ở những phiên bản kernel cũ ta có thể thấy hàm 
```c
void tasklet_hi_enable(struct tasklet_struct *);
```

được sử dụng. Tuy nhiên hai hàm này thực chất là một. 
Để vô hiệu hóa một tasklet, ta dùng hàm
```c
void tasklet_disable(struct tasklet_struct *);
```

Ta cũng có thể dùng hàm
```c
void tasklet_disable_nosync(struct tasklet_struct *);
```

hàm ```tasklet_disable``` sẽ vô hiệu hóa tasklet và trả về giá trị chỉ khi tasklet đã kết thúc việc thực thi (nếu đã từng thực thi), trong khi hàm ```tasklet_disable_nosync``` thì trả về lập tức, cho dù là việc thực khi đã kết thúc hay chưa.

### 2.1.3 Lập lịch cho tasklet
Có 2 hàm được sử dụng để lập lịch cho tasklet, tùy thuộc vào độ ưu tiên mà bạn muốn trao cho nó:
```c
void tasklet_schedule(struct tasklet_struct *t);
void tasklet_hi_schedule(struct tasklet_struct *t);
```

Kernel duy trì 2 danh sách, một danh sách mức ưu tiên thông thường, danh sách cài lại mức ưu tiên cao. ```tasklet_schedule``` đưa tasklet vào danh sách mức ưu tiên thông thường, còn ```tasklet_hi_schedule``` thì mức ưu tiên cao. Ta sẽ cần sử dụng độ ưu tiên cao khi cần thực hiện interrupt handler với độ trễ thấp. Có một số tính chất của tasklet:
- Gọi ```tasklet_schedule``` đối với một tasklet đã được lập lịch trước đó mà tasklet đó còn chưa được thực thi thì sẽ có chẳng chuyện gì xảy ra. 
- ```tasklet_schedule``` có thể được gọi ngay bên trong một tasklet, nghĩa là tasklet có thể lập lịch cho chính nó.
- Các tasklet có độ ưu tiên cao luôn được thực hiện trước những cái có độ ưu tiên thông thường. Tuy nhiên việc lạm dụng độ ưu tiên cao sẽ làm tăng độ trễ của toàn hệ thống. Vậy nên chỉ sử dụng độ ưu tiên cao cho những công việc nhanh và cần thiết.

Ta có thể dừng ngay một tasklet bằng cách sử dụng hàm ```tasklet_kill```. Hàm này sẽ ngăn tasklet sẽ chạy(tức là đã đc lập lịch mà muốn tasklet thôi không chạy nữa), hoặc đợi tasklet hoàn thành (nếu đang chạy) rồi kill nó:
```c
void tasklet_kill(struct tasklet_struct *t);
```

Xem ví dụ sau:
```c
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/interrupt.h>
/* for tasklets API */
char tasklet_data[]="We use a string; but it could be pointer to a structure";
/* Tasklet handler, that just print the data */
void tasklet_work(unsigned long data)
{
  printk("%s\n", (char *)data);
}
DECLARE_TASKLET(my_tasklet, tasklet_function, (unsigned long)
tasklet_data);
static int __init my_init(void)
{
  /*
  * Schedule the handler.
  * * Tasklet arealso scheduled from interrupt handler
  * */
 tasklet_schedule(&my_tasklet);
 return 0;
}

void my_exit(void)
{
  tasklet_kill(&my_tasklet);
}
module_init(my_init);
module_exit(my_exit);
```

## 2.2 Work queues
Có một ví dụ ở phần Wait queue đã nhắc đến Work queue, hôm nay ta sẽ đi xem work queue là gì. 
Work queue được xuất hiện từ Linux kernel 2.6, được sử dụng nhiều nhất (nhiều hơn tasklet). Bạn biết vì sao không? Vì tasklet được sử dụng ở atomic context, còn work queue được sử dụng ở process context. Do process context luôn xảy ra thường xuyên hơn là atomic context nên những gì được áp dụng ở process context sẽ luôn được sử dụng nhiều hơn, điển hình như mutex luôn được sử dụng nhiều hơn là spinlock. Và đây cũng là sự lựa chọn duy nhất khi ta muốn ngủ ở bottom half (nhắc đến nhiều lần quá mà chưa thấy giải thích, cừ từ từ khoai sẽ nhừ, sẽ có bài viết sau). Vì ngủ được, công việc có thể xử lý I/O data, giữ mutext, ngủ, delay.

Có 2 cách để làm việc với workqueue ở kernel:
- Thứ nhât, có một default shared workqueue, được handle bởi một một tập các kernel thread, mỗi thread chạy trên một CPU. Khi có một công việc cần lập lịch, chỉ cần đưa công việc đó vào hàng đợi global này, công việc sẽ được thực hiện trong một thời điểm thích hợp. 
- Thứ hai, ta tự tạo một workqueue rồi thêm công việc vào đó. Điều này có nghĩa là mỗi khi công việc (work handler) cần thực thi, kernel thread của ta sẽ thứ dậy và handle nó, thay vì thread được định nghĩa sẵn của shared work queue.

**Hàm thực hiện công việc được gọi là work handler.**
### 2.2.1 Kernel-global workqueue
Trừ khi ta không có lựa chọn nào khác (hoặc cần một performance thực sự khắt khe, hoặc cần control tất cả những thứ trong workqueue), nếu ta chỉ cần thỉnh thoảng submit task, thì ta nên sử dụng shared workqueue của kernel. Tất nhiên với lưu ý là khi sử dụng đồ chung thì ta cũng nên tốt bụng một tí, đừng độc quyền đồ chung làm của riêng. 

Do việc thực thi các task trong hàng đợi này được sắp xếp theo thứ tự trên mỗi CPU, ta không nên để  một task ngủ lâu vì các task còn lại trong queue sẽ không thực chạy. Ta chẳng biết task của ta phải chia sẻ queue với task nào khác nên việc task của ta, nên đừng bất ngờ khi task của ta phải hơi lâu mới lấy được CPU. 

Với loại workqueue này, ta luôn nên khởi tạo bằng macro ```INIT_WORK```, sau đó chỉ cần truyền struct ```work_struct``` vào. Có 4 hàm dùng để lập lịch work trên shared workqueue:
- hàm đầu tiên sẽ lập lịch cho công việc trên CPU hiện tại:
```c
int schedule_work(struct work_struct *work);
```

- hàm thứ hai giống hàm thứ nhất, trừ việc có delay (delay ở đây nghĩa là khoảng delay trước khi đưa công việc vào workqueue):
```c
static inline bool schedule_delayed_work(struct delayed_work *dwork, unsigned long delay);
```

- hàm thứ 3 lập lịch cho công việc trên một CPU chỉ định:
```c
int schedule_work_on(int cpu, struct work_struct *work);
```

- hàm thứ 4 giống hàm thứ 3, trừ việc có delay:
```c
int scheduled_delayed_work_on(int cpu, struct delayed_work *dwork, unsigned long delay);
```

Và shared workqueue mà nãy giờ chúng ta nhắc tới là ```system_wq```, được định nghĩa ở ```kernel/workqueue.c```:
```c
struct workqueue_struct *system_wq __read_mostly;
EXPORT_SYMBOL(system_wq);
```

Một công việc khi đã được đưa vào shared workqueue có thể bị hủy bởi hàm ```cancel_delayed_work```. Ta cũng có thể flush cái shared workqueue này bằng hàm:
```c
void flush_scheduled_work(void);
```

Ví dụ cụ thể với shared workqueue:
```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/sched.h>    /* for sleep */
#include <linux/wait.h>     /* for wait queue */
#include <linux/time.h>
#include <linux/delay.h>
#include <linux/slab.h>         /* for kmalloc() */
#include <linux/workqueue.h>

//static DECLARE_WAIT_QUEUE_HEAD(my_wq);
static int sleep = 0;

struct work_data {
    struct work_struct my_work;
    wait_queue_head_t my_wq;
    int the_data;
};

static void work_handler(struct work_struct *work)
{
    struct work_data *my_data = container_of(work, \
                                 struct work_data, my_work); 
    pr_info("Work queue module handler: %s, data is %d\n", __FUNCTION__, my_data->the_data);
    msleep(3000);
    wake_up_interruptible(&my_data->my_wq);
    kfree(my_data);
}

static int __init my_init(void)
{
    struct work_data * my_data;

    my_data = kmalloc(sizeof(struct work_data), GFP_KERNEL);
    my_data->the_data = 34;

    INIT_WORK(&my_data->my_work, work_handler);
    init_waitqueue_head(&my_data->my_wq);

    schedule_work(&my_data->my_work);
    pr_info("I'm goint to sleep ...\n");
    wait_event_interruptible(my_data->my_wq, sleep != 0);
    pr_info("I am Waked up...\n");
    return 0;
}

static void __exit my_exit(void)
{
    pr_info("Work queue module exit: %s %d\n", __FUNCTION__,  __LINE__);
}

module_init(my_init);
module_exit(my_exit);
```

Để ý một chút ở ví dụ trên, để truyền dữ liệu vào workqueue handler, ta đã đưa struct ```work_struct``` vào một custom data structure, và sử dụng ```container_of``` để lấy lại được con trỏ custom data. Đây là cách phổ biến nhất để truyền dữ liêu vào một workqueu handler.

### 2.2.2 Tự tạo workqueue
Một work queue được đại diện bằng struct ```workqueue_struct```, còn một công việc bị đưa vào hàng đợi của work queue thì được đại diện bằng struct ```work_struct```. Có 4 bước để lập lịch công việc bằng kernel thread của ta:
- Khai báo/khởi tạo một struct ```workqueue_struct```
- Tạo hàm thực hiện công việc (work handler)
- Tạo struct ```work_struct``` sẽ chứa công việc của ta
- Đưa hàm thực hiện công việc vào ```work_struct```

**Cú pháp**
Những hàm sau được định nghĩa ở include/linux/workqueue.h :
- Khai báo work và workqueue:
```c
struct workqueue_struct *myqueue;
struct work_struct thework;
```

- Định nghĩa hàm thực hiện công việc:
```c
void dowork(void *data) {
/* Code goes here */ };
```

- Khởi tạo workqueue và đưa hàm thực hiện công việc vào:
```c
myqueue = create_singlethread_workqueue( "mywork" );
INIT_WORK( &thework, dowork, <data-pointer> );
```

Ta cũng có thể tạo work queue bằng macro ```create_workqueue```. Sự khác nhau giữa ```create_workqueue``` và ```create_singlethread_workqueue``` là hàm ```create_workqueue```() sử dụng một thread cho mỗi CPU và ```create_singlethread_workqueue```() chỉ sử dụng 1 thread.

- Lập lịch công việc:
```c
queue_work(myqueue, &thework);
```

hoặc có thể lập lịch công việc sau một khoảng delay (delay ở đây nghĩa là khoảng delay trước khi đưa công việc vào workqueue):
```c
queue_dalayed_work(myqueue, &thework, <delay>);
```

Hai hàm trên trả về ```false``` nếu công việc đã được đưa lên một queue nào đó, trả về true nếu ngược lại. Chú ý là khoảng thời gian delay này là số ```jiffies``` phải đợi để được đưa vào hàng đợi. Ta có thể sử dụng hàm ```msecs_to_jiffies``` để chuyển đổ từ đơn vị micro giây sang ```jifffies```, ví dụ nếu cần delay 5ms:
```c
queue_delayed_work(myqueue, &thework, msecs_to_jiffies(5));
```

- Hàm ```flush``` đảm bảo rẳng tất bất kì công việc đã được lập lịch đều phải chạy và hoàn tất:
```c
void flush_workqueue(struct workqueue_struct *wq)
```

ngủ cho đến khi tất cả các công việc trong hàng đợi hoàn tất việc thực thi. Một công việc mới thêm vào workqueue sẽ không ảnh hưởng đến việc ngủ. Hàm này thường được dùng trong driver shutdown handler.

- Cleanup:
Sử dụng ```cancel_work_sync()``` hoặc ```cancel_delayed_work_sync()``` cho việc hủy công việc một cách đồng bộ, tức là nó sẽ hủy các công việc
nếu đã lập lịch nhưng chưa chạy, hoặc đợi cho đến khi công việc hoàn tất nếu đã chạy. Công việc sẽ bị hủy ngay cả khi nó tự đưa nó vào workqueue: 
```c
int cancel_work_sync(struct work_struct *work);
int cancel_delayed_work_sync(struct delayed_work *dwork);
```

Kể từ linux version 4.8 thì có thể dùng hai hàm ```cancel_work``` và ```cancel_delayed_work``` cho việc hủy công việc một cách không đồng bộ. Ngoài ra ta phải check liệu hàm có trả về ```true``` hay không.
```c

if ( !cancel_delayed_work( &thework) ){ //Nếu không thể hủy công việc
  flush_workqueue(myqueue);
  destroy_workqueue(myqueue);
}
```

Cũng phương pháp dùng workqueue tự tạo này, có một cách khác mà sẽ tạo ra chỉ một thread để handler trên tất cả các CPU. Trong trường hợp cần delay trước khi đưa công việc vào hàng đợi, thì ta sẽ khởi tạo bằng:
```c
INIT_DELAYED_WORK(_work, _func);
INIT_DELAYED_WORK_DEFERRABLE(_work, _func);
```

Rồi sau đó ta sử dụng hàm sau để lập lịch:
```c
int queue_delayed_work(struct workqueue_struct *wq, struct delayed_work *dwork, unsigned long delay);
```

```dwork``` này sẽ phải thực hiện trên CPU hiện tại. Ta có thể chỉ định cho ```dwork``` thực hiện trên CPU khác:
```c
int queue_delayed_work_on(int cpu, struct workqueue_struct *wq, struct delayed_work *dwork, unsigned long delay);
```

Cuối cùng mời bạn đọc ví dụ cụ thể sau:
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/workqueue.h>    /* for work queue */
#include <linux/slab.h>         /* for kmalloc() */

struct workqueue_struct *wq;
 
struct work_data {
    struct work_struct my_work;
    int the_data;
};
 
static void work_handler(struct work_struct *work)
{
    struct work_data * my_data = container_of(work, struct work_data, my_work);
    pr_info("Work queue module handler: %s, data is %d\n",
         __FUNCTION__, my_data->the_data);
    kfree(my_data);
}

static int __init my_init(void)
{
    struct work_data * my_data;

    pr_info("Work queue module init: %s %d\n", __FUNCTION__, __LINE__);
    wq = create_singlethread_workqueue("my_single_thread");
    my_data = kmalloc(sizeof(struct work_data), GFP_KERNEL);

    my_data->the_data = 34;

    INIT_WORK(&my_data->my_work, work_handler);
    queue_work(wq, &my_data->my_work);
 
    return 0;
}

static void __exit my_exit(void)
{
    flush_workqueue(wq);
    destroy_workqueue(wq);
    pr_info("Work queue module exit: %s %d\n", __FUNCTION__, __LINE__);
}

module_init(my_init);
module_exit(my_exit);
```

### 2.2.3 Kernel-global workqueue vs workqueue tự tạo

| Predefined work queue function | Equivalent standard work queue function | 
| :------ |:--- |
| ```schedule_work(w)``` | ```queue_work(keventd_wq,w)``` | 
| ```schedule_delayed_work(w,d)``` | ```queue_delayed_work(keventd_wq,w,d)``` | 
| ```schedule_delayed_work_on(cpu,w,d)``` | ```queue_delayed_work(keventd_wq,w,d)``` | 
| ```flush_scheduled_work()``` | ```flush_workqueue(keventd_wq)``` | 
















