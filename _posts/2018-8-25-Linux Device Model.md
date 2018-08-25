---
layout: post
title: Linux Device Model
subtitle:  Một cái nhìn tổng quát về mô hình thiết bị trong hệ thống Linux
gh-repo: 
gh-badge: [star, fork, follow]
tags: [kernel, embedded]
---

_Tham khảo từ linux-kernel-labs của Linux_


## 1. Tổng quan

Plug-and-Play (PnP) có nghĩa là “cắm và chạy”. Nói nôm na, đây là một tính năng thông minh, giúp máy tính tự động nhận diện thiết bị và nạp driver cho bạn sử dụng ngay, hễ gắn vào máy tính là thiết bị có thể chạy ngay tắp lự chẳng cần mất công setup phức tạp. Công nghệ này làm giảm xung đột tài nguyên mà chúng sử dụng bằng cách config tự động lúc khởi động hệ thống. Để đạt được mục tiêu này thì yêu cầu những đặc điểm sau:
  - Tự động phát hiện việc add/remove thiết bị vào hệ thống (thiết bị và các đường bus của nó phải khai báo driver phù hợp khi có sự thay đổi về config).
  - Quản lý tài nguyên (địa chỉ, irq lines, DMA channel, vùng nhớ) bao gồm phân bố tài nguyên và giải quyết xung đột.
  - Thiết bị phải cho phép phần mềm config (các tài nguyên thiết bị như - ports, interrupts, DMA - phải cho phép driver điều khiển).
  - Driver mà một thiết bị mới yêu cầu phải được nạp tự động bởi OS khi cần.
  - Khi thiết bị và đường bus của nó cho phép, hệ thống có thể add hoặc remove thiết bị khỏi hệ thống cho dù nó đang chạy mà không khoogn reboot hệ thống. (Hotplug)

Với một hệ thống hỗ trợ PnP, BIOS, OS và thiết bị phải hỗ trợ công nghệ này. Thiết bị thì phải có ID để cung cấp định danh cho driver, OS thì có thể nhận ra những thay đổi trong config.

Những thiết bị PnP phổ biến có thể kể đến là PCI devices (network cards), USB (keyboard, mouse, printer), ...

Trước phiên bản 2.6, kernel không có một model thống nhất để có thể lấy được thông tin từ chúng. Do đó, một mô hình cho thiết bị trên Linux, Linux Device Model, được phát triển.

Mục tiêu chính của mô hình là để duy trì những cấu trúc dữ liệu, cái mà phản ánh những tramkg thái
 và cấu trúc của hệ thống. Những thông tin như thế sẽ bao gồm những thiết bị ở trong hệ thống, thông tin về quản lý nguồn, bus mà thiết bị gắn vào, đang được driver nào điều khiển, bên cạnh những structure về bus, device, driver trong hệ thống.

Kernel sử dụng những đối tượng:

  - device : một thiết bị vật lý được gắn vào một bus.
  - driver : trình phần mềm liên kết với thiết bị và điều khiển thiết bị.
  - bus : đường bus mà những thiết bị khác gắn vào.
  - class: một lớp các thiết bị có chung đặc điểm. Chúng ta có những lớp như là discs, partition, serial port, ...
  - subsystem : bao gỗm thiết bị, đường bus, class,... VD: GPIO subsystem, SPI subsystem, ...

## 2. sysfs

Kernel thể hiện mô hình cho userspace thông qua hệ thống file sysfs. sysfs được mount ở thư mục /sys và chứa những thư mục con:

  - block : tất các những block device trong hệ thống (disk, partitions).
  - bus: các loại bus mà những thiết bị vật lý gắn lên (pci, ide, usb).
  - class : những driver class đang có trên hệ thống (net, sound, usb).
  - devices: cấu trúc có thứ bậc của những thiết bị kết nối với hệ thống.
  - firmware : thông tin từ firmware hệ thống (ACPI).
  - fs: thông tin về những hệ thống file được mount.
  - kernel: thông tin trạng thái của kernel (lodded-in users, hotplug).
  - modules: danh sách những module đang được load
  - power: thông tin đến vấn đề quản lý power.

Như có thể thấy thì có một sự tương đồng nhất định giữa những data structure trong nhưng model được mô tả với những thư mục con của sysfs nói trên. Mặc dù sự tương đồng có thể gây ra cho bạn sự nhầm lẫn về 2 concept, nhưng chúng thực sự khác biệt. Kernel device model có thể làm việc mà không có sysfs, nhưng điều ngược lại không đúng, có nghĩa là sysfs mô tả lại kernel device model cho userspace mà thôi.

Có một số  thông tin có thể tìm thấy trong những file ở sysfs. Một vài những thuộc tính chuẩn như:
  - dev : chứa Major và Minor của device. Được dùng để tự động tạo entry trong thư mục /dev
  - device : là symbolic link tới thư mục chứa các thiết bị, được dùng để mô tả những thiết bị hardware mà cung cấp các service cụ thể (VD ethi PCI card).
  - driver : là symbolic link tới thư mục drriver ( nằm ở /sys/bus/*/driver)

Còn một số những thuộc tính khác nữa dựa và bus và driver được sử dụng.

## 3. Những structure cơ bản trong Linux Device.

Linux Device Model cung cấp một số những structure để đảm bảo tương tác giữa thiết bị hardware device một device driver. Tất cả model đều được dựa trên kobject structure. Với structure này, một số những structure khác đã được implemented sẵn:
  - struct bus_type
  - struct device 
  - struct device_driver

### a. kobject structure
  kobject structre không thực hiện một chức năng đơn lẻ và độc lập mà nó thường được tích hợp vào một structure lớn hơn. Thực ra kobject hợp nhất những tập feature dùng cho những đối tượng có tính trừu tượng cao hơn trong hệ thống thứ bậc của Linux Device Model.

Ví dụ điển hình như struct cdev có chứa một đối tượng kobject:

```c
 struct cdev {
        struct kobject kobj ;
        struct module * owner ;
        const struct file_operations * ops ;
        struct list_head list ;
        dev_t dev ;
        unsigned int count ;
} ;
```
struct kobject thì bao gồm:

```c
 struct kobject {
        const char * name ;
        struct list_head entry ;
        struct kobject * parent ;
        struct kset * kset ;
        struct kobj_type * ktype ;
        struct sysfs_dirent * sd ;
        struct kref kref ;
        unsigned int state_initialized : 1 ;
        unsigned int state_in_sysfs : 1 ;
        unsigned int state_add_uevent_sent : 1 ;
        unsigned int state_remove_uevent_sent : 1 ;
        unsigned int uevent_suppress : 1 ;
};
```
Ta có thể thấy thì struct kobject cũng nằm trong một hệ thống thứ bậc: một object có một parent và chứa một kset. kset là con trỏ trỏ tới những object có chung level.

Struct trên có thể khởi tạo với hàm kobject_init. Trong quá trình khởi tạo thì việc đặt tên cho kobject cũng là điều cần thiết vì nó sẽ xuất hiện trong sysfs. Việc đặt tên thì sẽ dùng hàm kobject_set_name.

Bất cứ một thao tác nào với kobject thì phải thực hiện tăng một biến đếm interal với kobject_get hoặc giảm biến này với kobject_put nếu không còn dùng đến kobject nữa. Do đó, một đối tượng kobject chỉ được giải phóng khi biến đếm internal này bằng 0. Tài nguyên mà trước đó liên kết với device struct đã được giải phóng phải bao gồm kobject struct (vd struct cdev). Phương pháp này gọi là release và nó liên kết với những object bằng trường ktype (struct kobj_type).

struct kobject là một trong những cấu trúc cơ bản của Linux Device Model. Những struct ở mức cao hơn là bus_type, device, device_driver. 




