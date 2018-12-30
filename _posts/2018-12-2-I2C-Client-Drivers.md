---
layout: post
title: I2C Client Drivers
subtitle:  I2C Client Drivers trong Linux
gh-repo: 
gh-badge: [star, fork, follow]
tags: [kernel, embedded, device driver]
---

## I2C Client Drivers
I2C (viết tắt của từ tiếng Anh "Inter-Integrated Circuit") là một loại bus nối tiếp được phát triển bởi hãng sản xuất linh kiện điện tử Philips (bây giờ là NXP). Do tính ưu việt và đơn giản của nó, I2C đã được chuẩn hóa và được dùng rộng rãi trong các module truyền thông nối tiếp của vi mạch tích hợp ngày nay.
![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/rrm9do7j4o_image.png)
I2C sử dụng hai đường truyền tín hiệu: một đường xung nhịp đồng hồ (SCL) và một đường dữ liệu (SDA). Mỗi dây SDA hãy SCL đều được nối với điện áp dương của nguồn cấp thông qua một điện trở kéo lên (pullup resistor). SCL được tạo bởi master để đồng bộ dữ liệu gửi trên đường SDA. Cả slave lẫn master đều có thể gửi dữ liệu (tất nhiên là không cùng lúc).
Đường bus được điều khiển bởi master, trong các hệ thống embedded linux thì master chính là CPU. Đường bus sẽ được sử dụng để truyền thông nối tiếp giữa CPU là EEPROM, RTC chips, GPIO expanders, cảm biến nhiệt, ....
Tốc độ clock phổ biến từ 10KHz đến 100KHz, và 400KHz đến 2MHz. Ta sẽ không đề cập chi tiết đến thông số đường bus cũng như bus drivers (driver điều khiển bus). Trong bài viết này ta sẽ nghiên cứu về client driver, với mục đích điều khiển các thiết bị slave nằm trên bus I2C. Các vấn đề sẽ được đề cập bao gồm:
- Kiến trúc I2C client driver.
- Truy cập device để đọc/ghi dữ liệu.
- Khai báo client trong Device Tree.

### Kiến trúc driver

Xin nhắc lại là các bus vật lý mà các device nằm trên được gọi là bus controller. Các driver điều khiển các bus này còn được gọi là controller driver. Các driver điều khiển device nằm trên bus được gọi là protocol driver (trong trường hợp I2C hoặc SPI bus thì được gọi là client driver). Khi một device nằm trên bus controller, protocol driver của nó phải phụ thuộc vào controller driver. Controller driver có nhiệm vụ chia sẻ quyền truy cập bus với nhiều device, hay tổng quát hơn là cung cấp một layer trừu tượng nằm giữa device và bus. Khi ta thực hiện một thao tác (đọc hoặc ghi) trên I2C bus, thì controller driver sẽ đảm đương nhiệm vụ này một cách âm thầm ở background. Các controller driver sẽ cung cấp một tập các hàm để driver cho các device nằm trên bus có thể gọi. 
Tóm tắt lại, có 2 loại driver là controller driver và protocol driver, controller điều khiển bus, protocol điều khiển device. Controller driver xuất ra các hàm để protocol driver có thể sử dụng để  cấu hình hoặc tháo tác với device. Khi protocol họi các hàm này, bằng một cách nào đó controller driver sẽ điều khiển bus để thực hiện chức năng. Điều trên đúng với không chỉ I2C bus mà còn với SPI, USB, PCI, SDIO bus, ...

I2C protocol driver được dại diện trong kernel bằng ```struct i2c_driver```. I2C client device được đại diện bằng ```struct i2c_client```. 

#### Struct i2c_driver
Struct ```i2c_driver``` được định nghĩa như sau:
```c
struct i2c_driver {
    /* Standard driver model interfaces */
    int (*probe)(struct i2c_client *, const struct i2c_device_id *);
    int (*remove)(struct i2c_client *);
    /* driver model interfaces that don't relate to enumeration */
    void (*shutdown)(struct i2c_client *);
    struct device_driver driver;
    const struct i2c_device_id *id_table;
};
```

##### Hàm probe()
Hàm ```probe``` là một phần của struct ```i2c_driver```, và nó được thực hiện khi xảy ra match giữa một i2c device và i2c driver. Các nhiệm vụ cần thiết của hàm ```probe()``` là:
- Kiểm tra xem device được match có đúng là device mong muốn hay không.
- Kiểm tra chức năng I2C bus controller bằng cách sử dụng hàm  ```i2c_check_functionality```.
- Khởi tạo device.
- Thiết lập thông số device.
- Đăng kí kernel framework.

Nguyên mẫu hàm probe như sau:
```c
static int foo_probe(struct i2c_client *client, const struct i2c_device_id *id)
```

Các tham số bao gồm:
- ```struct i2c_client *client```: con trỏ tới i2c client, đại diện cho i2c device. Tham số này được kernel tự động điền vào hàm.
- ```const struct i2c_device_id *id```: con trỏ tới I2C device ID entry. 

**Dữ liệu riêng cho mỗi device**
Hệ thống I2C cung cấp cho ta khả năng lưu trữ một con trỏ để trỏ đến bất kì cấu trúc dữ liệu nào mà ta muốn, và đây được coi là dữ liệu riêng cho device đó. 
Để lưu và lấy dữ liệu ta sẽ sử dụng các hàm:
```c
/* set the data */
void i2c_set_clientdata(struct i2c_client *client, void *data);
/* get the data */
void *i2c_get_clientdata(const struct i2c_client *client);
```

Bản chất hai hàm trên sẽ gọi ```dev_set_drvdata``` và ```dev_get_drvdata``` để cập nhật và lấy giá trị của trường ```void *driver_data``` của struct device bên trong ```struct i2c_client```.

Ví dụ dưới đây minh họa cho việc sử dụng những dữ liệu riêng cho các device:
```c
/* This is the device specific data structure */
struct mc9s08dz60 {
    struct i2c_client *client;
    struct gpio_chip chip;
};

static int mc9s08dz60_probe(struct i2c_client *client, const struct i2c_device_id *id)
{
    struct mc9s08dz60 *mc9s;
    if (!i2c_check_functionality(client->adapter,
    I2C_FUNC_SMBUS_BYTE_DATA))
    return -EIO;
    mc9s = devm_kzalloc(&client->dev, sizeof(*mc9s), GFP_KERNEL);
    if (!mc9s)
        return -ENOMEM;
    [...]
    mc9s->client = client;
    i2c_set_clientdata(client, mc9s);

    return gpiochip_add(&mc9s->chip);
}
```

##### Hàm remove()

Nguyên mẫu hàm``` remove()``` đơn giản như sau:
```c
static int foo_remove(struct i2c_client *client)
```

Hàm ```remove()``` cũng được truyền vào một tham số  là ```struct i2c_client *client``` giống như hàm ```probe()``` (việc truyền tham số này do kernel đảm nhận). Nhiệm vụ của hàm ```remove()``` là giải phóng những gì hàm ```probe()``` cấp phát, bao gồm những dữ liệu riêng cho device:
```c
static int mc9s08dz60_remove(struct i2c_client *client)
{
    struct mc9s08dz60 *mc9s;
    /* We retrieve our private data */
    mc9s = i2c_get_clientdata(client);
    /* Wich hold gpiochip we want to work on */
    return gpiochip_remove(&mc9s->chip);
}
```

#### Khởi tạo và đăng kí driver I2C

Như ở bài Platform Driver, kernel cung cấp cho chúng ta mộ họ hàm là ```module_*_driver```, dành cho việc, đăng kí/hủy đăng kí driver với hệ thống trong các hàm init/exit. Với I2C thì hàm được sử dụng là:
```c
module_i2c_driver(foo_driver);
```

#### Driver và device provisioning
Như đã được biết về cơ chế match giữa driver và driver, ta cần cấp mảng ```device_id``` để thông báo những device mà driver có thể điều khiển. Với các I2C device thì mảng struct là ```i2c_device_id```. Struct này được định nghĩa như sau:
```c
struct i2c_device_id {
    char name[I2C_NAME_SIZE];
    kernel_ulong_t  driver_data;
};
```
Để hệ thống I2C biết về những device cần điều khiển thì ta sử dụng macro ```MODULE_DEVICE_TABLE``` để thông báo. 
VÍ dụ:
```c
static struct i2c_device_id foo_idtable[] = {
    { "foo", my_id_for_foo },
    { "bar", my_id_for_bar },
    { }
};
MODULE_DEVICE_TABLE(i2c, foo_idtable);
static struct i2c_driver foo_driver = {
    .driver = {
        .name = "foo",
    },
    .id_table = foo_idtable,
    .probe  = foo_probe,
    .remove = foo_remove,
}
```

### Giao tiếp với client.
Bản chất giao tiếp I2C chỉ là việc thao tác với các thanh ghi (ghi/đọc thông tin).  Ta sẽ nhắc lại một chút lý thuyết về các thao tác đọc/ghi của I2C.
- Thao tác ghi thông tin vào slave trên I2C bus: Để ghi dữ liệu trên I2C bus, master sẽ gửi một bit start + địa chỉ slave + bit kết thúc (đặt bằng 0 để thông báo thao tác ghi). Sau khi slave gửi tín hiệu nhận biết ACK, master gửi địa chỉ thanh ghi mà nó muốn ghi dữ liệu vào. Sau đó slave sẽ thông báo nhận biết lại bằng tín hiệu ACK để master biết nó đã sẵn sàng. Master sẽ bắt đầu gửi dữ liệu thanh ghi cho slave rồi kết thúc bằng bit stop.
![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/j5ye5d7qxx_image.png)
- Thao tác đọc thông tin từ slave trên I2C bus: Thao tác đọc cũng gần giống với thao tác ghi nhưng phức tạp hơn một chút. Để đọc được dữ liệu từ slave, master đầu tiên sẽ thực hiện thao tác ghi để thông báo địa chỉ slave + địa chỉ thanh ghi cần đọc cho slave. Tuy nhiên ngay sau thao tác ghi, master gửi lại bit start + theo địa chỉ của slave + bit kết thúc(đặt bằng 1 để thông báo thao tác đọc), ngay sau bit ACK thì slave sẽ gửi dữ liệu ở thanh ghi mà master mong muốn. Master khi nhận đủ dữ liệu mong muốn sẽ thông báo bằng bit NACK và kết thúc thao bằng bằng bit STOP.
![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/l2dulx6ziq_image.png)


Hệ thống I2C trong linux cung cấp cho ta 2 loại API, loại thứ nhất là loại truyền thống, dành riêng cho I2C, loại thứ 2 sử dụng giao tiếp SMBUS device, loại này tương thích với giao tiếp I2C thông thường.

#### Giao tiếp I2C truyền thống
Ta sử dụng 2 hàm dưới đây để giao tiếp với các I2C device:
```c
int i2c_master_send(struct i2c_client *client, const char *buf, int count);
int i2c_master_recv(struct i2c_client *client, char *buf, int count);
```

Hầu hết các hàm giao tiếp I2C đều phải lấy ```struct i2c_client``` là tham số để biết được chính xác phải giao tiếp với I2C device nào. Tham số thứ 2 chứa những byte cần đọc hoặc ghi vào, tham số thứ 3 là số byte để đọc hoặc ghi. Giống như hầu hết các hàm đọc/ghi khác, giá trị trả về của hàm là số byte được đọc/ghi thành công. 
Ngoài ra Linux còn cung cấp ta hàm:
```c
int i2c_transfer(struct i2c_adapter *adap, struct i2c_msg *msg,
int num);
```

Hàm này có thể thực hiện cả thao tác đọc hoặc ghi, bằng cách sử dụng bản tin có cấu trúc:
```c
struct i2c_msg {
__u16 addr;         /* địa chỉ slave */
__u16 flags;        /* Message flags */
__u16 len;          /* Độ dài bản tin */
__u8 *buf;          /* Con trỏ đến dữ liệu bản tin */
};
```

Để hiểu hơn về hàm ```i2c_transfer``` này, ta xem ví dụ:
```c
ssize_t eep_read(struct file *filp, char __user *buf, size_t count, loff_t *f_pos)
{
    [...]
    int _reg_addr = dev->current_pointer;
    u8 reg_addr[2];
    reg_addr[0] = (u8)(_reg_addr>> 8);
    reg_addr[1] = (u8)(_reg_addr& 0xFF);

    struct i2c_msg msg[2];
    msg[0].addr = dev->client->addr;
    msg[0].flags = 0;                   /* Write */
    msg[0].len = 2;                     /* Address is 2bytes coded */
    msg[0].buf = reg_addr;

    msg[1].addr = dev->client->addr;
    msg[1].flags = I2C_M_RD;            /* We need to read */
    msg[1].len = count;
    msg[1].buf = dev->data;
    if (i2c_transfer(dev->client->adapter, msg, 2) < 0)
        pr_err("ee24lc512: i2c_transfer failed\n");
    if (copy_to_user(buf, dev->data, count) != 0) {
        retval = -EIO;
    goto end_read;
    }
    [...]
}
```

Như ta đã biết ở phần lý thuyết, để đọc được dữ liệu từ slave, master đầu tiên sẽ thực hiện thao tác ghi để thông báo địa chỉ slave + địa chỉ thanh ghi cần đọc cho slave. Ngay sau thao tác ghi, master chỉ gửi địa chỉ của slave thì slave sẽ gửi dữ liệu ở thanh ghi mà master mong muốn. 

Trường flags của ```struct i2c_msg``` sẽ là ```I2C_M_RD``` nếu cần đọc và là ```0``` nếu cần ghi.

#### Các hàm tương thích SMBus
SMBus là các loại bus 2 dây được phát triển bởi Intel, nó tương tự như I2C. Các I2C device tương thích với chuẩn SMBus, tuy nhiên các SMBus device thì lại không tương thích ngược với I2C bus. Do đó ta có thể sử dụng các hàm SMBus để viết driver cho cả I2C device lẫn SMbus device.

Một số các API của SMBus:
```c
s32 i2c_smbus_read_byte_data(struct i2c_client *client, u8 command);
s32 i2c_smbus_write_byte_data(struct i2c_client *client,
u8 command, u8 value);
s32 i2c_smbus_read_word_data(struct i2c_client *client, u8 command);
s32 i2c_smbus_write_word_data(struct i2c_client *client,
u8 command, u16 value);
s32 i2c_smbus_read_block_data(struct i2c_client *client,
u8 command, u8 *values);
s32 i2c_smbus_write_block_data(struct i2c_client *client,
u8 command, u8 length, const u8 *values);
```

Việc sử dụng các API của SMBus sẽ đơn giản hóa việc đọc và ghi dữ liệu lên các thanh ghi ở các I2C device. Ở lệnh read thì ```command``` chính là địa chỉ thanh ghi và giá trị trả về là giá trị thanh ghi cần đọc. Với lệnh write thì sẽ thực hiện ghi giá trị ```value``` vào thanh ghi có địa chỉ ```command```. 

Ví dụ:
```c
struct mcp23016 {
    struct i2c_client *client;
    struct gpio_chip chip;
    struct mutex lock;
};
[...]
/* This function is called when one needs to change a gpio state */
static int mcp23016_set(struct mcp23016 *mcp, unsigned offset, intval)
{
    s32 value;
    unsigned bank = offset / 8 ;
    u8 reg_gpio = (bank == 0) ? GP0 : GP1;
    unsigned bit = offset % 8 ;
    value = i2c_smbus_read_byte_data(mcp->client, reg_gpio);
    if (value >= 0) {
        if (val)
            value |= 1 << bit;
        else
            value &= ~(1 << bit);
        return i2c_smbus_write_byte_data(mcp->client, reg_gpio, value);
    } else
        return value;
}
[...]
```

#### Khởi tạo I2C device trong file cấu hình board (file code C - cách cũ)
Có một điều chắc chắn là ta luôn phải thông báo cho kernel về sự xuất hiện của các thiết bị vật lý trong hệ thống. Như đã biết thì có hai kiểu khai báo device trong kernel: dùng file cấu hình board code c, device tree. Với kiểu khai báo trong code C thì ta sử dụng ```struct i2c_board_info``` để đại diện cho một I2C device.
Struct này được định nghĩa:
```c
struct i2c_board_info {
    char type[I2C_NAME_SIZE];
    unsigned short addr;
    void *platform_data;
    int irq;
};
```

trong đó thì type nên được đặt giống với trường ```i2c_driver.driver.name```. Sau đó ta sẽ tạo một mảng các ```i2c_board_info``` và truyền mảng đó như một parameter vào hàm ```i2c_register_board_info```:
```c
int i2c_register_board_info(int busnum, struct i2c_board_info const *info,
unsigned len)
```

Tuy nhiên do các khai báo này là cũ và không được khuyên dùng nữa nên chúng ta chỉ tìm hiểu nó đến đây. Về sau ta sẽ chủ đạo tập trung vào device tree.

### I2C và device tree

Để cấu hình và điều khiển được các I2C device, ta phải làm 2 bước:
- Định nghĩa và đăng kí I2C driver
- ĐỊnh nghĩa và đăng kí I2C device

I2C device thuộc vào kiểu device không memory mapped trong Device Tree, I2C bus thì là bus có đánh địa chỉ ở đó thuộc tính reg trong node là địa chỉ của device trên bus. Các I2C device node chính là các con của bus mà chúng nằm trên. Mỗi device chỉ nhận một địa chỉ. Ví dụ:
```dts
 &i2c2 { /* Phandle of the bus node */
    pcf8523: rtc@68 {
        compatible = "nxp,pcf8523";
        reg = <0x68>;
    };
eeprom: ee24lc512@55 { /* eeprom device */
    compatible = "packt,ee24lc512";
    reg = <0x55>;
    };
};
```

#### Định nghĩa và đăng kí I2C driver

Trước hết ta cần phải định nghĩa một ```struct of_device_id``` tương ứng với node của device trong device tree. Ví dụ:
```c
/* no extra data for this device */
static const struct of_device_id foobar_of_match[] = {
    { .compatible = "packtpub,foobar-device" },
    {}
};
MODULE_DEVICE_TABLE(of, foobar_of_match);
```

Sau đó ta định nghĩa ```i2c_driver``` như sau:
```c
static struct i2c_driver foo_driver = {
    .driver = {
    .name = "foo",
    .of_match_table = of_match_ptr(foobar_of_match), /* Only this line is
    added */
    },
    .probe = foo_probe,
    .id_table = foo_id,
};
```

#### Chú ý

Với những phiên bản kernel cũ hơn 4.10, trong hàm ```i2c_device_probe()``` (đây là hàm được gọi mỗi khi một I2C device đăng kí với I2C core) có viết:
```c
if (!driver->probe || !driver->id_table)
return -ENODEV;
```

Điều này có nghĩa là cho dù ta không cần dùng đến .```id_table``` (trường dùng để  đăng kí mảng các ```i2c_device_id``` cho ```i2c_driver```, theo kiểu match với file board .c), ta vẫn phải khai báo chúng. Nghĩa là dù ta match theo kiểu OF (dùng device tree), ta vẫn không thể loại bỏ .```id_table```. Vấn này đã được cải thiện sau phiên bản kernel 4.10. Nên người lập trình cần hết sức chú ý. Mình cũng đã một lần mắc phải lỗi này khi lập trình với kernel 4.3 mà chủ quan không khai báo .```id_table```, và kết quả là hàm ```probe()``` không thể được gọi.

### Tổng hợp thành quả

Tổng hợp các bước cần thiết sau để viết một I2C driver:
1. Khai báo mảng các device id được hỗ trợ bởi driver. Sử dụng mảng các i2c_device_id để match theo kiểu cũ, sử dụng mảng các ```of_device_id``` để match với device tree.
2. Gọi hàm ```MODULE_DEVICE_TABLE(i2c, my_id_table)``` và ```MODULE_DEVICE_TABLE(of, your_of_match_table)``` để đăng kí danh sách các device với I2C core và OF core.
3. Viết hàm ```probe``` và ```remove```. Trong hàm ```probe```, ta phải xác định device được match, cấu hình và định nghĩa các dữ liệu riêng của device rồi đăng kí nó với các kernel framework khác. Hàm ``remove`` sẽ undo tất các những gì đã làm trong hàm ```probe```.
4. Khai báo và điền ```struct i2c_driver``` và đặt trường ```id_table``` với mảng các ```i2c_device_id``` ta đã tạo ra trước đó. 
5. Gọi các macro ```module_i2c_driver``` để đăng kí driver với kernel. Ví dụ ```module_i2c_driver(serial_eeprom_i2c_driver)```.


_Tham khảo từ Linux Device Drivers Development_



