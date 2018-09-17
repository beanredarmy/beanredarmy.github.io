---
layout: post
title: Kernel Facilities and Helper Functions
subtitle:  Những công cụ và hàm hay dùng trong kernel
gh-repo: 
gh-badge: [star, fork, follow]
tags: [kernel, embedded]
---

_Tham khảo từ Linux Device Drivers Development_

Bản thân kernel là một phần tách biệt với những software ở userspace. Chúng không sử dụng một thư viện C nào cả. Vậy chúng sử dụng những gì? Chúng ta sẽ đi từng bước để tìm hiểu. Trong bài viết này, ta sẽ tìm hiểu một số vấn đề:
  - Giới thiệu về container data structure.
  - Xem cơ chế sleeping của kernel.
  - Cách sử dụng timers.
  - Cơ chế lock của kernel (Mutex, spinlock)
  - Một số công việc sử dụng API của kernel
  - Sử dụng IRQs

## 1. Hiểu về container_of macro.
  Đây là một trong những macro được sử dụng nhiều nhất khi lập trình kernel cũng như device driver. Khi nhắc đến việc quản lý những data structure trong code, hầu như ta luôn có nhu cầu là đưa 1 struct A vào 1 struct B khác, rồi truy ra được con trỏ của struct B khi đã biết được struct A. Lấy một ví dụ đơn giản về struct person:
 
  ```c
  struct person {
    int age;
    char *name;
  } p;
  ```
  Tức là giả sử bạn chỉ có được con trỏ của trường age hoặc name, làm sao bạn có thể lấy được con trỏ struct chứa 2 trường này, tức là con trỏ p? Macro container_of sẽ giúp ta làm việc này, đúng như cái tên của nó. Phân tích macro này một chút, nó được định nghĩa ở include/linux/kernel.h:
  ```c
  #define container_of(ptr, type, member) ({
  \
  const typeof(((type *)0)->member) * __mptr = (ptr);
  \
  (type *)((char *)__mptr - offsetof(type, member)); })
  ```
  Nhìn sợ thật, nhưng thôi hiểu định nghĩa hàm đơn giản là:
  ```c
  container_of(pointer, container_type, container_field);
  ```
  Trong đó:
    - pointer: Con trỏ đến trường được cho ở trong struct
    - container_type: Type của struct chứa pointer
    - container_field: Tên của trường mà pointer trỏ ở trong struct
  
  Với ví dụ struct person, khi cho trước một instance:
  ```c
  struct person somebody;
  [...]
  char *the_name_pointer = somebody.name;
  ```
  Ta có thể lấy được con trỏ của trường name một cách dễ dàng như trên. Và giờ ta muốn lấy được con trỏ của struct (chính là con trỏ của struct somebody), làm như thế này:
  ```c
  struct person *somebody_pointer;
  somebody_pointer = container_of(the_name_pointer, struct person, name);
  ```c
  Hàm container_of hoạt động như thế nào? CÓ thể nhìn ở định nghĩa hàm, nó lấy offset của name so với struct person để xác định được vị trí con trỏ. 

  Một ví dụ cụ thể hơn là:
  ```c
  struct family {
    struct person *father;
    struct person *mother;
    int number_of_suns;
    int salary;
  } f;
  /*
  * pointer to a field of the structure
  * (could be any member of any family)
  */
  struct *person = family.father;
  struct family *fam_ptr;
  /* now let us retrieve back its family */
  fam_ptr = container_of(person, struct family, father);
  ```
 Những driver ngoài đời cũng sẽ sử dụng hàm trên như dưới đây:
  ```c
  struct mcp23016 {
    truct i2c_client *client;
    struct gpio_chip chip;
  }
  /* retrive the mcp23016 struct given a pointer 'chip' field */
  static inline struct mcp23016 *to_mcp23016(struct gpio_chip *gc)
  {
    return container_of(gc, struct mcp23016, chip);
  } 
  static int mcp23016_probe(struct i2c_client *client,
                  const struct i2c_device_id *id)
  {
    struct mcp23016 *mcp;
    [...]
    mcp = devm_kzalloc(&client->dev, sizeof(*mcp), GFP_KERNEL);
    if (!mcp)
      return -ENOMEM;
    [...]
  }
  ```
## 2. Linked lists