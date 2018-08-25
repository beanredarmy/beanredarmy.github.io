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
![](/img/plug_and_play-sysfs.png)

## 3. Những structure cơ bản trong Linux Device.

Linux Device Model cung cấp một số những structure để đảm bảo tương tác giữa thiết bị hardware device một device driver. Tất cả model đều được dựa trên kobject structure. Với structure này, một số những structure khác đã được implemented sẵn:
  - struct bus_type
  - struct device 
  - struct device_driver

  ![](/img/plug_and_play-linux_device_model_structures.png)

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

### b. Xe Buýt

 Bus là một kênh giao tiếp giữa bộ xử lý và thiết bị vào ra. Để đảm bảo mô hình là nhất quán thì tất cả thiết bị vào ra phải được kết nối với bộ xử lý bằng những bus như thế (thậm chí bus này có thể là bus ảo, không tương ứng với bus vật lý nào cả)

 Khi thêm vào một bus hệ thống thì tương ứng sẽ có file xuất hiện trong /sys/bus. Giống như các đối tượng kobject, bus có thể được tổ chức thành các hệ thống thứ bậc và hiện diện ngay trong sysfs.

 Trong Linux Device Model, một bus được đại diện bởi [struct bus_type](http://elixir.free-electrons.com/linux/v4.9/source/include/linux/device.h#L109):

 ```c
struct bus_type {
        const char              *name;
        const char              *dev_name;
        struct device           *dev_root;
        struct bus_attribute    *bus_attrs;
        struct device_attribute *dev_attrs;
        struct driver_attribute *drv_attrs;
        struct subsys_private *p;
 
        int             (*match)(struct device * dev, struct device_driver * drv);
        int             (*uevent)(struct device *dev, struct kobj_uevent_env *env);
        int             (*probe)(struct device * dev);
        int             (*remove)(struct device * dev);
        //...
};
```

Chú ý răng bus luôn được liên kết với 1 name, các danh sách thuộc tính mặc định, một số các hàm cụ thể và private data của driver.  Hàm _uevent_ được dùng cho các hotplug device.

Đăng kí và hủy đăng kí một bus được thực hiện bởi các hàm [bus_register](http://elixir.free-electrons.com/linux/v4.9/source/drivers/base/bus.c#L879) và bus_unregister[http://elixir.free-electrons.com/linux/v4.9/source/drivers/base/bus.c#L965].
```c
#include <linux/device.h>
/* mybus.c */
 
//bus type
struct bus_type my_bus_type = {
	.name	= "mybus",
	.match	= my_match,
	.uevent	= my_uevent,
};
 
static int __init my_bus_init (void)
{
	int err;
 
	//...
	err = bus_register (&my_bus_type);
	if (err)
		return err;
	//...
}
 
static void __exit my_bus_exit (void)
{
	//...
	bus_unregister (&my_bus_type);
	//...
}


```

Ví dụ sau là các hàm được implement:

 ```c
#include<linux/device.h>
#include<linux/string.h>

/* match devices to drivers;  Just do a simple name test */
static int my_match (structure device *dev, struct device_driver *driver)
{
   return !strncmp(dev_name(dev), driver->name, strlen(driver->name)) ;
}

/*  respond to hotplug user events;  Add environment variable DEV_NAME */
static int my_uevent(struct device *dev, struct kobj_uevent_env *env)
{
   add_uevent_var(env, "DEV_NAME =% s", dev_name(dev));
   return 0 ;
}
```
Hàm match được sử dụng khi một thiết bị mới hoặc driver mới được thêm vào bus. Vai trò của nó là so sánh giữa ID của device và driver. Hàm uvent thì được gọi trước khi tạo ra một hotplug trên user-space và có vai trò tạo ra các biến môi trường tương ứng.

Một hàm khác trên bus đó là xem được những driver và device nào đang được gắn vào bus. Mặc dù ta không thể truy cập trực tiếp chúng ( danh sách những driver và device được lưu vào private data của driver, trường  [subsys_private * p](http://elixir.free-electrons.com/linux/v4.9/source/include/linux/device.h#L134) ), nhưng chúng có thể được scan ra bằng cách sử dụng các macro [bus_for_each_dev](http://elixir.free-electrons.com/linux/v4.9/source/drivers/base/bus.c#L281) và [bus_for_each_drv](http://elixir.free-electrons.com/linux/v4.9/source/drivers/base/bus.c#L431).

Linux Device Model cho phép ta có thể tạo những thuộc tính để liên kết với object. Những thuộc tính này sẽ có các file tương ứng trong các thư mục con của sysfs bus. Những thuộc tính liên kết với bus được mô tả bằng struct bus_attribute:

```c
  struct bus_attribute {
         struct attribute        attr;
         ssize_t (*show)(struct bus_type *, char * buf);
         ssize_t (*store)(struct bus_type *, const char * buf, size_t count);
};
```
Thường thì một thuộc tính sẽ được define bởi macro BUS_ATTR. Để thêm/xóa thuộc tính trong bus structure thì dùng hàm bus_create_file và bus_remove_file nhé.

Một ví dụ cho việc define một thuộc tính cho my_bus

``c
/* mybus.c */
 
#define MY_BUS_DESCR     "SO2 rules forever"
 
// export a simple bus attribute
static ssize_t my_show_bus_descr (struct bus_type *bus, char *buf)
{
        return snprintf (buf, PAGE_SIZE, "%s\n", MY_BUS_DESCR);
}
 
/*
 * define attribute - attribute name is descr;
 * full name is bus_attr_descr;
 * sysfs entry should be /sys/bus/mybus/descr
 */
BUS_ATTR (descr, 0444, my_show_bus_descr , NULL);
 
// specify attribute - in module init function
static int __init my_bus_init (void)
{
        int err;
        //...
        err = bus_create_file (&my_bus_type, &bus_attr_descr);
        if(err) {
                /* handle error */
        }
        //...
}
 
static void __exit my_bus_exit (void)
{
        //...
        bus_remove_file (&my_bus_type, &bus_attr_descr);
        //...
}
```
Bus có thể cjd đại diện bởi cả bus_type object hoặc là một device object, vì thực ra bus cũng là một thiết bị.

### c. Devices

Bất cứ một thiết bị nào trong hệ thống đều có một struct liên kết với nó. Các thiết bị có thể được thấy bởi một số các phương pháp của kernel (hotplug, device driver, system initialization) và sau đó được ghi nhận vào hệ thống. Tất các device trong kernel đều có entry /sys/devices.

Ở mức bottom level, một thiết bị trong Linux Device Model là một struct:
```c
struct device {
        //...
        struct device           *parent;
        struct device_private   *p;
        struct kobject          kobj;
 
        const char              *init_name; /* initial name of the device */
        //...
        struct bus_type         *bus;       /* type of bus device is on */
        struct device_driver    *driver;    /* which driver has allocated this device */
        //...
        void    (*release)(struct device * dev);
};
```
Trong các trường của struct có: một parent (thường là một controller), một đối tượng kobject, bus mà nó gắn vào, device driver, và một hàm được gọi khi device counter đạt bằng 0.

Như thông thường, chúng ta có các hàm đăng kí/hủy đăng ki device là device_register và deivce_unregister.

Để làm việc với các thuộc tính ta có struct device_attribute, macro DEVICE_ATTR để define, các hàm device_create_file và device_remove_file để thêm hoặc loại bỏ thuộc tính.

Một điều quan trọng là chúng ta thường không làm việc trực tiếp với struct device mà làm việc với struct chứa nó, như là:
```c
  // my device type
struct my_device {
	char *name;
	struct my_driver *driver;
	struct device dev;
};
```
Xem ví dụ:
```c
/* mybus.c */
 
/* BUS DEVICE (parent) */
 
// parent device release
static void my_bus_device_release (struct device *dev)
{
}
 
// parent device
static struct device my_bus_device = {
	.init_name   = "mybus0",
	.release  = my_bus_device_release
};
 
/* DEVICE */
 
/*
 * as we are not using the reference count, we use a no-op
 * release function
 */
static void my_dev_release (struct device *dev)
{
}
 
int my_register_device (struct my_device *mydev)
{
	mydev->dev.bus = &my_bus_type;
	mydev->dev.parent = &my_bus_device;
	mydev->dev.release = my_dev_release;
	dev_set_name(&mydev->dev, mydev->name);
 
	return device_register (&mydev->dev);
}
 
void my_unregister_device (struct my_device * mydev)
{
	device_unregister (&mydev->dev);
}
 
/* export register/unregister device functions */
EXPORT_SYMBOL (my_register_device);
EXPORT_SYMBOL (my_unregister_device);
```
Như bạn có thể thấy, các hàm my_register_device và my_unregister_device dùng để thêm hoặc xóa thiết bị khỏi một bus được định nghĩa trong file nơi bus được xác định. Không khởi tạo các đối tượng device; chúng sẽ được khởi tạo khi chúng được phát hiện trong hệ thống (thông qua hotplug hoặc trực tiếp từ trình điều khiển) và gọi my_register_device để thêm vào bus.

Để sử dụng  một cấu trúc được export, khởi tạo và đăng ký với phương thức export của bus phải được khai báo: 

```c
/* mydriver.c */
 
static struct my_device mydev;
char devname[NAME_SIZE];
//...
 
//register
int err;
 
sprintf(devname, "mydev0");
mydev.name = devname;
mydev.driver = &mydriver;
dev_set_drvdata(&mydev.dev, &mydev);
err = my_register_device(&mydev);
if (err < 0) {
	/*handle error */
}
 
//..
 
//unregister
my_unregister_device(&mydev);
```

### d. Driver

Linux Device Model cho phép dễ dàng liên kết giữa system device và driver. Driver có thể export thông tin mà không phụ thuộc vào một thiết bị vật lý.

Trong sysfs, thông tin về driver không được liên kết bởi một thư mục con nào mà chúng nằm trong các thư mục khác nhau: trong /sys/module với các module đã được load, trong devices có thể tìm thấy các driver liên kết với mỗi device, trong những class thuộc driver, trong /sys/bus mà driver liên kết.

Một device driver được xác định bởi cấu trúc struct device_driver:
```c
struct device_driver {
         const char              *name;
         struct bus_type         *bus;
 
         struct driver_private   *p;
 
         struct module           *owner;
         const char              *mod_name;     /* used for built-in modules */
 
         int     (*probe)        (struct device * dev);
         int     (*remove)       (struct device * dev);
         void    (*shutdown)     (struct device * dev);
         int     (*suspend)      (struct device * dev, pm_message_t state);
         int     (*resume)       (struct device * dev);
};
```
Giữa những trường của struct, a thấy được tên của driver (tên này sẽ xuất hiện trong sysfs), bus mà dirver làm việc, và các hàm được gọi...

Chúng ta cũng có các hàm đăng kí và hủy đăng kí driver tương ứng là driver_register và driver_unregister.

Để làm việc với các thuộc tính, ta có struct driver_attribute, macro để định nghĩa thuộc tính DRIVER_ATTR, các hàm driver_create_file và driver_remove_file để thêm hoặc xóa thuộc tính.

Giống như với device, device_driver thuông thường được đăng vào một struct chi tiết hơn:
```c
/* mybus.c */
 
// my driver type
 struct my_driver {
	struct module *module;
	struct device_driver driver;
};
 
#define to_my_driver(drv) container_of(drv, struct my_driver, driver);
 
int my_register_driver (struct my_driver *driver)
{
	int err;
 
	driver->driver.bus = &my_bus_type;
	err= driver_register (&driver->driver);
	if (err)
		return err;
	return 0;
}
 
void my_unregister_driver (struct my_driver *driver)
{
	driver_unregister (&driver->driver);
}
 
/* export register/unregister driver functions */
EXPORT_SYMBOL (my_register_driver);
EXPORT_SYMBOL (my_unregister_driver);
```
Điều đáng lưu ý là các hoạt động đăng ký / đăng ký driver được export để sử dụng trong các mô-đun khác.

Như với các device, các hoạt động của driver được định nghĩa lúc khởi tạo bus và được export ra để được sử dụng bởi các driver khác. Khi một driver làm việc với các device gắn liền với bus này được triển khai, nó sẽ gọi my_register_driver và my_unregister_driver để kết hợp với nó.

Để sử dụng một cấu trúc được export, khởi tạo và đăng ký với phương thức export của bus phải được khai báo: 
```c
/* mydriver.c */
 
static struct my_driver mydriver = {
	.module = THIS_MODULE,
	.driver = {
		.name = "mydriver",
	},
};
//...
 
//register
int err;
err = my_register_driver(&mydriver);
if (err < 0) {
	/*handle error */
}
//..
 
//unregister
my_unregister_driver(&mydriver);
```