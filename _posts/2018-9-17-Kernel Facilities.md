---
layout: post
title: Kernel Facilities and Helper Functions (Part 1)  
subtitle:  Những công cụ và hàm hay dùng trong kernel (Phần 1)
gh-repo: 
gh-badge: [star, fork, follow]
tags: [kernel, embedded, device driver]
---

_Tham khảo từ Linux Device Drivers Development_

Bản thân kernel là một phần tách biệt với những software ở userspace. Chúng không sử dụng một thư viện C nào cả. Vậy chúng sử dụng những gì? Chúng ta sẽ đi từng bước để tìm hiểu. Trong bài viết này, ta sẽ tìm hiểu một số vấn đề:
  - Giới thiệu về container data structure.
  - Xem cơ chế sleeping của kernel.
  - Cách sử dụng timers.
  - Cơ chế lock của kernel (Mutex, spinlock)
  - Sử dụng IRQs

# 1. Hiểu về container_of macro.
  Đây là một trong những macro được sử dụng nhiều nhất khi lập trình kernel cũng như device driver. Khi nhắc đến việc quản lý những data structure trong code, hầu như ta luôn có nhu cầu là đưa 1 struct A vào 1 struct B khác, rồi truy ra được con trỏ của struct B khi đã biết được struct A. Lấy một ví dụ đơn giản về ```struct person```:
 
  ```c
  struct person {
    int age;
    char *name;
  } p;
  ```
  Tức là giả sử bạn chỉ có được con trỏ của trường age hoặc name, làm sao bạn có thể lấy được con trỏ struct chứa 2 trường này, tức là con trỏ p? Macro ```container_of``` sẽ giúp ta làm việc này, đúng như cái tên của nó. Phân tích macro này một chút, nó được định nghĩa ở ```include/linux/kernel.h```:
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
  - ```pointer```: Con trỏ đến trường được cho ở trong struct
  - ```container_type```: Type của struct chứa pointer
  - ```container_field```: Tên của trường mà pointer trỏ ở trong struct
  
  Với ví dụ ```struct person```, khi cho trước một instance:
  ```c
  struct person somebody;
  [...]
  char *the_name_pointer = somebody.name;
  ```

  Ta có thể lấy được con trỏ của trường name một cách dễ dàng như trên. Và giờ ta muốn lấy được con trỏ của struct (chính là con trỏ của ```struct somebody```), làm như thế này:

  ```c
  struct person *somebody_pointer;
  somebody_pointer = container_of(the_name_pointer, struct person, name);
  ```
  Hàm ```container_of``` hoạt động như thế nào? Có thể nhìn ở định nghĩa hàm, nó lấy offset của ```name``` so với ```struct person``` để xác định được vị trí con trỏ. 

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
# 2. Linked lists
  Thử tưởng tưởng là ta có một driver điều khiển nhiều device, giả sử là 5 devices. Ta cần phải có một cách để track được các device. Hmm, và người ta sử dụng linked list (danh sách liên kết).
  Có 2 linked list được sử dụng nhiều nhất là:
   - Danh sách liên kết đơn
   - Danh sách liên kết đôi

  Developer chỉ sử dụng danh sách liên kết đôi vòng khi cần cấu trúc kiểu FIFO và LIFO. 
  Để sử dụng linked list thì cần có header <linux/list.h>.

  Struct hay được thấy nhất trong kernel chính là ```struct list_head```:
  ```c
  struct list_head {
    struct list_head *next, *prev;
  }
  ```
  ```struct list_head``` được dùng trong cả phần tử đầu danh sách hoặc các node. Trong kernel, để biểu diễn được một cấu trúc dữ liệu theo kiểu linked list, cần phải nhúng ```struct list_head``` vào. Ví dụ:
  ```c
  struct car {
    int door_number;
    char *color;
    char *model;
  };
  ```
  Trước khi tạo một danh sách car, ta cần cho ```struct list_head``` vào:
  ```c
  struct car {
    int door_number;
    char *color;
    char *model;
    struct list_head list;
  };
  ```  
  Sau đó ta cần tạo một biến ```struct list_head``` để trỏ vào phần tử đầu. Phần tử đầu này không liên kết với cái ô tô nào cả, nó hơi đặc biệt một xíu:

  ```c
  static LIST_HEAD(carlist);
  ```
  Giờ ta có thể tạo những chiếc xe khác và đưa nó vào danh sách carlist:

  ```c
  #include <linux/list.h>

  struct car *redcar = kmalloc(sizeof(*car, GFP_KERNEL));
  struct car *bluecar = kmalloc(sizeof(*car), GFP_KERNEL));

  /* Initialize each node's list entry */
  INIT_LIST_HEAD(&bluecar->list);
  INIT_LIST_HEAD(&redcar->list);

  /* allocate memory for color and model field and fill every filed */
  [...]
  list_add(&redcar->list,&carlist);
  list_add(&bluecar->list,&carlist);
  ```
  Và giờ carlist đã có 2 phần tử. Giờ ta sẽ đi sâu vào những API ở trên.

## 2.1. Tạo và khởi tạo list
Có 2 cách để tạo và khởi tạo:
### 2.1.1. Dynamic method

  Phương pháp khởi tạo động này bao gồm một ```struct list_head``` cùng với macro ```INIT_LIST_HEAD```:
  ```c
  struct list_head mylist;
  INIT_LIST_HEAD(&mylist);
  ```
  trong đó:
  ```c
  static inline void INIT_LIST_HEAD(struct list_head *list)
  {
    list->next = list;
    list->prev = list;
  }
  ```
### 2.1.2. Static method

  Phương pháp khởi tạo tĩnh sử dụng macro ```LIST_HEAD```:
  ```c
  LIST_HEAD(mylist);
  ```
  trong đó:
  ```c
  #define LIST_HEAD(name) \
      struct list_head name = LIST_HEAD_INIT(name)
  ```
  và 
  ```c
  #define LIST_HEAD_INIT(name) { &(name), &(name) }
  ```
  macro sẽ gán các con trỏ prev và view trong name bằng chính con trỏ name.
## 2.2. Tạo node
  Để tạo một node mới, chỉ cần tạo một đối tượng của cấu trúc dữ liệu rồi nhúng ```list_head``` vào trong đó. Sử dụng lại ví dụ oto, đầu tiên ta cấp phát 
  ```c
  struct car *blackcar = kzalloc(sizeof(struct car), GFP_KERNEL);
  /* non static initialization, since it is the embedded list field*/
  INIT_LIST_HEAD(&blackcar->list);
  ```
  Như đã nói thì ```INIT_LIST_HEAD``` sẽ cấp phát động cho list.
## 2.3. Thêm node vào danh sách
  Kernel cung cấp hàm ```list_add``` để thêm một node vào danh sách, hàm này thêm phần tử new vào ngay sau head:
  ```c
  void list_add(struct list_head *new, struct list_head *head);
  static inline void list_add(struct list_head *new, struct list_head *head)
  {
    __list_add(new, head, head->next);
  }
  ```
  trong đó:
  ```c
  static inline void __list_add(struct list_head *new,
              struct list_head *prev,
              struct list_head *next)
  {
    next->prev = new;
    new->next = next;
    new->prev = prev;
    prev->next = new;
  }
  ```
  Ví dụ đã đưa thêm vào list 2 chiếc xe:
  ```c
    list_add(&redcar->list, &carlist);
    list_add(&blue->list, &carlist);
  ```
  Cách này sẽ implement một stack.

  Một cách khác để thêm node đó là dùng hàm sau để thêm node vào cuối danh sách:
  ```c
  void list_add_tail(struct list_head *new, struct list_head *head);
  ```
  Hàm trên được dùng để implement một queue.
## 2.4. Xóa một node khỏi list
  Có thể sử dụng hàm sau:
  ```c
  void list_del(struct list_head *entry);
  ```
  ví dụ như:
  ```c
  list_del(&redcar->list);
  ```
  Chú ý. Hàm ```list_del``` chỉ đơn giản là ngắt node ra khỏi list rồi nối danh sách lại bằng các con trỏ ```prev``` và ```next```. Nó không hoàn toàn giải phóng node. Nếu cần giải phóng node thì phải sử dụng hàm ```kfree```.
## 2.5. Duyệt danh sách
  Chúng ta sử dụng macro ```list_for_each_entry(pos, head, member)``` để duyệt danh sách:
   - ```head``` là phần tử đầu danh sách
   - ```member``` là tên của ```struct list_head``` (với trường hợp của chúng ta là list)
   - ```pos``` là phần tử chỉ số dùng để duyệt bằng vòng lặp. (Giống như i ở trong for(int i = 0, i < 100, i++)).
   
  Dùng ví dụ sau cho dễ hiểu:
  ```c
  struct car *acar; /* loop counter */
  int blue_car_num = 0;
  /* 'list' is the name of the list_head struct in our data structure */
  list_for_each_entry(acar, carlist, list){
    if(acar->color == "blue")
    blue_car_num++;
  }
  ```
  Còn một điều nữa, tạo sao chúng ta lại cần ```list_head``` ở trong cấu trúc dữ liệu? Hãy xem định nghĩa hàm ```list_for_each_entry```:
  ```c
  #define list_for_each_entry(pos, head, member)  \
  for (pos = list_entry((head)->next, typeof(*pos), member);  \
        &pos->member != (head);  \
        pos = list_entry(pos->member.next, typeof(*pos), member));

  #define list_entry(ptr, type, member) \
    container_of(ptr, type, member)
  ```
  Vâng, và đây chúng ta hiểu được sức mạnh của hàm ```container_of```. :))))
