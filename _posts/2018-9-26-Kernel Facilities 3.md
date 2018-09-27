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
Ta lại được thấy một struct list_head ở đây, đó là wait_list. Bản chất ở đây sẽ tương tự như với wait_queue (xin nhắc lại là với wait_queue, những task nào cần ngủ sẽ được đưa vào một hàng đợi và đợi có tín hiệu đánh thức).

Hãy tưởng tượng mutex như một cái khóa. Process nào muốn sử dụng tài nguyên thì cần có chìa khóa này. Để đạt được chìa khóa, process A sẽ đi hỏi xem chìa khóa có không. Nếu có, process A sẽ lấy chìa và làm việc với tài nguyên, làm xong thì trả khóa, và nếu trong thời gian này, có một process B khác hỏi khóa, process B sẽ bị đưa vào wait_list ở trạng thái ngủ và đợi đến khi process A trả khóa thì mới lấy khóa. Nếu process A từ đầu không lấy được chìa có nghĩa là chìa đang bị trưng dụng bởi process nào đó, process A sẽ bị đưa vào wait_list giống như trường hợp ở process B. 

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

Hàm này sẽ kiểm tra xem liệu process chứa mutex có NULL hay không. Ngoài hàm mutex_lock còn có hàm mutex_trylock, sự khác biệt là hàm mutex_lock sẽ làm cho process ngủ đến khi lấy được mutex, còn với mutex_trylock thì sẽ trả về giá trị ngay, nếu lấy được mutex trả về 1, ngược lại trả về 0:
```c
int mutex_trylock(struct mutex *lock);
```

Cũng giống như họ hàm interruptible của wait queue, hàm mutex_lock_interruptible() khuyến khích được sử dụng. Khi sử dụng hàm này, những process không lấy được mutex được đưa vào một hàng đợi có thể bị ngắt, nghĩa là có thể đánh thức những process trong hàng đợi mutex bằng một signal nào đó. Đối với mutex_lock thì việc đợi là không thể bị ngắt. Hàm mutex_lock_killable() thì những process trong hàng đợi mutex chỉ có thể bị ngắt bởi tín hiệu kill process.

Như vậy nên cẩn thận khi sử dụng hàm mutex_lock(), hãy sử dụng khi đảm bảo khi mutex chắc chắn được giải phóng, và nên giải phóng trong khoảng thời gian ngắn. Ở user context thì nên luôn sử dụng mutex_lock_interruptible() vì mutex_lock() cũng không bị ngắt ngay cả khi nhận tín hiệu ctrl+c.

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

Giống như wait_queue, mutex không dùng cơ chế thăm dò. Mỗi khi mutex_unlock được gọi, kernel sẽ kiểm tra những process đang đợi trong wait_List. Một trong số chúng được đánh thức theo thứ tự mà chúng được đi ngủ. 

## 1.2 Spinlock

Spin có nghĩa là quay, hàm ý sẽ dùng cơ chế hỏi quay vòng cho đến khi đạt được lock, chứ không phải cho vào hàng đợi như mutex.
Bất cứ thread nào cần spinlock sẽ tạo ra một vòng lặp cho đến khi đạt được spinlock mới thoát ra khỏi vòng lặp nên spinlock sẽ tiêu tốn CPU rất nhiều. Vậy nên chỉ dùng cho những lần hỏi lock thật nhanh, đặc biệt là khi thời gian giữ spinlock nhỏ hơn thời gian lập lịch lại. Spinlock cần được giải phóng ngay khi critical task hoàn thành, không được dây dưa.

Để tránh việc lãng phí CPU (lãng phí như là lập lịch cho một thread có thể đang spin, hoặc là đợi một spinlock đang được giữ một process ngủ@@), kernel sẽ vô hiệu hóa việc chen hàng khi code giữ spinlock đang chạy. Chen hàng là hiện tượng một process A đang chạy thì bị ngắt, sau khi thực hiện ngắt xong thì process B lại được thực hiện thay vì process A. Nghĩa là phần code này sẽ chạy ngay mà không được phép ngủ (dù cho bị ngắt thì thực hiện ngắt xong phải quay lại thực hiện tiếp) để làm những process đang đợi kia không tiêu tốn thời gian và CPU.

Miễn là có một task nào đó đang giữ spinlock, những task khác có thể phải đang spin khi đang đợi nó. Khi sử dụng spinlock, cần phải đảm bảo rằng nó sẽ không bị giữ trong một thời gian dài. Việc spin trên một processor có nghĩa rằng chẳng còn task nào có thể chạy trên processor đó ở thời điểm đó, như vậy việc sử dụng spinlock trên các bộ vi xử lý 1 core là vô nghĩa, và có thể dẫn tới việc làm chậm hệ thống, thậm chí là deadlock. Ở các bộ vxl 1 core, ta nên sử dụng cặp đôi 
spin_lock_irqsave() và spin_unlock_irqrestore(), bộ đôi này tương ứng sẽ vô hiệu hóa ngắt trên CPU, tránh việc tranh chấp khi xảy ra ngắt.

Khi mà ta không biết được ta cần viết driver  cho hệ thống như thế nào, nói chung nên sử dụng spin_lock_irqsave(spinlock_t *lock, unsigned long flags), hàm này sẽ vô hiệu hóa ngắt trên processor đang chạy nó. Hàm spin_lock_irqsave sẽ gọi tới local_irq_save(flags), đây là một hàm phụ thuộc kiến trúc hệ thống, mục đích là lưu giữ trạng thái IRQ, và preempt_disable() để vô hiệu hóa việc chen hàng trên những CPU liên quan. Sau đó nên dùng hàm spin_unlock_irqrestore() để khôi phục lại những thao tác trước. 
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

Defer dịch sang tiếng việt là "hoãn" nhưng có nghĩa có vẻ không chuẩn lắm nên mình cứ để nguyên. Ta hiểu deferring là cơ chế mà ta lập lịch cho một công việc nào đấy được thực hiện trong tương lai. Đương nhiên là kernel sẽ cung cấp cho ta những công cụ để thực hiện cơ chế này. Có 3 cách để thực hiện nhiệm vụ này:
- Tasklets: Chạy trong atomic context
- Workqueues: Chạy trong process context 
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

Về cơ bản thì tasklet là không reentrant. Một đoạn code được gọi là reentrant nếu nó có thể bị ngắt khi đang thực thi, rồi sau đó có thể được thực thi tiếp. Tasklet được thiết kế để có thể chạy đồng thời trên chỉ 1 CPU (ngay cả trên SMP system), đó là CPU mà nó được lập lịch, những tasklet khác có thể chạy đồng thời trên những CPUs khác. Những API của tasklet khá đơn giản:
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

Chỉ có một điểm khác biệt giữa hai hàm này. Hàm đầu tạo một tasklet đã được enable và sẵn sàng để được lập lịch mà không cần gọi hàm nào nữa (bằng cách set trường count = 0), trong khi hàm sau thì tạo một tasklet bị disable (bằng cách set trường count = 1), do đó sau hàm này phải gọi tiếp hàm tasklet_enable() trước khi tasklet có thể lập lịch:
```c
#define DECLARE_TASKLET(name, func, data) \
struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(0), func, data }

#define DECLARE_TASKLET_DISABLED(name, func, data) \
struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(1), func, data }
```

### 2.1.2 Bật và vô hiệu hóa tasklet



















