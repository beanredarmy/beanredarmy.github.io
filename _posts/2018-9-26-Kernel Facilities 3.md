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

