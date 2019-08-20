## IIO Framework
Industrial I/O (IIO) là một kernel subsystem dùng cho các bộ ADC (Analog to Digital Converters) và DAC (Digital to Analog Converters). Do số lượng các loại sensor ngày càng tăng, kéo theo số lượng code thực thi lớn và không nhất quán, việc nhóm và phân loại chúng trở nên cần thiết. IIO framework được phát triển từ năm 2009 với mục đích giải quyết vấn đề này. Các cảm biến gia tốc, cảm biến đo dòng/áp, ánh sáng, độ ẩm đều sẽ thuộc về họ các IIO device.
Mô hình IIO dựa theo kiến trúc các device và kênh:
- Device đại diện cho con chip và đứng đầu trong cây.
- Kênh đại diện cho một đường dữ liệu của device. Một device có thể có nhiều kênh. Ví dụ một cảm biến gia tốc thường sẽ là một device với 3 kênh chov mo một tọa độ (X,Y,Z). 
Chip IIO là một cảm biến/converter vật lý. Tầng user space sẽ tiếp cận nó thông qua một character device, và bên trong sysfs là một tập các file đại diện cho từng kênh của nó.
Có 2 cách để tương tác với một IIO thông qua tầng user space:
- ```/sys/bus/iio/iio:deviceX/```: Đại diện cho cảm biến bằng các kênh của nó.
- ```/dev/iio:deviceX```: Một character device đại diện cho cảm biến.

[HÌNH VẼ]

Hình vẽ mô tả cách IIO framework tổ chức hoạt động giữa kernel và user space. Driver quản lý phần cứng và thông báo việc xử lý cho IIO core bằng cách sử dụng các API và IIO core cung cấp. 
Các API của IIO nằm ở nhiều header file:
```c
#include <linux/iio/iio.h> /* Bắt buộc */
#include <linux/iio/sysfs.h> /* Bắt buộc nếu sử dụng sysfs*/
#include <linux/iio/events.h> /* Dùng để quản lý event của IIO */
#include <linux/iio/buffer.h> /* Bắt buộc khi sử dụng triggered buffers */
#include <linux/iio/trigger.h>/* Chỉ sử dụng khi cần trigger driver (ít dùng)*/
```

Trong bài này ta sẽ mô tả các khái niệm của IIO framework như là:
- Cấu trúc dữ liệu của IIO (device, kênh, ...)
- Triggered buffer và capture liên tục.
- IIO triggers.
- Capture dữ liệu ở các chế độ one-shot hoặc chế độ liên tục.
- Các tools được dùng để test device.

### Cấu trúc dữ liệu của IIO
Một IIO device trong kernel được đại diện bằng ```struct iio_dev``` và được mô tả chức năng bằng ```struct iio_info```. 
#### struct iio_dev
Struct này sẽ cho ta biết:
- Có bao nhiêu kênh đươc sử dụng trong device?
- Các chế độ nào device có thể thực hiện: one-shot, triggered buffer?
- Các hàm callback nào có trong driver?
```c
struct iio_dev {
    [...]
    int modes;
    int currentmode;
    struct device dev;
    struct iio_buffer *buffer;
    int scan_bytes;
    const unsigned long *available_scan_masks;
    const unsigned long *active_scan_mask;
    bool scan_timestamp;
    struct iio_trigger *trig;
    struct iio_poll_func *pollfunc;
    struct iio_chan_spec const *channels;
    int num_channels;
    const char *name;
    const struct iio_info *info;
    const struct iio_buffer_setup_ops *setup_ops;
    struct cdev chrdev;
};
```

Các trường của ```iio_dev``` bao gồm:
- ```modes```: Đại diện cho các chế độ (mode) mà device hỗ trợ, bao gồm:
	- ```INDIO_DIRECT_MODE```: device cung cấp giao diện sysfs
	- ```INDIO_BUFFER_TRIGGERED```: device hỗ trợ triggers buffer. Mode này được thêm tự động cho device nếu ta cài đặt một trigger buffer bằng cách sử dụng hàm ```iio_triggered_buffer_setup()```.
	- ```INDIO_BUFFER_HARDWARE```: device có hardware buffer.
	- ```INDIO_ALL_BUFFER_MODES```: kết hợp của 2 mode trên.
- ```currentmode```: đại diện cho mode đang được sử dụng bởi device.
- ```dev```: đại diện cho ```struct device``` mà IIO device gắn vào.
- ```buffer```: Đây chính là buffer dữ liệu của ta, nó được đẩy ra user space khi ta sử dụng chế độ triggered buffer. Nó cũng được tự động cấp phát và liên kết với device khi ta sử dụng hàm ```iio_triggered_buffer_setup```.
- ```scan_bytes```: số byte được capture và gửi cho buffer. Khi sử dụng trigger buffer từ user space, buffer nên phải có độ rộng ít nhất là ```indio->scan_bytes```.
- ```available_scan_masks```: mảng các bitmask cho phép kênh. Khi sử dụng trigger buffer, ta có thể cho phép các kênh được capture và gửi cho IIO buffer hoặc không bằng cách điều chỉnh mảng này. Ví dụ:
```c
/*
* Bitmasks 0x7 (0b111) và 0 (0b000) được cho phép.
* Có nghĩa là ta có thể enable tất cả hoặc không một kênh nào.
* Và ta cũng không thể chỉ enable một chỉ hoặc hai kênh bất kì, ví dụ kênh X và Y
*/
static const unsigned long my_scan_masks[] = {0x7, 0};
indio_dev->available_scan_masks = my_scan_masks;
````

- ```active_scan_mask```: Bitmask được sử dụng để enable các kênh, chỉ những dữ liệu từ các kênh này mới được đẩy vào buffer. Ví dụ, với một bộ ADC 8 kênh, nếu ta chỉ muốn enable các kênh thứ nhất (0), ba (2) và kênh cuối (7) thì ta sử dụng bitmask ```active_scan_mas``` bằng ```0b10000101``` (0x85). Driver có thể sử dụng macro ```for_each_set_bit``` để rà và tìm dữ liệu trên kênh tương ứng rồi điền vào buffer.
- ```scan_timestamp```: Cho biết có capture timestamp đẩy vào buffer hay không. Nếu có thì timestamp sẽ được đẩy vào như phần tử cuối cùng của buffer. Timestamp này có độ rọng 8 byte.
- ```trig```: Trigger hiện tại của device.
- ```pollfunc```: Đây là hàm chạy khi trigger được nhận.
- ```channels```: mô tả tất cả các kênh mà device có.
- ```num_channels```: số lượng kênh trong channels.
- ```name```: tên của device.
- ```info```: thông tin của driver.
- ```setup_ops```: Tập các hàm callback để gọi trước và sau khi buffer được enable/disable:
	```c
	struct iio_buffer_setup_ops {
        int (* preenable) (struct iio_dev *);
        int (* postenable) (struct iio_dev *);
        int (* predisable) (struct iio_dev *);
        int (* postdisable) (struct iio_dev *);
        bool (* validate_scan_mask) (struct iio_dev *indio_dev,
            const unsigned long *scan_mask);
	};
	```
- ```chrdev```: character device liên kết được tạo bởi IIO core.

Hàm được dùng để cấp phát bộ nhớ cho IIO device là ```iio_device_alloc```():
```c
struct iio_dev *devm_iio_device_alloc(struct device *dev, int sizeof_priv)
```

```dev``` là device mà ```iio_dev``` được cấp phát và ```sizeof_priv``` là kích thước sử dụng cho private structure. Theo cách hày thì ta có thể truyền dữ liệu private cho mỗi deivce. Hàm này sẽ trả NULL nếu cấp phát lỗi:
```c
struct iio_dev *indio_dev;
struct my_private_data *data;
indio_dev = iio_device_alloc(sizeof(*data));
if (!indio_dev)
return -ENOMEM;
/*data is given the address of reserved momory for private data */
data = iio_priv(indio_dev);
```

Sau khi cấp phát bộ nhớ cho IIO device thì ta cần đăng kí nó:
```c
int iio_device_register(struct iio_dev *indio_dev)
```

Device sẽ sẵn sàng nhận yêu cầu từ user space sau khi thực thi làm trên.
Ta có thể lấy dữ liệu private bằng cách:
```c
struct my_private_data *the_data = iio_priv(indio_dev);
```

Khi không cần dùng device nữa, hàm hủy đăng kí là:
```c
void iio_device_unregister(struct iio_dev *indio_dev)
```

Sau khi hủy đăng ký thì ta sẽ giải phóng bộ nhớ bằng hàm:
```c
void iio_device_free(struct iio_dev *iio_dev)
```

#### iio_info structure
```struct iio_info``` được sử dụng để khai báo các callback để IIO core sử dụng, với mục đích đọc/ghi các giá trị kênh.
```c
struct iio_info {
    struct module *driver_module;
    const struct attribute_group *attrs;
    int (*read_raw)(struct iio_dev *indio_dev,
    struct iio_chan_spec const *chan,
    int *val, int *val2, long mask);
    int (*write_raw)(struct iio_dev *indio_dev,
    struct iio_chan_spec const *chan,
    int val, int val2, long mask);
    [...]
};
```

Các trường bao gồm:
- ```driver_module```: thường đặt là ```THIS_MODULE```.
- ```attrs```: đại diện cho các thuộc tính của device.
- ```read_raw```: đây là hàm callback, được gọi khi user đọc file trong sysfs. Tham số ```mask``` là bitmask cho phép ta biết được loại giá trị được yêu cầu. Tham số chan là kênh mong muốn.
- ```write_raw```: Hàm callback để ghi dữ liệu vào device. 
Dưới đây là ví dụ để cài đặt một ```struct iio_info```:
```c
static const struct iio_info iio_dummy_info = {
    .driver_module = THIS_MODULE,
    .read_raw = &iio_dummy_read_raw,
    .write_raw = &iio_dummy_write_raw,
[...]
/*
* Provide device type specific interface functions and
* constant data.
*/
indio_dev->info = &iio_dummy_info;
```

#### Các kênh IIO
Ví dụ với cảm biến gia tốc, 3 kênh X, Y, Z, mỗi kênh sẽ được đại diện bằng một ```struct iio_chan_spec```:
```c
struct iio_chan_spec {
    enum iio_chan_type type;
    int channel;
    int channel2;
    unsigned long address;
    int scan_index;
    struct {
        charsign;
        u8 realbits;
        u8 storagebits;
        u8 shift;
        u8 repeat;
        enum iio_endian endianness;
    } scan_type;
    long info_mask_separate;
    long info_mask_shared_by_type;
    long info_mask_shared_by_dir;
    long info_mask_shared_by_all;
    const struct iio_event_spec *event_spec;
    unsigned int num_event_specs;
    const struct iio_chan_spec_ext_info *ext_info;
    const char *extend_name;
    const char *datasheet_name;
    unsigned modified:1;
    unsigned indexed:1;
    unsigned output:1;
    unsigned differential:1;
    };
```
Struct bao gồm các trường:
- ```type```: chỉ ra kênh sẽ đo kiểu dữ liệu gì. Nếu đo điện áp thì sẽ là ```IIO_VOLTAGE```. Nếu là cảm biến ánh sáng thì là ```IIO_LIGHT```. Với cảm biến gia tốc thì là ```IIO_ACCEL```. Ta có thể xem thêm các kiểu khác trong ```include/uapi/linux/iio/types.h``` ở ```enum iio_chan_type```. 
- ```channel```: index của kênh, có hiệu lực khi ```.indexed``` được đặt bằng 1.
- ```channel2```: modifier của kênh, có hiệu lực khi ```.modified``` được đặt bằng 1.
- ```modified```:   Chỉ ra liệu modifier có được áp dụng cho thuộc tính kênh hay không. Trong trường hợp bằng 1, modifier được đặt trong ```.channel2```.
- ```indexed```: Chỉ ra liệu thuộc tính kênh có được đánh index hay không. Nếu có thì index kênh được đặt trong trường ```.channel```.
- ```scan_index``` và ```scan_type```: Những trường này được sử dụng để xác định phần tử từ một buffer khi ta sử dụng buffer triggers. ```scan_index``` là vị trí của dữ liệu kênh được capture bên trong buffer. Các kênh có ```scan_index``` thấp thơn sẽ được đặt trước các kênh có index cao hơn. Đặt ```.scan_index = -1``` sẽ ngăn không caputre kênh.
User space có thể nhìn thấy các thuộc tính của kênh ở sysfs nếu ta điều chỉnh các bitmask, bao gồm các trường mask sau:
- ```info_mask_separate```: các thuộc tính chỉ riêng cho kênh
- ```info_mask_shared_by_type```: thuộc tính được chia sẻ bởi tất các kênh cùng một kiểu.
- ```info_mask_shared_by_dir```: thuộc tính được chia sẻ bởi tất cả các kênh cùng một thư mục.
- ```info_mask_shared_by_all```: thuộc tính được chia sẻ bởi tất cả các kênh, cho dù chúng khác kiểu và khác thư mục.
4 trường trên có thể được đặt các giá trị hoặc OR các giá trị sau:
enum iio_chan_info_enum {
    IIO_CHAN_INFO_RAW = 0,
    IIO_CHAN_INFO_PROCESSED,
    IIO_CHAN_INFO_SCALE,
    IIO_CHAN_INFO_OFFSET,
    IIO_CHAN_INFO_CALIBSCALE,
    [...]
    IIO_CHAN_INFO_SAMP_FREQ,
    IIO_CHAN_INFO_FREQUENCY,
    IIO_CHAN_INFO_PHASE,
    IIO_CHAN_INFO_HARDWAREGAIN,
    IIO_CHAN_INFO_HYSTERESIS,
    [...]
};
##### Quy định đặt tên cho thuộc tính kênh
Tên thuộc tính được IIO core tạo ra theo mẫu:
```{direction}_{type}_{index}_{modifier}_{info_mask}``` , trong đó:
- ```direction```: có thể là ```in``` hoặc ```out```:
	```c
	static const char * const iio_direction[] = {
        [0] = "in",
        [1] = "out",
    };
    ```

 - ```type```: có thể là một trong các kiểu sau:
    ```c
    static const char * const iio_chan_type_name_spec[] = {
        [IIO_VOLTAGE] = "voltage",
        [IIO_CURRENT] = "current",
        [IIO_POWER] = "power",
        [IIO_ACCEL] = "accel",
        [...]
        [IIO_UVINDEX] = "uvindex",
        [IIO_ELECTRICALCONDUCTIVITY] = "electricalconductivity",
        [IIO_COUNT] = "count",
        [IIO_INDEX] = "index",
        [IIO_GRAVITY] = "gravity",
    };
    ```
- ```index``` : phụ thuộc vào trường ```.indexed``` có được set hay không. Nếu được set thì ```index``` sẽ được lấy từ trường ```.channel``` để thay vào ```{index}```
- ```modifier```: dựa vào trường ```.modified``` có được set hay không. Nếu được set thì ```modifier``` sẽ được lấy từ trường ```channel2```, và ```{modifier}``` được thay thế dựa theo ```struct iio_modifier_names```:
    ```c
    static const char * const iio_modifier_names[] = {
        [IIO_MOD_X] = "x",
        [IIO_MOD_Y] = "y",
        [IIO_MOD_Z] = "z",
        [IIO_MOD_X_AND_Y] = "x&y",
        [IIO_MOD_X_AND_Z] = "x&z",
        [IIO_MOD_Y_AND_Z] = "y&z",
        [...]
        [IIO_MOD_CO2] = "co2",
        [IIO_MOD_VOC] = "voc",
    };
    ```

- ```info_mask``` dựa vào mask của info kênh:
    ```c
    /* relies on pairs of these shared then separate */
    static const char * const iio_chan_info_postfix[] = {
        [IIO_CHAN_INFO_RAW] = "raw",
        [IIO_CHAN_INFO_PROCESSED] = "input",
        [IIO_CHAN_INFO_SCALE] = "scale",
        [IIO_CHAN_INFO_CALIBBIAS] = "calibbias",
        [...]
        [IIO_CHAN_INFO_SAMP_FREQ] = "sampling_frequency",
        [IIO_CHAN_INFO_FREQUENCY] = "frequency",
        [...]
    };
    ```

**Phân biệt các kênh**
Ta có thể cảm thấy khó khăn khi có nhiều kênh dữ liệu cho mỗi loại kênh. Khi đó làm thế nào để phân biệt chúng. Có 2 giải pháp là dùng ```index``` và ```modifiers```.
- Sử dụng ```index```: Khi cho một ADC device với một kênh thì việc đánh ```index``` là không cần thiết. Giả sử ta có 4 hoặc 8 kênh, để định danh chúng ta có thể đánh ```index``` cho chúng. Đặt trường ```.indexed = 1``` thì ```{index}``` sẽ là giá trị của trường ```.channel```:

```c
static const struct iio_chan_spec adc_channels[] = {
{
    .type = IIO_VOLTAGE,
    .indexed = 1,
    .channel = 0,
    .info_mask_separate = BIT(IIO_CHAN_INFO_RAW),
},
{
    .type = IIO_VOLTAGE,
    .indexed = 1,
    .channel = 1,
    .info_mask_separate = BIT(IIO_CHAN_INFO_RAW),
},
{
    .type = IIO_VOLTAGE,
    .indexed = 1,
    .channel = 2,
    .info_mask_separate = BIT(IIO_CHAN_INFO_RAW),
},
{
    .type = IIO_VOLTAGE,
    .indexed = 1,
    .channel = 3,
    .info_mask_separate = BIT(IIO_CHAN_INFO_RAW),
},
}
```

Khi đó các thuộc tính kênh được xuất ra là:
```sh
/sys/bus/iio/iio:deviceX/in_voltage0_raw
/sys/bus/iio/iio:deviceX/in_voltage1_raw
/sys/bus/iio/iio:deviceX/in_voltage2_raw
/sys/bus/iio/iio:deviceX/in_voltage3_raw
```

- Sử dụng ```modifier```: Khi cho một cảm biến ánh sáng có 2 kênh, một kênh cho tia hồng ngoại và một kênh còn lại cho cả hồng ngoại lẫn ánh sáng nhìn thấy, nếu không sử dụng ```index``` hoặc ```modifier``` thì tên thuộc tính đều sẽ là ```in_intensity_raw```. Sử dụng ```index``` để phân biệt cũng được nhưng sẽ dễ gây ra nhầm lẫn khi sử dụng tên, trong khi sử dụng ```modifier``` giúp ta có được tên  thuộc tính có ý nghĩa hơn. Ví dụ:
```c
static const struct iio_chan_spec mylight_channels[] = {
{
    .type = IIO_INTENSITY,
    .modified = 1,
    .channel2 = IIO_MOD_LIGHT_IR,
    .info_mask_separate = BIT(IIO_CHAN_INFO_RAW),
    .info_mask_shared = BIT(IIO_CHAN_INFO_SAMP_FREQ),
},
{
    .type = IIO_INTENSITY,
    .modified = 1,
    .channel2 = IIO_MOD_LIGHT_BOTH,
    .info_mask_separate = BIT(IIO_CHAN_INFO_RAW),
    .info_mask_shared = BIT(IIO_CHAN_INFO_SAMP_FREQ),
},
{
    .type = IIO_LIGHT,
    .info_mask_separate = BIT(IIO_CHAN_INFO_PROCESSED),
    .info_mask_shared = BIT(IIO_CHAN_INFO_SAMP_FREQ),
},
}
```

Khi đó các thuộc tính sẽ là:
```sh
/sys/bus/iio/iio:deviceX/in_intensity_ir_raw cho kênh đo cường độ hồng ngoại
/sys/bus/iio/iio:deviceX/in_intensity_both_raw cho kênh đo cả 2
/sys/bus/iio/iio:deviceX/in_illuminance_input cho xử lý dữ liệu
/sys/bus/iio/iio:deviceX/sampling_frequency cho tần số lấy mẫu, chia sẻ cho tất cả kênh.
```

#### Ví dụ hoàn chỉnh
Chúng ta thử xem một ví dụ hoàn chỉnh, tổng hợp những thứ vừa được học xem nó có gì. Đây là một ví dụ có 4 kênh đo điện áp:
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/platform_device.h>
#include <linux/interrupt.h>
#include <linux/of.h>
#include <linux/iio/iio.h>
#include <linux/iio/sysfs.h>
#include <linux/iio/events.h>
#include <linux/iio/buffer.h>
#define FAKE_VOLTAGE_CHANNEL(num) \
{ \
    .type = IIO_VOLTAGE, \
    .indexed = 1, \
    .channel = (num), \
    .address = (num), \
    .info_mask_separate = BIT(IIO_CHAN_INFO_RAW), \
    .info_mask_shared_by_type = BIT(IIO_CHAN_INFO_SCALE) \
}
struct my_private_data {
    int foo;
    int bar;
    struct mutex lock;
};
static int fake_read_raw(struct iio_dev *indio_dev,
struct iio_chan_spec const *channel, int *val,
int *val2, long mask)
{
    return 0;
}
static int fake_write_raw(struct iio_dev *indio_dev,
struct iio_chan_spec const *chan,
int val, int val2, long mask)
{
    return 0;
}
static const struct iio_chan_spec fake_channels[] = {
    FAKE_VOLTAGE_CHANNEL(0),
    FAKE_VOLTAGE_CHANNEL(1),
    FAKE_VOLTAGE_CHANNEL(2),
    FAKE_VOLTAGE_CHANNEL(3),
};
static const struct of_device_id iio_dummy_ids[] = {
{   .compatible = "packt,iio-dummy-random", },
{   /* sentinel */ }
};
static const struct iio_info fake_iio_info = {
    .read_raw = fake_read_raw,
    .write_raw = fake_write_raw,
    .driver_module = THIS_MODULE,
};
static int my_pdrv_probe (struct platform_device *pdev)
{
    struct iio_dev *indio_dev;
    struct my_private_data *data;
    indio_dev = devm_iio_device_alloc(&pdev->dev, sizeof(*data));
    if (!indio_dev) {
        dev_err(&pdev->dev, "iio allocation failed!\n");
        return -ENOMEM;
    }
    data = iio_priv(indio_dev);
    mutex_init(&data->lock);
    indio_dev->dev.parent = &pdev->dev;
    indio_dev->info = &fake_iio_info;
    indio_dev->name = KBUILD_MODNAME;
    indio_dev->modes = INDIO_DIRECT_MODE;
    indio_dev->channels = fake_channels;
    indio_dev->num_channels = ARRAY_SIZE(fake_channels);
    indio_dev->available_scan_masks = 0xF;
    iio_device_register(indio_dev);
    platform_set_drvdata(pdev, indio_dev);
    return 0;
}
static void my_pdrv_remove(struct platform_device *pdev)
{
    struct iio_dev *indio_dev = platform_get_drvdata(pdev);
    iio_device_unregister(indio_dev);
}
static struct platform_driver mypdrv = {
    .probe = my_pdrv_probe,
    .remove = my_pdrv_remove,
    .driver = {
        .name = "iio-dummy-random",
        .of_match_table = of_match_ptr(iio_dummy_ids),
        .owner = THIS_MODULE,
    },
};
module_platform_driver(mypdrv);
MODULE_AUTHOR("John Madieu <john.madieu@gmail.com>");
MODULE_LICENSE("GPL");
```

Sau khi load module thì ra sẽ có được output như sau:
```sh
~# ls -l /sys/bus/iio/devices/
lrwxrwxrwx 1 root root 0 Jul 31 20:26 iio:device0 ->
../../../devices/platform/iio-dummy-random.0/iio:device0
lrwxrwxrwx 1 root root 0 Jul 31 20:23 iio_sysfs_trigger ->
../../../devices/iio_sysfs_trigger
Các kênh của device:
~# ls /sys/bus/iio/devices/iio\:device0/
dev in_voltage2_raw name uevent
in_voltage0_raw in_voltage3_raw power
in_voltage1_raw in_voltage_scale subsystem
~# cat /sys/bus/iio/devices/iio:device0/name
iio_dummy_random
```

### Hỗ trợ triggered buffer
Trong nhiều ứng dụng phân tích dữ liệu, ta thường phải capture dữ liệu bằng cách sử dụng một tín hiệu ngoài, có thể gọi là trigger. Trigger có thể là:
- Tín hiệu data đã sẵn sàng
- Ngắt ngoài (Ví dụ GPIO)
- Ngắt mềm bằng xung CPU tuần hoàn
- Tín hiệu đọc/ghi từ user space thông qua sysfs
Một số loại trigger hiện có trên Linux là:
- ```iio-trig-intterupt```: Hỗ trợ IRQ như một IIO trigger. Để bật mode này trong kernel thì sử dụng option ```CONFIG_IIO_INTERRUPT_TRIGGER```. 
- ```iio-trig-hrtimer```: Sử dụng HRT như một IIO trigger, coi đó là một nguồn ngắt. Để bật mode này trong kernel thì sử dụng option ```IIO_HRTIMER_TRIGGER```. 
- ```iio-trig-sysfs```: Cho phép ta sử dụng sysfs để kích hoạt capture dữ liệu. Để bật mode này trong kernel thì sử dụng option ```CONFIG_IIO_SYSFS_TRIGGER```.
- ```iio-trig-bfin-timer``` : Sử dụng blackfin timer như IIO trigger (vẫn trong giai đoạn phát triển).
IIO framework cung cấp cho ta các API để:
- Khai báo số cho trigger.
- Chọn kênh để dữ liệu của kênh đó được đưa vào buffer.

Buffer và trigger kết nối với nhau như thế nào? Khi IIO device của ta hỗ trợ trigger buffer, ta phải cài đặt hàm ```iio_dev.pollfunc```, đây là hàm thực thi khi có trigger. Hàm này có nhiệm vụ tìm các kênh được cho phép thông qua ```indio_dev->active_scan_mask```, lấy dữ liệu từ các kênh và đẩy vào  ```indio_dev->buffer``` bằng hàm ```iio_push_to_buffers_with_timestamp```. 
IIO core cung cấp một tập các hàm để cài đặt triggered buffer, ta có thể tìm trong ```drivers/iio/industrialio-triggered-buffer.c``` .
Các bước để cài đặt triggered buffer như sau:
1. Điền ```struct iio_buffer_setup_ops``` nếu cần:
```c
const struct iio_buffer_setup_ops sensor_buffer_setup_ops = {
    .preenable = my_sensor_buffer_preenable,
    .postenable = my_sensor_buffer_postenable,
    .postdisable = my_sensor_buffer_postdisable,
    .predisable = my_sensor_buffer_predisable,
};
```

2. Do xử lý trigger giống như việc xử lý ngắt nên ta cần 2 hàm top half và bottom half. Viết hàm top half cho trigger:
```c
irqreturn_t sensor_iio_pollfunc(int irq, void *p)
{
    pf->timestamp = iio_get_time_ns((struct indio_dev *)p);
    return IRQ_WAKE_THREAD;
}
```

3. Viết hàm bottom half cho trigger, hàm này sẽ lấy data từ các kênh và đưa vào buffer:
```c
irqreturn_t sensor_trigger_handler(int irq, void *p)
{
    u16 buf[8];
    int bit, i = 0;
    struct iio_poll_func *pf = p;
    struct iio_dev *indio_dev = pf->indio_dev;
    /* one can use lock here to protect the buffer */
    /* mutex_lock(&my_mutex); */
    /* read data for each active channel */
    for_each_set_bit(bit, indio_dev->active_scan_mask,
    indio_dev->masklength)
        buf[i++] = sensor_get_data(bit)
    /*
    * If iio_dev.scan_timestamp = true, the capture timestamp
    * will be pushed and stored too, as the last element in the
    * sample data buffer before pushing it to the device buffers.
    */
    iio_push_to_buffers_with_timestamp(indio_dev, buf, timestamp);
    /* Please unlock any lock */
    /* mutex_unlock(&my_mutex); */
    /* Notify trigger */
    iio_trigger_notify_done(indio_dev->trig);
    return IRQ_HANDLED;
}
```

4. Cuối cùng cài đặt triggered buffer trong hàm ```probe``` bằng hàm:
```c
iio_triggered_buffer_setup(indio_dev, sensor_iio_polfunc,
sensor_trigger_handler,
sensor_buffer_setup_ops);
```

Hàm ```iio_triggered_buffer_setup``` sẽ tự động gán mode ```INDIO_DIRECT_MODE``` cho device. 

Một chú ý quan trọng là khi capture một buffer liên tiếp (đẩy dữ liệu từ nhiều kênh vào buffer), có thể xảy ra hiện tượng xung đột giữa trigger handler của triggered buffer với hàm callback ```read_raw```() để đọc giá trị đơn kênh ở một thời điểm. Vì vậy ở hàm ```read_raw```() ta sẽ sử dụng hàm ```iio_buffer_enabled```() để kiểm tra. Hàm callback sẽ đại loại như sau:
```c
static int my_read_raw(struct iio_dev *indio_dev,
const struct iio_chan_spec *chan,
int *val, int *val2, long mask)
{
    [...]
    switch (mask) {
    case IIO_CHAN_INFO_RAW:
    if (iio_buffer_enabled(indio_dev))
    return -EBUSY;
    [...]
}
```

Ta tổng kết lại những trường và hàm quan trọng vừa mới nghiên cứu:
- ```iio_buffer_setup_ops```: cung cấp các hàm callback sẽ được gọi khi cài đặt buffer (trước/sau bật/tắt). Nếu không khai báo thì hàm ```iio_triggered_buffer_setup_ops``` mặc định sẽ được sử dụng.
- ```sensor_iio_pollfunc```: hàm top half của trigger. Cũng như mọi hàm top half khác của ngắt, hàm này chạy ở interrupt context, và nó phải làm ít công việc nhất có thể. Trong 99% trường hợp ta sẽ cần lấy giá trị timestamp ở thời điểm có trigger. 
- ```sensor_trigger_handler```: hàm bottom half của trigger, chạy ở thread của kernel (hay còn gọi là process context), cho phép ta làm bất cứ việc gì, kể cả ngủ hay sử dụng mutex. Nhiệm vụ của hàm này thường là đọc dữ liệu từ device, đẩy dữ liệu vào buffer cùng với timestamp ghi được ở top half.

Trigger sẽ cho driver biết thời điểm đọc dữ liệu từ nhiều kênh của device và đẩy vào buffer. Ngoài ra nếu không cần thiết, ta có thể chỉ sử dụng chế độ single shot, tức là chỉ đọc một giá trị trên một kênh sử dụng hàm ```read_raw```() của kênh.

#### IIO trigger và sysfs (user space)
Có 2 nơi trong sysfs liên quan đến trigger là:
- ```/sys/bus/iio/devices/triggerY/```: nơi được tạo mỗi khi IIO trigger được đăng kí với IIO core và tương ứng với trigger có index là Y. Trong thư mục này có thuộc tính name, thuộc tính nà sẽ được sử dụng về sau để liên kết với device.
- ```/sys/bus/iio/devices/iio:deviceX/trigger/*``` : thư mục sẽ được tạo tự động khi device bật trigger buffer. Ta có thể liên kết trigger với device bằng cách viết tên của trigger (ở thuộc tính name) vào file ```current_trigger```.

##### Tạo trigger bằng sysfs
Trigger bằng sysfs được enable khi ta bật option ```CONFIG_IIO_SYSFS_TRIGGER = y```. Khi bật option này thì thư mục  ```/sys/bus/iio/devices/iio_sysfs_trigger``` tự động được tạo, và thư mục này dùng để quản lý các sysfs trigger. Trong thư mục có 2 file là ```add_trigger``` và ```remove_trigger```. 

**add_trigger file**

Đây là file dùng để tạo một sysfs trigger mới. Ta sẽ tạo bằng cách ghi vào một số dương X (coi số X này như index của trigger), khi đó sẽ xuất hiện /sys/bus/iio/devices/triggerX.
Ví dụ
```sh
# echo 2 > add_trigger
```

sẽ tạo một trigger buffer ở ```/sys/bus/iio/devices/trigger2```. Nếu ta lấy tên của trigger này:
```sh
$ cat /sys/bus/iio/devices/trigger2/name
sysfstrig2
```

Trong ```/sys/bus/iio/devices/trigger2/``` cũng tồn tại một file ```trigger_now``` và nếu ta ghi 1 vào file này thì sẽ bắt đầu capture dữ liệu.

**remove_trigger file**

 Để xóa bỏ một sysfs trigger thì ta dùng file này. Ví dụ:
 ```sh
# echo 2 > remove_trigger
```

**Liên kết device với sysfs trigger**

Để liên kết trigger 2 này với một device thì ta sẽ ghi giá trị ```sysfstrig2``` vào file ```current_trigger``` của ```/sys/bus/iio/devices/iio:deviceX/trigger/*``` . Ví dụ
```sh
# set trigger2 as current trigger for device0
# echo sysfstrig2 > /sys/bus/iio/devices/iio:device0/trigger/current_trigger
```

Để tháo bỏ liên kết thì ta lại ghi chuỗi "" vào file:
```sh
# echo "" > iio:device0/trigger/current_trigger
```

##### Tạo trigger bằng interrupt.
Trước hết ta xem ví dụ:
```c
static struct resource iio_irq_trigger_resources[] = {
    [0] = {
        .start = IRQ_NR_FOR_YOUR_IRQ,
        .flags = IORESOURCE_IRQ | IORESOURCE_IRQ_LOWEDGE,
    },
};
static struct platform_device iio_irq_trigger = {
    .name = "iio_interrupt_trigger",
    .num_resources = ARRAY_SIZE(iio_irq_trigger_resources),
    .resource = iio_irq_trigger_resources,
};
platform_device_register(&iio_irq_trigger);
```

Với khai báo như vậy, ta sẽ có một IRQ trigger và load một module cho IRQ trigger đó. Nếu hàm probe của module này thành công thì sẽ có một thư mục tương ứng trong sysfs. 
Ví dụ:
```sh
$ cd /sys/bus/iio/devices/trigger0/
$ cat name
irqtrig85
```

Khi có được tên của trigger thì việc tiếp theo là liên kết nó với device bằng cách ghi tên trigger vào ```current_trigger```:
```sh
# echo "irqtrig85" >
```

```/sys/bus/iio/devices/iio:device0/trigger/current_trigger```.
Kể từ giờ mỗi lần có ngắt thì dữ liệu của device sẽ được capture.

##### Tạo trigger bằng hrtimer
hrtimer trigger được enable nếu ta bật config ```CONFIG_IIO_CONFIGFS``` và mount vào hệ thống:
```sh
# mkdir /config
# mount -t configfs none /config
```


Lúc này nếu ta load module ``` iio-trig-hrtimer``` thì sẽ tạo ra một IIO groups, cho phép ra tạo hrtimer trigger bằng ```/config/iio/triggers/hrtimer``` .
Ví dụ:
```sh
# create a hrtimer trigger
$ mkdir /config/iio/triggers/hrtimer/my_trigger_name
# remove the trigger
$ rmdir /config/iio/triggers/hrtimer/my_trigger_name
```

Mỗi hrtimer trigger chứa một file thuộc tính ```sampling_frequency```. Ta sẽ thảo luận kĩ hơn vấn đề này ở phần sau.

#### IIO buffer.
IIO buffer cung cấp cho ta phương thức capture dữ liệu liên tục từ nhiều kênh cùng một lúc. Buffer này có thể truy cập từ user space thông qua character device  ```/dev/iio:device.``` 

##### IIO buffer ở sysfs
Một IIO buffer có một thư mục chứa các thuộc tính liên kết là ```/sys/bus/iio/iio:deviceX/buffer/*```. Các thuộc tính liên kết là:
- ```length```: Tổng số mẫu dữ liệu có thể lưu trữ ở buffer. 
- ```enable```: Kích hoạt buffer capture.
- ```watermark```: Thuộc tính này xuất hiện từ kernel 4.2. Đây là một số nguyên dương chỉ ra số lượng các phần tử mà một lệnh read phải đợi để thoát khỏi trạng thái block. Ví dụ với lệnh poll, lệnh này sẽ bị block cho đến khi đạt được số ```watermark```. Số này chỉ có nghĩa nếu nó lớn hơn số lượng phần tử cần đọc. 

##### Cài đặt IIO buffer.
Một kênh mà dữ liệu của nó được đọc và đẩy vào buffer ta gọi là một scan element. Ta có thể cấu hình chúng thông qua thư mục ```/sys/bus/iio/iio:deviceX/scan_elements/*```. Thư mục này chứa các thuộc tính để cấu hình:
- ```en```: được sử dụng để enable kênh. Khi và chỉ khi thuộc tính này khác không, capture bởi trigger mới lấy mẫu dữ liệu của kênh. Ví dụ ```in_voltage0_en``` , ```in_voltage1_en```.
- ```*_type```: Mô tả scan element lưu trữ trong buffer và form mà nó đọc từ userspace. Ví dụ ```in_voltage0_type```. Format là ```[be|le]:[s|u]bits/storagebitsXrepeat[>>shift]```. Trong đó:
	- ```be``` hoặc ```le``` chỉ ra kiểu endian.
	- ```s``` và ```u``` chỉ ra là số có dấu hay không dấu.
	- ```bits``` là số bit dữ liệu valid
	- ```storagebits``` là số bit dữ liệu mà kênh chiếm dụng trong buffer. Có nghĩa là giả sử dữ liệu có thể được mã hóa bằng 12 bit nhưng lại có thể chiếm 16 bit trong buffer. Do đó ta cần phải dịch phải 4 lần 16 bit trong buffer để đạt được giá trị thực tế.
	- ```shift``` là số lần dịch trước khi bỏ đi những bit không cần. 
	- ```repeat``` là số lần lặp ```bit/storagebit```. Nếu giá trị này là 0 hoặc 1 nghĩa là không lặp lại.

Lấy ví dụ trong document đễ minh họa rõ hơn. Ví dụ ta có một driver cho cảm biến gia tốc 3 trục, dữ liệu mỗi trục được lưu được trong các thanh ghi 8 bit nhưng lại được lưu trong buffer bằng 12 bit.

[HÌNH VẼ]

```sh
$ cat /sys/bus/iio/devices/iio:device0/scan_elements/in_accel_y_type
le:s12/16>>4
```
Ta hiểu là đây là kiểu dữ liệu có dấu, little endian, lưu trữ ở buffer bằng 16 bit, cần phải dịch phải 4 bit trước khi lấy 12 bit dữ liệu.
Trong ```struct iio_chan_spec``` có trường ```scan_type``` có nhiệm vụ xác định cách mà giá trị của kênh được lưu vào buffer.
```c
struct iio_chan_spec {
    [...]
    struct {
        char sign; /* Should be 'u' or 's' as explained above */
        u8 realbits;
        u8 storagebits;
        u8 shift;
        u8 repeat;
        enum iio_endian endianness;
    } scan_type;
    [...]
};
```

Trường này hoàn toàn phù hợp với format ```[be|le]:[s|u]bits/storagebitsXrepeat[>>shift]``` ở trên.
Ví dụ:
```c
struct struct iio_chan_spec accel_channels[] = {
{
    .type = IIO_ACCEL,
    .modified = 1,
    .channel2 = IIO_MOD_X,
    /* other stuff here */
    .scan_index = 0,
    .scan_type = {
    .sign = 's',
    .realbits = 12,
    .storagebits = 16,
    .shift = 4,
    .endianness = IIO_LE,
},
}
/* similar for Y (with channel2 = IIO_MOD_Y, scan_index = 1)
* and Z (with channel2 = IIO_MOD_Z, scan_index = 2) axis
*/
}
```

#### Ví dụ thực tế
Đây là một driver của một cảm biến gia tốc 3 trục tên là BMA220 của hãng BOSH. Đây là một device giao tiếp tương thích cả SPI và I2C, các thanh ghi 8 bit và chứa bộ ngắt controller phát hiện chuyển động. Có thể tham khảo data sheet tại http://www.mouser.fr/pdfdocs/BSTBMA220DS00308.PDF 
Driver này có thể được bật bằng CONFIG_BMA200 từ kernel version 4.8.
Đầu tiên ta khai báo các kênh IIO bằng ```struct``` ```iio_chan_spec```. Khi sử dụng triggered buffer thì ta cũng cần điền các trường ```.scan_index``` và ```.scan_type```:
```c
#define BMA220_DATA_SHIFT 2
#define BMA220_DEVICE_NAME "bma220"
#define BMA220_SCALE_AVAILABLE "0.623 1.248 2.491 4.983"
#define BMA220_ACCEL_CHANNEL(index, reg, axis) { \
    .type = IIO_ACCEL, \
    .address = reg, \
    .modified = 1, \
    .channel2 = IIO_MOD_##axis, \
    .info_mask_separate = BIT(IIO_CHAN_INFO_RAW), \
    .info_mask_shared_by_type = BIT(IIO_CHAN_INFO_SCALE), \
    .scan_index = index, \
    .scan_type = { \
    .sign = 's', \
    .realbits = 6, \
    .storagebits = 8, \
    .shift = BMA220_DATA_SHIFT, \
    .endianness = IIO_CPU, \
}, \
}
static const struct iio_chan_spec bma220_channels[] = {
    BMA220_ACCEL_CHANNEL(0, BMA220_REG_ACCEL_X, X),
    BMA220_ACCEL_CHANNEL(1, BMA220_REG_ACCEL_Y, Y),
    BMA220_ACCEL_CHANNEL(2, BMA220_REG_ACCEL_Z, Z),
};
```

```.info_mask_separate = BIT(IIO_CHAN_INFO_RAW)``` có nghĩa là sẽ có một thuộc tính ```*_raw``` ở sysfs cho mỗi kênh, và  ```.info_mask_shared_by_type = BIT(IIO_CHAN_INFO_SCALE)``` có nghĩa là chỉ có một thuộc tính ```*_scale``` ở sysfs cho tất cả các kênh cùng kiểu 
```sh
jma@jma:~$ ls -l /sys/bus/iio/devices/iio:device0/
(...)
# without modifier, a channel name would have in_accel_raw (bad)
-rw-r--r-- 1 root root 4096 jul 20 14:13 in_accel_scale
-rw-r--r-- 1 root root 4096 jul 20 14:13 in_accel_x_raw
-rw-r--r-- 1 root root 4096 jul 20 14:13 in_accel_y_raw
-rw-r--r-- 1 root root 4096 jul 20 14:13 in_accel_z_raw
(...)
```

Khi đọc file ```in_accel_scale``` sẽ gọi đến hàm ```read_raw```() với ```mask``` được đặt bằng ```IIO_CHAN_INFO_SCALE``` và đọc file ```in_accel_x_raw``` sẽ gọi hàm ```read_raw```()  với ```mask``` được đặt bằng ```IIO_CHAN_INFO_RAW```.

```.scan_type``` chỉ ra rằng dữ liệu sẽ chiếm 8 bit trong buffer mặc dù thực tế số bit có ý nghĩa chỉ là 6 bit, dữ liệu cần phải dịch phải 2 lần trước khi bỏ đi các bit thừa. Bất kì các scan element sẽ có dạng:
```sh
$ cat /sys/bus/iio/devices/iio:device0/scan_elements/in_accel_x_type
le:s6/8>>2
```

Tiếp theo ta sẽ xem hàm pollfunc (thực tế là viết bottom half của nó), nhiệm vụ chính là đọc dữ liệu từ device và đẩy các giá trị này vào buffer. Việc giao tiếp với chip sẽ thực hiện trên SPI framework. Sau khi thao tác xong ta sẽ thông báo kết thúc việc đẩy vào buffer cho IIO bằng hàm ```iio_trigger_notify_done```():
```c
static irqreturn_t bma220_trigger_handler(int irq, void *p)
{
    int ret;
    struct iio_poll_func *pf = p;
    struct iio_dev *indio_dev = pf->indio_dev;
    struct bma220_data *data = iio_priv(indio_dev);
    struct spi_device *spi = data->spi_device;
    mutex_lock(&data->lock);
    data->tx_buf[0] = BMA220_REG_ACCEL_X | BMA220_READ_MASK;
    ret = spi_write_then_read(spi, data->tx_buf, 1, data->buffer, ARRAY_SIZE(bma220_channels) - 1);
    if (ret < 0)
        goto err;
    iio_push_to_buffers_with_timestamp(indio_dev,data->buffer, pf->timestamp);
    err:
    mutex_unlock(&data->lock);
    iio_trigger_notify_done(indio_dev->trig);
    return IRQ_HANDLED;
}
```

Tiếp tục đến hàm callback ```read_raw```(), đây là hàm được gọi mỗi lần ta đọc file của kênh trong sysfs:
```c
static int bma220_read_raw(struct iio_dev *indio_dev,
struct iio_chan_spec const *chan,
int *val, int *val2, long mask)
{
    int ret;
    u8 range_idx;
    struct bma220_data *data = iio_priv(indio_dev);
    switch (mask) {
        case IIO_CHAN_INFO_RAW:
        /* If buffer mode enabled, do not process single-channel read */
        if (iio_buffer_enabled(indio_dev))
            return -EBUSY;
        /* Else we read the channel */
        ret = bma220_read_reg(data->spi_device, chan->address);
        if (ret < 0)
            return -EINVAL;
        *val = sign_extend32(ret >> BMA220_DATA_SHIFT, 5);
        return IIO_VAL_INT;
        case IIO_CHAN_INFO_SCALE:
        ret = bma220_read_reg(data->spi_device, BMA220_REG_RANGE);
        if (ret < 0)
            return ret;
        range_idx = ret & BMA220_RANGE_MASK;
        *val = bma220_scale_table[range_idx][0];
        *val2 = bma220_scale_table[range_idx][1];
        return IIO_VAL_INT_PLUS_MICRO;
    }
    return -EINVAL;
}
```

Mỗi lần ta đọc file ```*_raw``` trong sysfs, hàm callback được gọi và truyền ```IIO_CHAN_INFO_RAW``` vào tham số mask, và truyền kênh tương ứng vào tham số ```*chan```. ```*val``` và ```*val2``` là các tham số output, chúng nên được đặt bằng các giá trị raw được đọc từ device. Tương tự mỗi lần ta đọc file ```*_scale``` trong sysfs thì sẽ truyền ```IIO_CHAN_INFO_SCALE``` vào tham số mask, các tham số các cũng được truyền tương tự như raw.

Hàm ```write_raw```() cũng tương tự như với hàm đọc:
```c
static int bma220_write_raw(struct iio_dev *indio_dev,
struct iio_chan_spec const *chan,
int val, int val2, long mask)
{
    int i;
    int ret;
    int index = -1;
    struct bma220_data *data = iio_priv(indio_dev);
    switch (mask) {
        case IIO_CHAN_INFO_SCALE:
        for (i = 0; i < ARRAY_SIZE(bma220_scale_table); i++)
        if (val == bma220_scale_table[i][0] &&
        val2 == bma220_scale_table[i][1]) {
            index = i;
            break;
        }
        if (index < 0)
        return -EINVAL;
        mutex_lock(&data->lock);
        data->tx_buf[0] = BMA220_REG_RANGE;
        data->tx_buf[1] = index;
        ret = spi_write(data->spi_device, data->tx_buf,
        sizeof(data->tx_buf));
        if (ret < 0)
            dev_err(&data->spi_device->dev, "failed to set measurement range\n");
        mutex_unlock(&data->lock);
        return 0;
    }
    return -EINVAL;
}
```

Hàm này được gọi mỗi khi ta ghi giá trị vào device, thông thường là thay đổi các tham số scale. Ví dụ
```sh
 echo <desired-scale> >
/sys/bus/iio/devices/iio;devices0/in_accel_scale .
```

Sau khi đã có các hàm đọc và ghi thì ta sẽ điền chúng vào ```struct iio_info```:
```c
static const struct iio_info bma220_info = {
    .driver_module = THIS_MODULE,
    .read_raw = bma220_read_raw,
    .write_raw = bma220_write_raw, /* Only if your driver need it */
};
```

Trong hàm ```probe```, ta sẽ cài đặt ```struct iio_dev```:
```c
/*
* We provide only two mask possibility, allowing to select none or every
* channels.
*/
static const unsigned long bma220_accel_scan_masks[] = {
    BIT(AXIS_X) | BIT(AXIS_Y) | BIT(AXIS_Z), 0
};
static int bma220_probe(struct spi_device *spi)
{
    int ret;
    struct iio_dev *indio_dev;
    struct bma220_data *data;
    indio_dev = devm_iio_device_alloc(&spi->dev, sizeof(*data));
    if (!indio_dev) {
        dev_err(&spi->dev, "iio allocation failed!\n");
        return -ENOMEM;
    }
    data = iio_priv(indio_dev);
    data->spi_device = spi;
    spi_set_drvdata(spi, indio_dev);
    mutex_init(&data->lock);
    indio_dev->dev.parent = &spi->dev;
    indio_dev->info = &bma220_info;
    indio_dev->name = BMA220_DEVICE_NAME;
    indio_dev->modes = INDIO_DIRECT_MODE;
    indio_dev->channels = bma220_channels;
    indio_dev->num_channels = ARRAY_SIZE(bma220_channels);
    indio_dev->available_scan_masks = bma220_accel_scan_masks;
    ret = bma220_init(data->spi_device);
    if (ret < 0)
        return ret;
    /* this call will enable trigger buffer support for the device */
    ret = iio_triggered_buffer_setup(indio_dev, iio_pollfunc_store_time,
    bma220_trigger_handler, NULL);
    if (ret < 0) {
        dev_err(&spi->dev, "iio triggered buffer setup failed\n");
        goto err_suspend;
    }
    ret = iio_device_register(indio_dev);
    if (ret < 0) {
        dev_err(&spi->dev, "iio_device_register failed\n");
        iio_triggered_buffer_cleanup(indio_dev);
        goto err_suspend;
    }
    return 0;
    err_suspend:
    return bma220_deinit(spi);
}
```

### Truy cập dữ liệu IIO
Như ta đã biết thì có 2 kiểu để truy cập vào dữ liệu của IIO device, một cách thông qua các file trong sysfs của từng kênh (one-shot capture), hoặc thông qua IIO character device (triggerd buffer).
#### Chế độ one-shot capture
Bằng cách đọc các file của từng kênh trong sysfs, ta sẽ chỉ capture đúng dữ liệu của kênh đó. Ví dụ:
```sh 
# cd /sys/bus/iio/devices/iio:device0
# cat in_voltage3_raw
6646
# cat in_voltage_scale
0.305175781
```

Giá trị chính xác sẽ bằng giá trị raw nhân với giá trị scale:
Voltage value : 6646 * 0.305175781 = 2028.19824053

#### Truy cập dữ liệu buffer.
Yêu cầu tiên quyết để có thể đọc được dữ liệu từ buffer là triggered buffer phải được enable trong driver. Sau đó để yêu cầu dữ liệu, ta phải tạo trigger, enable các kênh ADC, và kích hoạt trigger.

##### Capture sử dụng sysfs trigger.
Các bước để capture:
1. Tạo trigger:
```sh
# echo 0 > /sys/devices/iio_sysfs_trigger/add_trigger
```

Ở đây 0 chính là index mà ta gán cho trigger. Sau lệnh này thì một thư mục của trigger sẽ được tạo ở ```/sys/bus/iio/devices/``` , là ```trigger0``` .
2. Liên kết trigger với device: Một trigger được định danh duy nhất bằng tên của nó, ta sẽ sử dụng tên này để gán trigger cho một device. Với ```trigger0``` thì tên của nó là ```sysfstrig0``` nên:
```sh
# echo sysfstrig0 > /sys/bus/iio/devices/iio:device0/trigger/current_trigger
```

Ngoài ra ta có thể sử dụng lênh tổng quát hơn sau:
```sh
 cat /sys/bus/iio/devices/trigger0/name >
/sys/bus/iio/devices/iio:device0/trigger/current_trigger
```
3. Enable một số scan element: Bước nãy sẽ chọn một số kênh mà ta muốn đẩy dữ liệu của chúng vào buffer, ngoài ra a cũng phải chú ý đến trường ```available_scan_masks``` trong driver:
```sh
# echo 1 >
/sys/bus/iio/devices/iio:device0/scan_elements/in_voltage4_en
# echo 1 >
/sys/bus/iio/devices/iio:device0/scan_elements/in_voltage5_en
# echo 1 >
/sys/bus/iio/devices/iio:device0/scan_elements/in_voltage6_en
# echo 1 >
/sys/bus/iio/devices/iio:device0/scan_elements/in_voltage7_en
```

4. Cài đặt kích thước buffer: đây là số lượng các tập dữ liệu mà buffer có thể giữ:
```sh
# echo 100 > /sys/bus/iio/devices/iio:device0/buffer/length
```

5. Enable buffer: 
```sh
# echo 1 > /sys/bus/iio/devices/iio:device0/buffer/enable
```

6. Bật trigger để bắt đầu capture dữ liệu:
```sh
# echo 1 > /sys/bus/iio/devices/trigger0/trigger_now
```

7. Disable buffer:
```sh
# echo 0 > /sys/bus/iio/devices/iio:device0/buffer/enable
```

8. Gỡ trigger ra khỏi device:
```sh
# echo "" > /sys/bus/iio/devices/iio:device0/trigger/current_trigger
```

9. Lấy dữ liệu từ buffer trong IIO character device:
```sh
# cat /dev/iio\:device0 | xxd -
```

##### Capture dữ liệu bằng hrtimer trigger
Ta có thể capture dữ liệu bằng bộ hrtimer của kernel như sau:
```sh
# echo /sys/kernel/config/iio/triggers/hrtimer/trigger0
# echo 50 > /sys/bus/iio/devices/trigger0/sampling_frequency
# echo 1 > /sys/bus/iio/devices/iio:device0/scan_elements/in_voltage4_en
# echo 1 > /sys/bus/iio/devices/iio:device0/scan_elements/in_voltage5_en
# echo 1 > /sys/bus/iio/devices/iio:device0/scan_elements/in_voltage6_en
# echo 1 > /sys/bus/iio/devices/iio:device0/scan_elements/in_voltage7_en
# echo 1 > /sys/bus/iio/devices/iio:device0/buffer/enable
# cat /dev/iio:device0 | xxd -
0000000: 0188 1a30 0000 0000 8312 68a8 c24f 5a14 ...0......h..OZ.
0000010: 0188 1a30 0000 0000 192d 98a9 c24f 5a14 ...0.....-...OZ.
[...]
```

Lại có:
```sh
$ cat /sys/bus/iio/devices/iio:device0/scan_elements/in_voltage_type
be:s14/16>>2
```
 Ta có Voltage processing: 0x188 >> 2 = 98 * 250 = 24500 = 24.5 v

### IIO tools
Dưới đây là một số các công cụ dùng để đơn giản hóa việc phát triển một IIO device:
- ```lsiio.c```: liệt kê các IIO triggers, devices và kênh
- ```iio_event_monitor.c```: điều khiển một ioctl của IIO device  
- ```generic_buffer.c```: lấy, xử lý và in dữ liệu từ một IIO buffer.
- ```libiio```: thư viện được phát triển bởi analog device để giao tiếp với các IIO devices: https://github.com/analogdevicesinc/libiio

### Tổng kết
Ở bài viết này, ta đã làm quen với IIO framework, biết được device, trigger và kênh là gì. Ta có thể giao tiếp với chúng qua sysfs hoặc character device.



