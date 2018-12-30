---
layout: post
title: Device Tree trong Linux
subtitle:  Device Tree trong Linux
gh-repo: 
gh-badge: [star, fork, follow]
tags: [kernel, embedded, device driver]
---

## Device Tree trong Linux

Device Tree (DT) là một file mô tả phần cứng, có kiểu định dạng giống JSON, nó mô tả một cấu trúc cây, ở đó thì các device được đại diện bằng các node. Mỗi node có các thuộc tính, các thuộc tính có thể được gán giá trị hoặc để trống. 
![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/albz3w6xon_image.png)
Device Tree xuất thân từ Open Firmware, là một chuẩn được công nhận bởi nhiều công ty và có mục đính chính là định nghĩa giao tiếp cho các hệ firmware trên máy tính. Có thể tìm hiểu chi tiết về DT ở http://www.devicetree.org/. Bài viết sẽ đề cấp đến những điều cơ bản của DT, bao gồm:
- Quy ước đặt tên, alias và gán nhãn.
- Mô tả kiểu dữ liệu và các API.
- Quản lý địa chỉ và truy cập tài nguyên của device.
- Triển khai kiểu match OF.

### Cơ chế device tree
Ta enable device tree bằng cách đặt option ```CONFIG_OF``` thành ```Y```. Các header cho phép sử dụng các API của DT trong driver là ```<linux/of.h>``` và ```<linux/of_device.h>```

Dưới đây là một mô tả node mẫu:
![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/cwrj2e7d9j_image.png)

Một số kiểu dữ liệu sử  dụng trong DT bao gồm:
- Chuỗi, nằm trong dấu ngoặc kép. Ta có thể sử dụng dấu phẩy để tạo một danh sách các chuỗi.
- Số nguyên dương 32-bit, nằm trong dấu ngoặc nhọn.
- Dữ liệu boolean, là một thuộc tính trống. Nếu nó được khai báo trong device tree tức giá trị là ```true```, nếu không được khai báo thì tức là ```false```.

#### Quy ước đặt tên
Mỗi node được biểu diễn dưới dạng ```<name>[@address]```, với ```<name>``` là chuỗi có độ dài lên tới 31 kí tự, và ```[@address]``` là địa chỉ được sử đụng để truy cập vào node. ```<@address>``` có thể có hoặc không. 
Ví dụ:
```dts
expander@20 {
compatible = "microchip,mcp23017";
reg = <20>;
[...]
};
```

hoặc
```dts
i2c@021a0000 {
compatible = "fsl,imx6q-i2c", "fsl,imx21-i2c";
reg = <0x021a0000 0x4000>;
[...]
};
```

Ngoài ra, label cũng có thể có hoặc không. Ta chỉ cần gán nhãn cho node nếu có ý định tham chiếu nó làm thuộc tính của một node khác. 

#### Alias, lable và và phandle
Device tree dưới đây sẽ giải thích về cách chúng hoạt động:
```dts
aliases {
    ethernet0 = &fec;
    gpio0 = &gpio1;
    gpio1 = &gpio2;
    mmc0 = &usdhc1;
[...]
};
gpio1: gpio@0209c000 {
    compatible = "fsl,imx6q-gpio", "fsl,imx35-gpio";
    [...]
};
node_label: nodename@reg {
    [...];
    gpios = <&gpio1 7 GPIO_ACTIVE_HIGH>;
};
```

Lable là cách để định danh node bằng một cái tên duy nhất. Thực tế, tên này sẽ được chuyển thành một giá trị 32-bit duy nhất bằng DT compiler. Ở ví dụ trên, ```gpio1``` và ```node_label``` là các label(nhãn). Do nhãn là duy nhất với một node nên được dùng để chỉ định node.

Phandle (pointer handle) là một giá trị 32-bit liên kết với một node, và được sử đụng để định danh một node để có có thể tham chiếu đến node đó từ một thuộc tính của node khác. Bằng cách sử dụng ```<&mylabel>```, ta có thể trỏ đến một node có tên là mylabel. Ở ví dụ trên,``` &gpio1``` là một phandle trỏ tới node gpio1.

Để kernel có thể rà soát hết cả cây để tìm một node thì khái niệm alias được ra đời. Alias không được sử dụng trực tiếp trong DT nhưng được Linux kernel sử dụng. Ta có thể sử dụng hàm ```find_node_by_alias() ```và truyền vào alias để tìm tới một node. 

#### Trình biên dịch device tree (DT Compiler)
![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/wusthnw22o_image.png)
Device tree có hai dạng chính: dạng source code như ta đã biết hay còn gọi là DTS và có đuôi là ``.dts``; dạng đã được biên dịch hay còn gọi là DTB và có đuôi là ```.dtb```. Ngoài ra source code cũng có thể có đuôi là ```.dtsi```, đại diện cho DT ở mức SoC, và được include vào một file ```.dts``` (Giống như source code C ```.c``` include header ```.h```). 

Để biên dịch được file source ```.dts``` sang file ```.dtb```, ta sử dụng DT compiler (dtc). Để biên dịch tất cả các ```.dts``` file cho một board ARM:
```bash
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make dtbs
```

Hoặc chỉ biên dịch một file DT imx6dl-sabrelite.dts:
```bash 
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make imx6dl-sabrelite.dtb
```

Việc biên dịch ngược từ .dtb sang .dts cũng thực hiện được:
```bash
dtc -I dtb -O dtsarch/arm/boot/dts imx6dl-sabrelite.dtb
>path/to/my_devicetree.dts
```

### Định địa chỉ device.
Mỗi device được gán vào ít nhất một node trên DT. Có một số thuộc tính thông thường mà loại thiết bị nào cũng có, đặc biệt là các thiết bị nằm trên bus mà kernel biết (như SPI, I2C, platform, MDIO...). Các thuộc tính đó là ```reg```, ```#address-cells``` và ```#size-cells```. Mục đích của 3 thuộc tính này là định địa chỉ của chúng trên bus. Trong đó thuộc tính địa chỉ là ```reg```. ```#size-cells``` chỉ ra có bao nhiêu 32-bit cell được sử dụng để đaị diện cho kích thước địa chỉ của device. ```#address-cells``` chỉ ra có bao nhiêu 32-bit cell được dùng để đại diện cho địa chỉ. 

Thuộc tính ```reg``` là một danh sách các cặp giá trị có dạng ```reg = <adress0 size0 [address1 size1] [address2 size2] ...>```, mỗi cặp giá trị address và cell đại diện cho vùng địa chỉ mà device sử dụng.

Các device sẽ thừa kế ```#size-cells``` và ```#address-cells``` của các node cha (thông thường là các node đại diện cho bus controller). Thực tế hai thông số này không ảnh hưởng đến node khai báo nó mà chỉ ảnh hưởng đến các node con của nó. Nghĩa là trước khi khai báo thuộc tính ```reg``` của một node thì phải biết được hai thuộc tính ```#address-cells``` và ```#size-cells``` của node cha.

#### Định địa chỉ SPI và I2C

SPI và I2C device là các loại device non-memory mapped, vì CPU không thể truy cập được địa chỉ của chúng. Thay vào đó, driver của device cha (bus controller driver) sẽ thực hiện truy cập. Mỗi I2C, SPI device được đại diện bằng một node con của I2C/SPI bus node. Do là các device non-memory mapped, thuộc tính ```#size-cells``` là 0. 
Ví dụ:
```dts
&i2c3 {
    [...]
    status = "okay";
    temperature-sensor@49 {
        compatible = "national,lm73";
        reg = <0x49>;
    };
    pcf8523: rtc@68 {
        compatible = "nxp,pcf8523";
        reg = <0x68>;
    };
};
&ecspi1 {
    fsl,spi-num-chipselects = <3>;
    cs-gpios = <&gpio5 17 0>, <&gpio5 17 0>, <&gpio5 17 0>;
    status = "okay";
    [...]
    ad7606r8_0: ad7606r8@1 {
        compatible = "ad7606-8";
        reg = <1>;
        spi-max-frequency = <1000000>;
        interrupt-parent = <&gpio4>;
        interrupts = <30 0x0>;
        convst-gpio = <&gpio6 18 0>;
    };
};
```

Thuộc tính ```reg``` của I2C device chính là địa chỉ của device trên đường bus, còn với SPI device thì ```reg``` đại diện cho chỉ số của đường chip-select của device mà controller node sở hữu. Ví dụ với ad7606r8 ADC có chip-select là 1, sẽ tương ứng với ```<&gpio5 17 0>``` trong cs-gpios của controller node (ecspi1).

#### Định địa chỉ platform device
Với các memory-mapped device thì vùng nhớ của chúng có thể được truy cập bởi CPU, do đó thuộc tính reg định nghĩa địa chỉ của device, địa chỉ này là danh sách các vùng nhớ mà ta có thể truy cập vào thiết bị. Mỗi vùng nhớ được đại diện bằng một cặp giá trị base và length, base là địa chỉ cơ sở vùng nhớ, length là kích thước vùng nhớ: ```reg = <base0 length0 [base1 length1] [address2 length2] ... >```

Ví dụ:
```dts
soc {
    #address-cells = <1>;
    #size-cells = <1>;
    compatible = "simple-bus";
    aips-bus@02000000 { /* AIPS1 */
        compatible = "fsl,aips-bus", "simple-bus";
        #address-cells = <1>;
        #size-cells = <1>;
        reg = <0x02000000 0x100000>;
        [...];
        spba-bus@02000000 {
            compatible = "fsl,spba-bus", "simple-bus";
            #address-cells = <1>;
            #size-cells = <1>;
            reg = <0x02000000 0x40000>;
            [...]
            ecspi1: ecspi@02008000 {
                #address-cells = <1>;
                #size-cells = <0>;
                compatible = "fsl,imx6q-ecspi", "fsl,imx51-ecspi";
                reg = <0x02008000 0x4000>;
                [...]
            };
            i2c1: i2c@021a0000 {
                #address-cells = <1>;
                #size-cells = <0>;
                compatible = "fsl,imx6q-i2c", "fsl,imx21-i2c";
                reg = <0x021a0000 0x4000>;
                [...]
            };
        };
    };
};
```

### Kiểm soát tài nguyên

Mục đích chính của driver là điều khiển và quản lý device và thực hiện các chức năng mà user-space yêu cầu. Để thực hiện được mục đích đó thì driver trước tiên phải lấy được những thông số cấu hình, đặc biệt là tài nguyên như vùng nhớ, ngắt, DMA chanel, clock,...

Ta sẽ làm việc với ví dụ dưới đây. Đây là một node i.MX6 UART, định nghĩa ở ```arch/arm/boot/dts/imx6qdl.dtsi```:
```dts
uart1: serial@02020000 {
    compatible = "fsl,imx6q-uart", "fsl,imx21-uart";
    reg = <0x02020000 0x4000>;
    interrupts = <0 26 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clks IMX6QDL_CLK_UART_IPG>,
    <&clks IMX6QDL_CLK_UART_SERIAL>;
    clock-names = "ipg", "per";
    dmas = <&sdma 25 4 0>, <&sdma 26 4 0>;
    dma-names = "rx", "tx";
    status = "disabled";
};
```

#### Tài nguyên được đặt tên 
Khi driver yêu cầu một danh sách các tài nguyên thì ta không thể đảm bảo rằng danh sách đó được sắp xếp theo thứ tự mong muốn (do người viết device tree thường không phải là người viết driver). Để tránh việc phải đảm bảo thứ tự khi viết driver, ta tương ứng định nghĩa danh sách tên cho tài nguyên theo thứ tự trong device tree. 
Ví dụ về một node device giả:
```dts
fake_device {
    compatible = "packt,fake-device";
    reg = <0x4a064000 0x800>, <0x4a064800 0x200>, <0x4a064c00 0x200>;
    reg-names = "config", "ohci", "ehci";
    interrupts = <0 66 IRQ_TYPE_LEVEL_HIGH>, <0 67 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-names = "ohci", "ehci";
    clocks = <&clks IMX6QDL_CLK_UART_IPG>, <&clks IMX6QDL_CLK_UART_SERIAL>;
    clock-names = "ipg", "per";
    dmas = <&sdma 25 4 0>, <&sdma 26 4 0>;
    dma-names = "rx", "tx";
};
```

Từ đó driver có thể lấy được giá trị của các tài nguyên :
```c
struct resource *res1, *res2;
res1 = platform_get_resource_byname(pdev, IORESOURCE_MEM, "ohci");
res2 = platform_get_resource_byname(pdev, IORESOURCE_MEM, "config");

struct dma_chan *dma_chan_rx, *dma_chan_tx;
dma_chan_rx = dma_request_slave_channel(&pdev->dev, "rx");
dma_chan_tx = dma_request_slave_channel(&pdev->dev, "tx");

int txirq, rxirq;
txirq = platform_get_irq_byname(pdev, "ohci");
rxirq = platform_get_irq_byname(pdev, "ehci");

struct clk *clck_per, *clk_ipg;
clk_ipg = devm_clk_get(&pdev->dev, "ipg");
clk_ipg = devm_clk_get(&pdev->dev, "pre");
```

#### Truy cập thanh ghi
Driver có thể lấy vùng nhớ và map nó vào không gian địa chỉ ảo. Ví dụ
```c
struct resource *res;
void __iomem *base;
res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
/*
* Here one can request and map the memory region
* using request_mem_region(res->start, resource_size(res), pdev->name)
* and ioremap(iores->start, resource_size(iores)
*
* These function are discussed in chapter 11, Kernel Memory Management.
*/
base = devm_ioremap_resource(&pdev->dev, res);
if (IS_ERR(base))
return PTR_ERR(base);
```

```platform_get_resource()``` sẽ đặt trường ```start``` và ```end``` của ```struct res``` dựa theo vùng nhớ mà ta đã gán cho ```reg``` trong device tree. Ví dụ với 
```reg = <0x02020000 0x4000>``` thì hàm ```platform_get_resource()``` sẽ đặt ```res.start = 0x02020000``` và ```res.end = 0x02023fff```.

### Kiểm soát ngắt
Giao tiếp với ngắt được chia thành 2 phần chính: phần điều khiển (controller) và phần thực thi (consumer). Có bốn thuộc tính được sử dụng để mô tả ngắt trong DT.
Phần controller là loại device đề xuât IRQ line cho phần consumer. Phần controller có 2 thuộc tính:
- ```interrupt-controller```: đây là một thuộc tính boolean, nếu được khai báo, có nghĩa device là controller.
- ```#interrupt-cells```: định nghĩa có bao nhiêu cell được sử dụng để khai báo ngắt cho controller. 

Phần consumer là device tạo ra IRQ, có 2 thuộc tính:
- ```interrupt-parent```: với những node tạo ngắt, đây là thuộc tính sẽ chứa một phandle đến phần controller mà nó được gán vào. 
- ```interrupts```: Khai báo ngắt.

Với trường hợp của i.MX6, interrupt controller là Global Interrupt Controller (GIC). 
#### Interrupt handler
Kiểm soát ngắt bao gồm việc lấy IRQ number từ Device Tree, map nó vào Linux IQR và đăng kí hàm callback cho nó. Code cho driver sẽ khá đơn giản;
```c
int irq = platform_get_irq(pdev, 0);
ret = request_irq(irq, imx_rxint, 0, dev_name(&pdev->dev), sport);
```

Hàm ```platform_get_irq()``` sẽ trả về số hiệu ngắt, số này có thể được sử dụng bởi hàm ```devm_request_irq()```. Tham số thứ 2 được đặt là 0 có nghĩa là ta sẽ lấy ngắt đầu tiên trong device node. Nếu có nhiều ngắt thì ta có thể thay đổi tham số này thành chỉ số của ngắt cần thiết, hoặc sử dụng các tài nguyên được đặt tên.

Trong ví dụ trước, device node chứa một khai báo ngắt là:
```dts
interrupts = <0 66 IRQ_TYPE_LEVEL_HIGH>;
```

Vì ngắt có phần controller là GIC nên tham số thứ nhất (trong ví dụ là 0) trong khai báo ngắt trên sẽ chỉ ra kiểu interrupt:
- Bằng 0 nếu là Shared peripheral interrupt (SPI), đây là các tín hiệu ngắt được chia sẻ giữa các core.
- Bằng 1 nếu là Private peripheral interrupt (PPI), đây là tín hiệu ngắt riêng cho một core. 

Tham số thứ hai trong khai báo ngắt (trong ví dụ là 66) là số hiệu ngắt. Số ngắt này phụ thuộc vào kiểu ngắt là PPI hay là SPI. 

Tham số thứ 3, trong ví dụ là ```IRQ_TYPE_LEVEL_HIGH```, chỉ ra mức ngắt.

### 3. Lấy các dữ liệu cụ thể khác của một device.
Như ta đã biết, có một số thuộc tính của device mà ta chỉ cần khai báo đúng tên thì code driver có thể lấy được những thuộc tính đó bằng các hàm cho sẵn. Tuy nhiên nếu ta muốn ta muốn khai báo một số tham số đặc trưng khác của device thì như thế nào. 
Ví dụ dưới đây sẽ khai báo một device node với các dữ liệu cụ thể khác:
```dts
node_label: nodename@reg{
    string-property = ""a string"";
    string-list = ""red fish"", ""blue fish"";
    one-int-property = <197>; /* One cell in this property */
    int-list-property = <0xbeef 123 0xabcd4>;
    /* each number (cell) is 32
    * bit integer(uint32). There
    * are 3 cells in this property
    */
    mixed-list-property = "a string", <0xadbcd45>, <35>, [0x01 0x23 0x45]
    byte-array-property = [0x01 0x23 0x45 0x67];
    one-cell-property = <197>;
    boolean-property;
};
```

#### Chuỗi
Ta có thể khai báo một chuỗi như sau:
```c
string-property = "a string";
```

và dùng hàm of_property_read_string() để đọc chuỗi đó:
```c
const char *my_string = NULL;
of_property_read_string(pdev->dev.of_node, "string-property", &my_string);
```

Nguyên mẫu của hàm là:
```c
int of_property_read_string(const struct device_node *np, const
char *propname, const char **out_string)
```

#### Số nguyên 32 bit và mảng
Với các thuộc tính là số nguyên và mảng số nguyên:
```dts
one-int-property = <197>;
int-list-property = <1350000 0x54dae47 1250000 1200000>;
```

ta có tể sử dụng hàm ```of_property_read_u32()``` và ```of_property_read_u32_array()``` tương ứng để đọc các giá trị này.
Nguyên mẫu của hai hàm:
```c
int of_property_read_u32(const struct device_node *np,
const char *propname, u32 index, u32 *out_value)

int of_property_read_u32_array(const struct device_node *np,
const char *propname, u32 *out_values, size_tsz);
```

Ví dụ:
```c
unsigned int number;
of_property_read_u32(pdev->dev.of_node, "one-cell-property", &number);
```

và
```c
unsigned int cells_array[4];
if (of_property_read_u32_array(pdev->dev.of_node,"int-list-property",cells_array, 4)) {
    dev_err(&pdev->dev, "list of cells not specified\n");
    return -EINVAL;
}
```

#### Boolean
Ta sử dụng hàm ```of_property_read_bool()``` để đọc các thuộc tính boolean từ device tree. Ví dụ:
```c
bool my_bool = of_property_read_bool(pdev->dev.of_node,"boolean-property");
if(my_bool){
    /* boolean is true */
} else
    /* Bolean is false */
}
```

#### Các node con

Ta có thể thêm bất kì một node con vào một device node. Ví dụ như nếu cho một node đại diện cho device là bộ nhớ flash thì các node con của device sẽ là các phân vùng. 
Ví dụ:
```dts
eeprom: ee24lc512@55 {
    compatible = "microchip,24xx512";
    reg = <0x55>;
    partition1 {
        read-only;
        part-name = "private";
        offset = <0>;
        size = <1024>;
    };
    partition2 {
        part-name = "data";
        offset = <1024>;
        size = <64512>;
    };
};
```

Ta có thể sử dụng hàm ```for_each_child_of_node()``` để duyệt qua các node con của một node cho trước.
Ví dụ:
```c
struct device_node *np = pdev->dev.of_node;
struct device_node *sub_np;
for_each_child_of_node(np, sub_np) {
    /* sub_np will point successively to each sub-node */
    [...]
    int size;
    of_property_read_u32(client->dev.of_node, "size", &size);
    ...
}
```

### Platform driver và Device Tree
Ngày nay thì việc kết hợp sử dụng platform driver với device tree được khuyến cáo là nên dùng, và hạn chế khai báo phần cứng trên các source code c. Ở bài viết về platform driver thì ta đã thảo luận về kiểu match OF (Open Firmware), đây là kiểu match dựa trên cơ chế DT.

#### Kiểu match OF.
Đây là kiểu match đầu tiên được thực hiện bởi platform core để match giữa driver và device. Kiểu match này sử dụng trường ```compatible``` trong ```of_match_table``` của ```platform_driver```. Mỗi device node trên DT cũng có một thuộc tính device tree, có thể là một chuỗi hoặc danh sách các chuỗi. Bất cứ một platform driver nào khai báo trường ```compatible``` của nó giống một thuộc tính ```compatible``` của device trên DT thì sẽ xảy ra match.
```of_match_table``` là một struct kiểu ```of_device_id```:
```c
struct of_device_id {
    [...]
    char compatible[128];
    const void *data;
};
```

trong đó:
- ```char compatible[128]```: là chuỗi dùng để match với thuộc tính ```compatible``` của device node trong DT.
- ```const void *data```: là con trỏ có thể trỏ đến bất cứ struct nào, sử dụng cho việc lưu dữ liệu cho loại device.

Do ```of_match_table``` là một con trỏ nên ta thường sẽ truyền vào một mảng các struct
```of_device_id``` để driver có thể match với nhiều loại device:
```c
static const struct of_device_id imx_uart_dt_ids[] = {
    { .compatible = "fsl,imx6q-uart", },
    { .compatible = "fsl,imx1-uart", },
    { .compatible = "fsl,imx21-uart", },
    { /* sentinel */ }
};
```

Để thông báo cho kernel về danh sách các device mà driver hỗ trợ thì ta dùng macro ```MODULE_DEVICE_TABLE```:
```c
MODULE_DEVICE_TABLE(of, imx_uart_dt_ids);
```

Một khi ta đã khai báo mảng of_device_id thì tiếp theo cần phải truyền con trỏ vào trường ```of_match_table``` của platform driver:
```c
static struct platform_driver serial_imx_driver = {
    [...]
    .driver = {
        .name = "imx-uart",
        .of_match_table = imx_uart_dt_ids,
        [...]
    },
};
```

Như vậy thì driver của ta đã tương thích với device tree:
```c
uart1: serial@02020000 {
    compatible = "fsl,imx6q-uart", "fsl,imx21-uart";
    reg = <0x02020000 0x4000>;
    interrupts = <0 26 IRQ_TYPE_LEVEL_HIGH>;
    [...]
};
```

Ở device tree trên thì có 2 chuỗi ```compatible``` được khai báo. Nếu chuỗi thứ nhất không match thì driver sẽ kiểm tra match với chuỗi thứ 2.

Khi xảy ra match, hàm ```probe``` của driver được gọi cùng với tham số ```struct```
```platform_device```. Tham số này luôn chứa struct device dev, struct này lại chứa ```struct device_node *of_node```. ```of_node``` là node liên kết với device của ta, và nó được dùng để lấy cài đặt của device:
```c
static int serial_imx_probe(struct platform_device *pdev)
{
    [...]
    struct device_node *np;
    np = pdev->dev.of_node;
    if (of_get_property(np, "fsl,dte-mode", NULL))
        sport->dte_mode = 1;
        [...]
}
```

Giả sử platform driver hỗ trợ nhiều loại device, với mỗi loại device có mỗi loại dữ liệu riêng (trường ```const void *data``` trong ```of_device_id```). Để có thể lấy được dữ liệu này cho device vừa match ta sử dụng hàm ```of_match_device```. Ví dụ:
```c
static int my_probe(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;
    const struct of_device_id *match;
    match = of_match_device(imx_uart_dt_ids, &pdev->dev);
    if (match) {
        /* Devicetree, extract the data */
        my_data = match->data
    } else {
        /* Board init file */
        my_data = dev_get_platdata(&pdev->dev);
    }
    [...]
}
```

#### Khi kernel không hỗ trợ DT?
Kernel sẽ hỗ trợ Device Tree khi ta bật option ```CONFIG_OF```. Ta sẽ tránh việc sử dụng các API của device tree khi kernel không hỗ trợ:
```c
#ifdef CONFIG_OF
static const struct of_device_id imx_uart_dt_ids[] = {
    { .compatible = "fsl,imx6q-uart", },
    { .compatible = "fsl,imx1-uart", },
    { .compatible = "fsl,imx21-uart", },
    { /* sentinel */ }
    };
    /* other devicetree dependent code */
    [...]
#endif
```

Ngoài cách trên, ta có thể sử dụng macro ```of_match_ptr```, macro này sẽ trả về NULL khi ta không bật option ```CONFIG_OF```. Macro này được định nghĩa ở ```include/linux/of.h``` :
```c
#define of_match_ptr(_ptr) (_ptr) /* When CONFIG_OF is enabled */
#define of_match_ptr(_ptr) NULL
/* When it is not */
```

Và ta có thể sử dụng như sau:
```c
static int my_probe(struct platform_device *pdev)
{
    const struct of_device_id *match;
    match = of_match_device(of_match_ptr(imx_uart_dt_ids),
    &pdev->dev);
    [...]
}
static struct platform_driver serial_imx_driver = {
[...]
    .driver = {
        .name = "imx-uart",
        .of_match_table = of_match_ptr(imx_uart_dt_ids),
    },
};
```

#### Dữ liệu device với nhiều loại hardware  
Thỉnh thoảng thì driver có thể hỗ trợ nhiều hardware khác nhau, mỗi loại có dữ liệu và thông số cấu hình khác nhau. Những dữ liệu này có thể là bảng các hàm, tập thanh ghi, hoặc một dữ liệu nào đó đặc trưng cho phần cứng. Như ta đã biết thì ```struct of_device_id``` có trường ```const void *data```. Ta có thể truyền bất kì dữ liệu vào vào trường này.
Giả sử ta có 3 device khác nhau. 
```of_device_id.data ```sẽ chứa con trỏ đến các dữ liệu của mỗi device. 
Ví dụ:
Đầu tiên ta khai báo các struct riêng:
```c
/* i.MX21 type uart runs on all i.mx except i.MX1 and i.MX6q */
enum imx_uart_type {
    IMX1_UART,
    IMX21_UART,
    IMX6Q_UART,
};
/* device type dependent stuff */
struct imx_uart_data {
    unsigned uts_reg;
    enum imx_uart_type devtype;
};
```

Tạo một mảng các struct dữ liệu:
```c
static struct imx_uart_data imx_uart_devdata[] = {
    [IMX1_UART] = {
        .uts_reg = IMX1_UTS,
        .devtype = IMX1_UART,
    },
    [IMX21_UART] = {
        .uts_reg = IMX21_UTS,
        .devtype = IMX21_UART,
    },
    [IMX6Q_UART] = {
        .uts_reg = IMX21_UTS,
        .devtype = IMX6Q_UART,
    },
};
```

Ứng với mỗi trường ```compatible``` là một phần tử của mảng dữ liệu:
```c
static const struct of_device_id imx_uart_dt_ids[] = {
    {   .compatible = "fsl,imx6q-uart", 
        .data = &imx_uart_devdata[IMX6Q_UART], 
    },
    {   .compatible = "fsl,imx1-uart", 
        .data =    &imx_uart_devdata[IMX1_UART], 
    },
    {   .compatible = "fsl,imx21-uart", 
        .data = &imx_uart_devdata[IMX21_UART], 
    },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, imx_uart_dt_ids);
static struct platform_driver serial_imx_driver = {
    [...]
    .driver = {
        .name   = "imx-uart",
        .of_match_table = of_match_ptr(imx_uart_dt_ids),
    },
};
```

Ở hàm ```probe``` thì ta sẽ lấy dữ liệu cho device được match:
```c
static int imx_probe_dt(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;
    const struct of_device_id *of_id;
    of_id = of_match_device(of_match_ptr(imx_uart_dt_ids), &pdev->dev);
    if (!of_id)
        /* no device tree device */
        return 1;
    [...]
    sport->devdata = of_id->data; /* Get private data back*/
}
```

### Kiểu match kết hợp
Tuy việc khai báo phần cứng bằng các file code c (board init file) là lỗi thời và khuyến cáo không nên dùng, nhưng có nhiều trường hợp, device vẫn được khai báo theo cách trên, và đôi khi, được khai báo vừa kết hợp board init file và device tree. 
Ví dụ ta viết struct các device được hỗ trợ theo kiểu match ID:
```c
static const struct platform_device_id sdma_devtypes[] = {
    {
        .name = "imx51-sdma",
        .driver_data = (unsigned long)&sdma_imx51,
    }, 
    {
        .name = "imx53-sdma",
        .driver_data = (unsigned long)&sdma_imx53,
    }, 
    {
        .name = "imx6q-sdma",
        .driver_data = (unsigned long)&sdma_imx6q,
    }, 
    {
        .name = "imx7d-sdma",
        .driver_data = (unsigned long)&sdma_imx7d,
    }, 
    {
        sentinel */
    }
};
MODULE_DEVICE_TABLE(platform, sdma_devtypes);
```

Và các device giống hệt được hỗ trợ theo kiểu match OF:
```c
static const struct of_device_id sdma_dt_ids[] = {
    { .compatible = "fsl,imx6q-sdma", .data = &sdma_imx6q, },
    { .compatible = "fsl,imx53-sdma", .data = &sdma_imx53, },
    { .compatible = "fsl,imx51-sdma", .data = &sdma_imx51, },
    { .compatible = "fsl,imx7d-sdma", .data = &sdma_imx7d, },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, sdma_dt_ids);
```

Lúc đó, hàm ```probe``` của ta sẽ phải xử lý như sau:
```c
static int sdma_probe(struct platform_device *pdev)
{
    const struct of_device_id *of_id = of_match_device(of_match_ptr(sdma_dt_ids), &pdev->dev);
    struct device_node *np = pdev->dev.of_node;

    /* If device tree, */
    if (of_id)
        drvdata = of_id->data;
    /* else, hard-coded */
    else if (pdev->id_entry)
        drvdata = (void *)pdev->id_entry->driver_data;
    if (!drvdata) {
        dev_err(&pdev->dev, "unable to find driver data\n");
        return -EINVAL;
    }
    [...]
}
```

Và ta khai báo platform driver:
```c
static struct platform_driver sdma_driver = {
    .driver = {
        .name = "imx-sdma",
        .of_match_table = of_match_ptr(sdma_dt_ids),
    },
    .id_table = sdma_devtypes,
    .remove = sdma_remove,
    .probe = sdma_probe,
};
module_platform_driver(sdma_driver);
```

_Tham khảo từ Linux Device Drivers Development_
