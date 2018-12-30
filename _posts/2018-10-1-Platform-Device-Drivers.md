---
layout: post
title: Linux Platform Device Driver
subtitle:  Linux Platform Device Driver
gh-repo: 
gh-badge: [star, fork, follow]
tags: [kernel, embedded, device driver]
---

## Linux Platform Device Driver

_Tham khảo từ Linux Device Drivers Development_
Chúng ta hầu như đều đã biết về các thiết bị plug and play (cắm vào là chạy), chúng được kiểm soát và điều khiển bởi kernel ngay khi cắm vào. Những ví dụ điển hình về các thiết bị plug and play là USB hoặc PCI express, đây là những thiết bị được tự động phát hiện (ví dụ như chúng ta có một cổng USB, sau khi cắm một thiết bị là chuột máy tính vào cổng USB này, kernel sẽ được báo về sự xuất hiện của thiết bị chuột, chứ không phải là của bàn phím hay gì khác). Còn các platform device lại không như vậy, không hotplug, chúng không thể đại loại như nói với kernel rằng "Hú! Bố mày đây nè", còn kernel cần được biết thiết bị platform nào sẽ được kết nối (bằng cách khai báo cho kernel, ví dụ EEPROM được kết nối vào I2C bus ở địa chỉ 0xDA). I2C, UART, SPI, ... là những thiết bị platform. 

Có những bus vật lý có thể bạn đã biết là: USB, I2S, I2C, UART, SPI, PCI, SATA,... Những bus như thế được gọi là controller. Do chúng là một thành phần của SoC nên không thể bỏ đi được, cũng không tự động phát hiện được, nên chúng cũng được gọi là platform device.

P/s: Có một số quan niệm cho rằng platform device là on-chip device (được hàn cứng trong SoC). Đúng! Nhưng chưa đủ! Những thiết bị được cắm vào I2C hoặc SPI bus đều không phải on-chip, nhưng chúng vẫn là platform device, đơn giản vì ta không thể thấy vàliệt kê những chip nào đang nằm trên bus mà không khai báo. Còn với thiết bị PCI, USB, chúng đều on-chip nhưng không phải là platform device, đơn giản vì chúng là có thể thấy và liệt kê được (discoverable & enumarable).

Bus USB là platform device, còn thiết bị cắm vào bus USB thì không! Ví dụ ta có ```drivers/usb/host/ohci-pnx4008.c```, đây là một USB host controller, và cũng là một platform device. Platform device này được đăng kí ở board file ```arch/arm/mach-pnx4008/core.c:pnx4008_init```. Và bên trong hàm probe (sẽ thảo luận sau), nó sẽ đăng kí nó với bus vật lý I2C rằng nó là một I2C device bằng hàm ```i2c_register_driver```. Như vậy chip USB Host controller sẽ giao tiếp với CPU của ta qua I2C bus, còn thiết bị USB thì lại giao tiếp với USB Host controller qua USB bus. 

Pseudo platform bus, thường được gọi tắt là platform bus, là một loại bus ảo của kernel, dành cho những device mà không nằm trên một bus vật lý nào mà kernel đã biết.

Làm việc với platform driver yêu cầu hai bước sau:
- Đăng kí một **platform driver** (với tên driver là duy nhất trong hệ thống) để quản lý device.
- Đăng kí **platform device** với cùng tên của driver để kernel biết device của mình đang được sử dụng.

Trong bài viết ta sẽ thảo luận về:
- Platform device cùng với driver của chúng.
- Cơ chế kết nối (matching) giữa device và driver trong kernel.
- Đăng kí platform driver, platform device, platform data.

### Platform driver
Trước hết cần chú ý rằng, platform driver dùng để điều khiển platform device, nhưng không phải tất cả các platform device đều được điều khiển bằng platform driver. Platform driver đơn giản là những driver điều khiển các thiết bị nằm trên pseudo platform bus (nghĩa là các đường bus không chính thống). VÍ dụ I2C device và SPI device, chúng là những platform device, và nằm trên I2C hoặc SPI bus(các đường bus chính thống - conventional bus), chứ không phải platform bus. 

Platform driver luôn phải triển khai hàm ```probe```, hàm này được gọi bởi kernel mỗi khi kernel module được thêm và device nhận được driver tương ứng. Khi lập trình với platform driver, ta phải điền vào một struct là  ```struct platform_driver```, và đăng kí driver.
```c
static struct platform_driver mypdrv = {
	.probe		= my_pdrv_probe,
	.remove		= my_pdrv_remove,
	.driver		= {
		.name	= "my_platform_driver",
		.owner	= THIS_MODULE,
	},
};
```
Các trường của struct trên bao gồm;
- ```probe()```: Hàm này được gọi sau khi match device với driver tương tứng:
```c
static int my_pdrv_probe(struct platform_device *pdev)
```

- ```remove()```: Hàm này được gọi để loại bỏ driver khi device không cần nữa;
```c
static int my_pdrv_remove(struct platform_device *pdev)
```

- ```struct device_driver```: struct mô tả driver, bao gồm tên, owner, một vài trường khác ...

Để đăng kí platform driver cho kernel thì ta chỉ cần gọi hàm ```platform_driver_register()``` hoặc hàm ```platform_driver_probe()``` trong hàm ```init```. Hai hàm này khác nhau ở chỗ:
- ```platform_driver_register()``` đăng kí và đặt driver vào một danh sách các driver duy trì bởi kernel. Khi đó hàm ```probe``` có thể được gọi mỗi khi việc match device và driver xảy ra. Ví dụ là có thể driver có thể xuất hiện sau khi boot kernel hoàn tất, khi nó xuất hiện và match với device thì sẽ gọi hàm ```probe```. Để ngăn driver khỏi phải đưa vào danh sách trên, ta sử dụng hàm dưới.
- ```platform_driver_probe()```, kernel lập tức chạy một vòng lặp để thử match device và driver bằng tên mà chúng đăng kí, nếu match thì gọi hàm ```probe```, đồng nghĩa với việc device đã xuất hiện. Nếu không match, thì driver bị bỏ qua. Hàm này ngăn việc hàm ```probe``` bị gọi về sau vì nó không đăng kí driver với hệ thống. Khi dùng hàm này, hàm ```probe``` được đặt trong vùng ```__init```, nghĩa là hàm ```probe``` sẽ được giải phóng khi quá trình boot kernel hoàn tất, qua đó hàm ```probe``` về sau sẽ không được gọi nữa đồng thời giảm bộ nhớ chiếm dụng của dirver. Và luôn nhớ dùng phương pháp sau để đảm bảo rằng device đã xuất hiện trong hệ thống:

```c
ret = platform_driver_probe(&mypdrv, my_pdrv_probe);
```

Ví dụ sau đăng kí một platform driver đơn giản với kernel:
```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/platform_device.h>

static int my_pdrv_probe (struct platform_device *pdev){ 
	pr_info("Hello! device probed!\n"); 
return 0; 
} 

static void my_pdrv_remove(struct platform_device *pdev){ 
	pr_info("good bye reader!\n"); 
} 

static struct platform_driver mypdrv = { 
	.probe = my_pdrv_probe, 
	.remove = my_pdrv_remove, 
	.driver = { 
		.name = KBUILD_MODNAME, 
		.owner = THIS_MODULE, 
	}, 
}; 

static int __init my_drv_init(void) 
{ 
	pr_info("Hello Guy\n"); 

	/* Registering with Kernel */ 
	platform_driver_register(&mypdrv); 
	return 0; 
} 

static void __exit my_pdrv_remove (void) 
{ 
	Pr_info("Good bye Guy\n"); 

	/* Unregistering from Kernel */ 
	platform_driver_unregister(&my_driver); 
} 

module_init(my_drv_init); 
module_exit(my_pdrv_remove); 
```

Module driver trên có hàm ```init``` và ```exit``` không làm gì ngoài việc đăng kí và hủy đăng kí driver với platform bus, và hầu hết các driver khác cũng như thế. Trong trường hợp này, ta có thể rút ngắn việc phải viết ```module_init``` và ```module_exit``` bằng cách sử dụng macro  ```module_platform_driver```.
Marco ```module_platform_driver``` được định nghĩa:
```c
/*
* module_platform_driver() - Helper macro for drivers that don't
* do anything special in module init/exit. This eliminates a lot
* of boilerplate. Each module may only use this macro once, and
* calling it replaces module_init() and module_exit()
*/
#define module_platform_driver(__platform_driver) \
module_driver(__platform_driver, platform_driver_register, \
platform_driver_unregister)
```

Với macro này, ta sẽ không cần các macro ```module_init``` và ```module_exit```, cũng như cũng không cần viết các hàm ```init``` và ```exit``` nữa. 

Chú ý. Hàm ```probe``` không phải là hàm thay thế của ```init```. Hàm ```probe``` được gọi mỗi lần match giữa device và driver, trong khi hàm ```init``` chỉ chạy một lần khi kernel module được load.

Ví dụ khi sử dụng macro ```module_platform_driver```:
```c
[...]
static int my_driver_probe (struct platform_device *pdev){
	[...]
}

static void my_driver_remove(struct platform_device *pdev){
	[...]
}

static struct platform_driver my_driver = {
	[...]
};
module_platform_driver(my_driver);
```

Một số macro tương tự sau cũng được dùng để đăng kí driver với các loại bus.
- ```module_platform_driver(struct platform_driver)``` cho platform driver để đăng kí chúng trên các đường bus không phải là chính thống
- ```module_spi_driver(struct spi_driver)``` cho SPI drivers
- ```module_i2c_driver(struct i2c_driver)``` cho I2C drivers
- ```module_pci_driver(struct pci_driver)``` cho PCI drivers
- ```module_usb_driver(struct usb_driver)``` cho USB drivers
- ```module_mdio_driver(struct mdio_driver)``` cho mdio
- ...

Nếu ta không biết loại bus cụ thể của driver, thì driver của ta chính là platform.

### Platform device
Platform device là các device nằm trên platform bus. Cũng giống như những gì ta làm với driver, ta phải báo cho kernel biết rằng device đang cần một driver. Một platform device được biểu diễn bằng struct ```platform_device```:
```c
struct platform_device {
	const char *name;
	u32 id;
	struct device dev;
	u32 num_resources;
	struct resource *resource;
};
```

Để driver và device có thể match với nhau, trường ```name``` của ```struct platform_device``` và ```static struct platform_driver.driver.name``` phải giống nhau. 

#### Resource và platform data.
Không giống như những thiết bị hot-plug, kernel sẽ không biết gì về những platform device nếu chúng không được khai báo. Có 2 phương pháp để thông báo cho kernel về những tài nguyên (irq, dma, memory region, I/O port, bus) và dữ liệu (những dữ liệu custom và private mà ta muốn truyền vào driver):

##### Cách 1. Phương pháp cũ và không khuyên dùng nữa.
Phương pháp này được sử dụng với những phiên bản kernel không hỗ trợ device tree. Với cách này, device được đăng kí bằng một source file code c.
**Resources**

Các tài nguyên (resources) là những thành phần đặc trưng phần cứng của device, những thứ device cần để được cài đặt và hoạt động đúng. Chỉ có 6 loại tài nguyên trong kernel (được định nghĩa trong ```include/linux/ioport.h```) :
```c
#define IORESOURCE_IO 0x00000100 /* PCI/ISA I/O ports */ 
#define IORESOURCE_MEM 0x00000200 /* Memory regions */ 
#define IORESOURCE_REG 0x00000300 /* Register offsets */ 
#define IORESOURCE_IRQ 0x00000400 /* IRQ line */ 
#define IORESOURCE_DMA 0x00000800 /* DMA channels */ 
#define IORESOURCE_BUS 0x00001000 /* Bus */
```

Các tài nguyên được đại diện bằng ```struct resource```:
```c
struct resource {
	resource_size_t start;
	resource_size_t end;
	const char *name;
	unsigned long flags;
};
```

Trong đó:
- ```start/end```: là những điểm đầu và cuối của tài nguyên. Ví dụ với I/O hoặc memory regions, nó chỉ ra nơi các tài nguyên bắt đầu và kết thúc. Với IRQ line, bus hoặc DMA channel, ```start/end ```phải có cùng giá trị.
- ```flag```: đây là mặt nạ bit đặc trưng cho kiểu tài nguyên, ví dụ như ```IORESOURCE_BUS```.
- ```name```: định danh và mô tả tài nguyên.

Khi ta đã cung cáp các tài nguyên của device cho kernel biết, việc ở phía bên viết driver là lấy những tài nguyên này ra để làm việc. Hàm ```probe``` sẽ đảm nhiệm công việc này. 
```c
int probe(struct platform_device *pdev);
```

Tham số ```pdev``` được điền tự động bởi kernel, với những dữ liệu và tài nguyên ta đã đăng kí trước đó.
```struct resource``` trong ```struct platform_device``` có thể được lấy bằng hàm ```platform_get_resource()```. Hàm có nguyên mẫu như sau:
```c
struct resource *platform_get_resource(struct platform_device *dev, unsigned int type, unsigned int num);
```

Tham số thứ nhất là ```platform_device```, tham số thứ 2 loại tài nguyên cần lấy. Ví dụ với memory thì ta điền ```IORESOURCE_MEM```. Tham số ```num``` là chỉ số của tài nguyên mong muốn trong mảng các tài nguyên. 
Nếu tài nguyên là một IRQ thì ta phải sử dụng hàm 
```c
int platform_get_irq(struct platform_device * pdev, unsigned intnum).
```

Ví dụ:
```c
static int my_driver_probe(struct platform_device *pdev) 
{ 
	struct my_gpios *my_gpio_pdata = 
	(struct my_gpios*)dev_get_platdata(&pdev->dev); 

	int rgpio = my_gpio_pdata->reset_gpio; 
	int lgpio = my_gpio_pdata->led_gpio; 

	struct resource *res1, *res2; 
	void *reg1, *reg2; 
	int irqnum; 

	res1 = platform_get_resource(pdev, IORESSOURCE_MEM, 0); 
	if((!res1)){ 
		pr_err(" First Resource not available"); 
		return -1; 
	} 
	res2 = platform_get_resource(pdev, IORESSOURCE_MEM, 1); 
	if((!res2)){ 
		pr_err(" Second Resource not available"); 
		return -1; 
	} 

	/* extract the irq */ 
	irqnum = platform_get_irq(pdev, 0); 
	Pr_info("\n IRQ number of Device: %d\n", irqnum); 

	/* 
	* At this step, we can use gpio_request, on gpio, 
	* request_irq on irqnum and ioremap() on reg1 and reg2. 
	* ioremap() is discussed in chapter 11, Kernel Memory Management 
	*/ 
	[...] 
	return 0; 
}
```

**Platform data**

Những kiểu dữ liệu mà không nằm trong các kiểu tài nguyên mà kernel liệt kê sẵn sẽ gọi là platform data. ```struct platform_device``` chứa trường ```struct device```, trường này lại chứa một trường struct khác là ```platform_data```. 
```platform_device.device.platform_data``` chính là nơi mà ta chứa dữ liệu. Ví dụ, ta cần khai báo một platform device cần có 2 chân GPIO là ```platform_data```, một irq number, 2 vùng nhớ xem như tài nguyên:
```c
/* 
* Other data than irq or memory must be embedded in a structure 
* and passed to "platform_device.device.platform_data" 
*/ 
struct my_gpios { 
	int reset_gpio; 
	int led_gpio; 
}; 

/*our platform data*/ 
static struct my_gpios needed_gpios = { 
	.reset_gpio = 47, 
	.led_gpio = 41, 
}; 

/* Our resource array */ 
static struct resource needed_resources[] = { 
	[0] = { /* The first memory region */ 
		.start = JZ4740_UDC_BASE_ADDR, 
		.end = JZ4740_UDC_BASE_ADDR + 0x10000 - 1, 
		.flags = IORESOURCE_MEM, 
		.name = "mem1", 
	}, 
	[1] = { 
		.start = JZ4740_UDC_BASE_ADDR2, 
		.end = JZ4740_UDC_BASE_ADDR2 + 0x10000 -1, 
		.flags = IORESOURCE_MEM, 
		.name = "mem2", 
	}, 
	[2] = { 
		.start = JZ4740_IRQ_UDC, 
		.end = JZ4740_IRQ_UDC, 
		.flags = IORESOURCE_IRQ, 
		.name = "mc", 
	}, 
}; 

static struct platform_devicemy_device = { 
	.name = "my-platform-device", 
	.id = 0, 
	.dev = { 
		.platform_data = &needed_gpios, 
	}, 
	.resource = needed_resources, 
	.num_resources = ARRY_SIZE(needed_resources), 
}; 
platform_device_register(&my_device);
```

Trong các ví dụ trước, ta đã sử dụng ```IORESOURCE_IRQ``` và ```IORESOURCE_MEM``` để thông báo cho kernel về loại tài nguyên ta đã cung cấp. Còn đối với platform data, ta chỉ cần lấy ```pdev->dev.platform_data```. Tuy nhiên, vẫn khuyên dùng hàm được kernel cung cấp (thực ra chúng hoạt động y hệt nhau, nhưng việc sử dụng hàm giúp code tường minh hơn):
```c
void *dev_get_platdata(const struct device *dev)
struct my_gpios *picked_gpios = dev_get_platdata(&pdev->dev);
```

**Khai báo platform device ở đâu?**

Các thiết bị được đăng kí cùng với tài nguyên và dữ liệu của nó. Ở phương pháp cũ này, ta sẽ khai báo một module riêng, hoặc trong file init của board ở ```arch/<arch>/mach-xxx/yyyy.c``` (ví dụ ```arch/arm/mach-imx/mach-imx6q.c```). Hàm ```platform_device_register()``` sẽ có nhiệm vụ đăng kí platform device với kernel.
```c
static struct platform_device my_device = { 
	.name = "my_drv_name", 
	.id = 0, 
	.dev.platform_data = &my_device_pdata, 
	.resource = jz4740_udc_resources, 
	.num_resources = ARRY_SIZE(jz4740_udc_resources), 
}; 
platform_device_register(&my_device);
```

Ta phải để ý sao cho tên của device phải trùng với tên của driver để kernel có thể match được chúng với nhau.

##### Cách 2. Cách mới và khuyên dùng

Với phương pháp đầu tiên, bất kì sự thay đổi về cấu hình device đều dẫn đến phải build lại cả kernel. Điều này thực sự bất tiện. Để đơn giản hơn, người ta tách việc khai báo device ra khỏi kernel source, và sử dụng device tree (DTS). Với device tree thì platform data và resource là như nhau. Device tree là một file mô tả phần cứng và có cấu trúc tương tự như một cây, mỗi device được đại diện bởi một node, và dữ liệu cũng như tài nguyên của thiết bị chính là thuộc tính của các node. Với phương pháp này, ta chỉ cần biên dịch lại mỗi device tree khi có thay đổi. Sẽ có một bài viết riêng về device tree.

### Match device và driver 
Để có thể matching, Linux gọi hàm ```platform_match(struct device *dev,
struct device_driver *drv)```. Trong mô hình Linux device, mỗi bus duy trì một danh sách các driver cũng như device đăng kí với nó. Và bus driver có trách nhiệm match device và driver lại với nhau. Mỗi lần ta kết nối một device mới hoặc thêm driver vào một bus, bus đó sẽ bắt đầu thực hiện vòng lặp để match. 

Giả sử ta đăng kí một thiết bị I2C và một I2C bus. Kernel sẽ thực hiện vòng lặp match trên I2C bus đó, bằng cách gọi hàm match I2C mà I2C bus driver đã đăng kí, rồi kiểm tra xem liệu đã có một driver nào cùng tên với thiết bị đó hay chưa. Nếu không match thì không có gì xảy ra. Nếu match, kernel sẽ thông báo (thông qua cơ chế giao tiếp gọi là netlink socket) cho trình quản lý thiết bị (```udev/mdev```), trình quản lý sẽ load module driver mà vừa match lên. Khi driver được load, hàm ```probe``` của nó được gọi để thực hiện việc lấy dữ liệu từ device,... Không chỉ I2C hoạt động với phương thức như vậy, các bus khác cũng có cơ chế match tương tự. Một vòng lặp kiểm tra match trên bus sẽ xảy ra khi đăng kí một device hoặc driver mới. 

[HINH VE]

Mỗi driver và device được đăng kí nằm trên một bus. Điều này tạo thành một cây. Các bus USB có thể là con của các bus PCI. Trong khi MDIO bus là con của các device khác... 

[HÌNH VẼ]

Khi ta đăng kí một driver với hàm ```platform_driver_probe()```, kernel sẽ tra một bảng các platform device cùng nằm trên bus và kiểm tra xem có match không. Nếu match thì sẽ gọi tới hàm probe.

#### Làm thế nào để platform device và platform driver match với nhau?
Cơ chế match đã đề cập ở trên diễn ra như thế nào đối với platform device và driver?
Ta đã biết răng cả device lẫn driver đều phải được đăng kí với kernel, vậy làm sao để kernel biết device nào được điều khiển bởi driver nào? Câu trả lời là ```MODULE_DEVICE_TABLE```. Đây là một macro, khi gọi macro này, driver sẽ tạo ra một bảng ID, bảng này sẽ mô tả những device mà driver hỗ trợ. 
Cùng lúc đó, nếu driver được biên dịch thành module, ta nên đặt tên module giống trường driver.name. Nếu không đặt cùng tên, module sẽ không được tự động load khi driver match với device, trừ khi ta sử dụng macro ```MODULE_ALIAS``` để thêm các tên khác cho module. 
Ở thời điểm biên dịch, những thông tin này sẽ được lấy từ driver để tạo thành một bảng các device id . Khi kernel cần tìm driver cho một device, nó chạy vòng lặp hàm match trên các driver có trên bus, hàm sẽ rà soát trong các bảng device của driver. Nếu tìm thấy driver có tên trùng với trường ```compatible``` (với device tree), ```device/vendor id ```hoặc ```name``` (với bảng device ID), thì module của driver tương ứng sẽ được load lên.
Macro ```MODULE_DEVICE_TABLE``` được định nghĩa trong ```linux/module.h```:
```c
#define MODULE_DEVICE_TABLE(type, name)
```

Với ```type``` có thể là ```i2c, spi, acpi, of, platform, usb, pci``` hoặc một loại bus khác có trong ```include/linux/mod_devicetable.h```. Điều này phụ thuộc vào device nằm trên loại đường bus nào, hoặc cơ chế match mà ta muốn dùng. 
Tham số name là con trỏ đến một mảng ```XXX_device_id```. Nếu ta cần match I2C device thì nó sẽ là ```i2c_device_id```, hoặc với SPI device thì là ```spi_device_id```. Cả ```i2c_device_id``` và spi_device_id đều là match device khi ta define device với platform data trên source code (cách cũ). Nếu muốn match device được định nghĩa trong device tree, ta phải sử dụng ```of_device_id``` (Open Firmware).

Ta sẽ đi chi tiết vào cơ chế match, ngoại trừ với kiểu match OF sẽ được bàn trong một bài viết về device tree.

##### Hàm match device và driver
Hàm này có trách nhiệm match giữa platform device và driver trong kernel, hàm được định nghĩa trong ```/drivers/base/platform.c```:
```c
static int platform_match(struct device *dev, struct device_driver *drv) 
{ 
	struct platform_device *pdev = to_platform_device(dev); 
	struct platform_driver *pdrv = to_platform_driver(drv); 

	/* When driver_override is set, only bind to the matching driver */ 
	if (pdev->driver_override) 
	return !strcmp(pdev->driver_override, drv->name); 

	/* Attempt an OF style match first */ 
	if (of_driver_match_device(dev, drv)) 
	return 1; 

	/* Then try ACPI style match */ 
	if (acpi_driver_match_device(dev, drv)) 
	return 1; 

	/* Then try to match against the id table */ 
	if (pdrv->id_table) 
	return platform_match_id(pdrv->id_table, pdev) != NULL; 

	/* fall-back to driver name match */ 
	return (strcmp(pdev->name, drv->name) == 0); 
}
```

Ta có thể thấy có 4 cơ chế match có thể được thực hiện ở hàm trên. Tất cả thực chất đều là so sánh chuỗi. Ví dụ hàm ```platform_match_id```:
```c
static const struct platform_device_id *platform_match_id(const struct platform_device_id *id, struct platform_device *pdev) 
{ 
	while (id->name[0]) { 
		if (strcmp(pdev->name, id->name) == 0) { 
			pdev->id_entry = id; 
			return id; 
		} 
		id++; 
	} 
	return NULL; 
}
```

**Kiểu match bằng ID table**
Kiểu match này đã được sử dụng khá lâu rồi, nó dựa vào một struct là ```device_id```. Tất cả các struct này được định nghĩa trong ```include/linux/mod_devicetable.h```. Chúng có thể là ```struct i2c_device_id``` cho I2C, ```struct platform_device_id``` cho platform devices... 
```c
struct platform_device_id {
	char name[PLATFORM_NAME_SIZE];
	kernel_ulong_t driver_data;
};
```

Khi một bảng ID được đăng kí, nó sẽ được rà soát mỗi khi kernel chạy hàm match để tìm driver cho một platform device mới. Nếu match, hàm ```probe``` của driver sẽ được gọi (kèm theo tham số ```struct platform_device``` được chỉ định, tham số này giữ con trỏ của device id được match). 
Trường ```driver_data``` thỉnh thoảng được ép kiểu thành một địa chỉ để có thể trỏ đến bất kì kiểu dữ liệu nào. 
Ví dụ với ```drivers/tty/serial/imx.c``` :
```c
static const struct platform_device_id imx_uart_devtype[] = {
	{
		.name = "imx1-uart",
		.driver_data = (kernel_ulong_t) &imx_uart_devdata[IMX1_UART],
	}, {
		.name = "imx21-uart",
		.driver_data = (kernel_ulong_t) &imx_uart_devdata[IMX21_UART],
	}, {
		.name = "imx6q-uart",
		.driver_data = (kernel_ulong_t) &imx_uart_devdata[IMX6Q_UART],
	}, {
		/* sentinel */
	}
};
```

Trường ```.name``` phải cùng tên với tên của device mà ta đã đăng kí. 
Hãy để ý lại hàm ```platform_match_id```:
```c
static const struct platform_device_id *platform_match_id(const struct platform_device_id *id, struct platform_device *pdev) 
{ 
	while (id->name[0]) { 
		if (strcmp(pdev->name, id->name) == 0) { 
			pdev->id_entry = id; 
			return id; 
		} 
		id++; 
	} 
	return NULL; 
}
```

Ta thấ rằng mỗi lần match thì ```id_entry``` của ```platform_device``` sẽ được gán bằng ```platform_device_id``` phù hợp của driver mà ta đăng kí. Về sau hàm ```probe``` của driver sẽ được sử dụng tham số  ```platform_device *pdev```, ```pdev``` này đã có ```id_entry``` được gán sẵn, ta chỉ việc lấy ```id_entry``` này để định danh cho device thích hợp.
Ví dụ hàm ```probe``` của imx:
```c
static void serial_imx_probe_pdata(struct imx_port *sport,
		struct platform_device *pdev)
{
	struct imxuart_platform_data *pdata = dev_get_platdata(&pdev->dev);

	sport->port.line = pdev->id;
	sport->devdata = (struct imx_uart_data	*) pdev->id_entry->driver_data;

	if (!pdata)
		return;

	if (pdata->flags & IMXUART_HAVE_RTSCTS)
		sport->have_rtscts = 1;
}
```

Ta thấy ```id_entry``` đã được sử dụng để lấy dữ liệu của device.

** Dữ liệu của mỗi device trong ID table**

Một driver có thể hỗ trợ cho nhiều kiểu device. Trong trường hợp này, mỗi kiểu device sẽ cần những dữ liệu riêng. Ta sẽ sử dụng device id là chỉ số của mảng chứa những dữ liệu device thay vì một con trỏ dữ liệu. 
Các bước bao gồm
1. Định nghĩa một enum là các kiểu device mà driver hỗ trợ:
```c
enum abx80x_chip {
	AB0801,
	AB0803,
	AB0804,
	AB0805,
	AB1801,
	AB1803,
	AB1804,
	AB1805,
	ABX80X
};
```

2. Định nghĩa một struct mô tả data của device:
```c
struct abx80x_cap {
	u16 pn;
	boolhas_tc;
};
```

3. Tạo một mảng các dữ liệu:
```c
static struct abx80x_cap abx80x_caps[] = {
	[AB0801] = {.pn = 0x0801},
	[AB0803] = {.pn = 0x0803},
	[AB0804] = {.pn = 0x0804, .has_tc = true},
	[AB0805] = {.pn = 0x0805, .has_tc = true},
	[AB1801] = {.pn = 0x1801},
	[AB1803] = {.pn = 0x1803},
	[AB1804] = {.pn = 0x1804, .has_tc = true},
	[AB1805] = {.pn = 0x1805, .has_tc = true},
	[ABX80X] = {.pn = 0}
};
```

4. Tạo mảng ```platform_device_id``` mà driver hỗ trợ, và đăng kí với driver.
```c
static const struct i2c_device_id abx80x_id[] = {
	{ "abx80x", ABX80X },
	{ "ab0801", AB0801 },
	{ "ab0803", AB0803 },
	{ "ab0804", AB0804 },
	{ "ab0805", AB0805 },
	{ "ab1801", AB1801 },
	{ "ab1803", AB1803 },
	{ "ab1804", AB1804 },
	{ "ab1805", AB1805 },
	{ "rv1805", AB1805 },
	{ }
};
MODULE_DEVICE_TABLE(i2c, abx80x_id);

static struct i2c_driver abx80x_driver = {
	.driver		= {
		.name	= "rtc-abx80x",
	},
	.probe		= abx80x_probe,
	.remove		= abx80x_remove,
	.id_table	= abx80x_id,
};
module_i2c_driver(abx80x_driver);
```

5. Tạo hàm ```probe```:
```c
static int rs5c372_probe(struct i2c_client *client, const struct i2c_device_id *id)
{
	[...]
	/* We pick the index corresponding to our device */
	int index = id->driver_data;
	/*
	* And then, we can access the per device data
	* since it is stored in abx80x_caps[index]
	*/
}
```

Tuy nhiên, ngày nay một các platform driver không còn cung cấp bảng device id như trên nữa, chúng chỉ cần điền tên của driver trong trường ```driver.name```. Cơ chế match vẫn hoạt động do trong hàm ```platform_match```, ở cuối hàm, ta vẫn match được driver và device dựa theo tên của chúng. 


### Tổng kết
Như vậy ở bài này, ta đã được tìm hiểu về platform driver, platform device, được biết rằng chúng đều nằm trên pseudo platform bus. Với cơ chế match của bus, device sẽ tìm được driver điều khiển nó. Cuối cùng ta có thể triển khai hàm probe để xử lý device mong muốn. 

BeanRedArmy 10-11-2018
