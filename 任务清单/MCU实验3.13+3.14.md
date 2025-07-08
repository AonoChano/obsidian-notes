### **实验一：3.13 ADC0809 模数转换实验分析**

这个实验的目的是将电位器产生的模拟电压信号，通过 ADC0809 芯片转换为数字信号，并在数码管上显示出电压值。

#### **程序流程图分析 (第一张图)**

这张流程图分为两个并行的部分：主函数 (`主函数`) 和中断服务程序 (`定时器0中断处理函数`)。

**1. 主函数 (Main Function) - 负责数据采集和处理**
*   **初始化部分 (执行一次):**
    1.  `设定定时器工作方式` & `设定定时器初值`: 初始化定时器0，用于定时中断。这个中断的主要作用是动态扫描刷新数码管，以及为ADC0809提供时钟信号。
    2.  `开中断` & `允许中断`: 开启总中断(EA)和定时器0中断(ET0)。
    3.  `0809片选置0`: 这步的描述可能不准确。根据硬件连接表，ADC0809的通道选择引脚A, B, C接地，这表示硬件上已经固定选择了通道0 (IN0)。此步骤在软件上可能是指初始化与ADC相关的IO口。

*   **主循环 (while(1)) - 持续执行:**
    1.  `0809启动信号`: 向ADC0809的 `START` 引脚发送一个启动脉冲（先拉高再拉低），启动一次A/D转换。
    2.  `转换是否完成`: 循环检测ADC0809的 `EOC` (End of Conversion) 引脚。当转换正在进行时，EOC为低电平；转换完成后，EOC变为高电平。程序在此等待，直到EOC变高。这是一种**查询方式**。
    3.  `允许0809输出` & `读出转换结果`: 向ADC0809的 `OE` (Output Enable) 引脚发送一个高电平，使其输出转换结果。然后从P0口读取这个8位的数字量（范围0-255）。
    4.  `转换计算`: 将读取到的数字量（0-255）转换为实际的电压值。例如，如果参考电压是5V，则 `电压 = (读取值 / 255) * 5.0`。
    5.  `分离小数点后数字、个、十位`: 将计算出的电压值（如3.45V）分解为单个数字（如'3', '4', '5'），以便在数码管上显示。
    6.  `数码管显示转换后数值`: 这一步通常是将分解出的数字存入一个全局数组（显示缓冲区）。实际的显示动作由定时器中断完成。

**2. 定时器0中断处理函数 (Timer 0 ISR) - 负责提供时钟和显示刷新**
*   `0809时钟信号取反`: ADC0809需要一个时钟信号（CLK）来工作。这个流程图表明，程序利用定时器中断，通过在一个IO口上反复取反电平，来软件模拟一个稳定的时钟信号提供给ADC0809的CLK引脚。
*   **(隐含的功能)**: 除了提供时钟，这个中断服务程序最重要的任务是**动态扫描数码管**。它会轮流选中每一位数码管（通过P2.0~P2.3），并将显示缓冲区中对应的数字段码送到P1口，利用人眼的视觉暂留效应，实现多位数码管的稳定显示。

#### **硬件连接分析**

*   **MCU 与 ADC0809**:
    *   `P0口` 作为数据总线，用于读取转换后的8位数字量。
    *   `P3.0 (EOC)`, `P3.6 (WR/START)`, `P3.7 (RD/OE)` 作为控制线。
    *   `ALE` 连接到 `CLK`，这是利用51单片机ALE引脚自身会输出固定频率脉冲的特性，为ADC提供时钟，这与流程图中用定时器产生时钟是两种不同的实现方法，但目的相同。
    *   `A, B, C` 接地，硬连线选择了 `IN0` 通道。
*   **MCU 与 数码管**:
    *   `P1口` 连接数码管的段选线 (a-h)，控制显示哪个数字。
    *   `P2.0~P2.3` 连接数码管的位选线，控制哪一位数码管被点亮。
#### 实验13**参考代码 (适用于AT89S52)**

```c
#include <reg52.h>

#define BYTE unsigned char
#define WORD unsigned int

sbit adc_clk   = P3^1; // ADC 时钟输出
sbit adc_eoc   = P3^0; // ADC 转换完成输入
sbit adc_cs    = P2^7; // ADC 片选 (低电平有效)
sbit adc_start = P3^6; // ADC 启动转换
sbit adc_oe    = P3^7; // ADC 数据输出使能 (低电平有效)


// volatile 关键字确保主循环和中断能正确访问这些共享变量
volatile WORD g_adc_raw_value = 0; // 存储从ADC P0口直接读出的原始8位数据
volatile BYTE g_display_digits[4]; // 存储待显示的4个数字，[0]为个位，[1-3]为小数位

// 数码管共阳极字形码表，存放在ROM中以节省RAM
BYTE code led_glyph_table[] = {
    0xC0, /* 0 */ 0xF9, /* 1 */ 0xA4, /* 2 */ 0xB0, /* 3 */
    0x99, /* 4 */ 0x92, /* 5 */ 0x82, /* 6 */ 0xF8, /* 7 */
    0x80, /* 8 */ 0x90  /* 9 */
};


void initialize_system(void) {
    // 初始化ADC控制引脚
    adc_cs    = 0; // 始终选中ADC芯片
    adc_oe    = 1; // 平时禁止ADC数据总线输出
    adc_start = 1; // 转换启动信号默认为高电平

    // 配置并启动定时器0，设置为1毫秒中断一次
    TMOD = 0x01;  // 定时器0工作在16位定时模式
    TH0  = 0xFC;  // 装载1ms定时初值 (11.0592MHz晶振)
    TL0  = 0x66;
    ET0  = 1;     // 允许定时器0中断
    TR0  = 1;     // 启动定时器0
    EA   = 1;     // 开放总中断
}

// 定时器0中断服务程序，负责所有高频的实时任务
void timer0_isr_routine(void) interrupt 1 {
    static BYTE scan_cursor = 0; // 静态变量，用于记录当前扫描到哪一位数码管

    // --- 任务1: ADC时钟生成与数据采集 ---
    adc_clk = ~adc_clk; // 每次中断都翻转时钟线，为ADC提供工作时钟

    // 检查转换是否结束 (EOC=1)
    if (adc_eoc == 1) {
        adc_oe = 0;             // 使能ADC数据输出
        g_adc_raw_value = P0;   // 从P0口读取结果
        adc_oe = 1;             // 立刻禁止输出，释放P0口

        // 产生一个下降沿脉冲来启动下一次转换
        adc_start = 0;
        adc_start = 1;
    }

    // --- 任务2: 4位数码管动态扫描与刷新 ---
    
    // a. 消隐：先关闭所有段选和位选，防止切换时产生暗淡的重影
    P1 = 0xFF; // 关闭所有段
    P2 |= 0x0F; // 关闭所有位选（通过将P2低4位置1）

    // b. 位选：根据扫描光标选择要点亮的数码管
    //    使用位操作，只修改P2的低4位，不影响P2.7上的adc_cs
    P2 = (P2 & 0xF0) | (~(1 << scan_cursor) & 0x0F);

    // c. 段选：从显示缓冲区取出对应数字，并查表获得段码送到P1口
    P1 = led_glyph_table[g_display_digits[scan_cursor]];

    // d. 小数点处理：当扫描到第一位(X.XXX)时，点亮其小数点
    if (scan_cursor == 0) {
        P1 &= 0x7F; // 将段码的最高位(DP)清零来点亮小数点
    }

    // e. 移动光标，准备下一次中断刷新下一位
    scan_cursor++;
    if (scan_cursor == 4) {
        scan_cursor = 0; // 循环扫描
    }

    // --- 任务3: 重装定时器初值 ---
    TH0 = 0xFC;
    TL0 = 0x66;
}



void main(void) {
    // 使用32位长整型进行中间计算，防止乘法溢出
    unsigned long temp_calc_value; 
    WORD voltage_mv; // 存储以毫伏为单位的电压值, e.g., 4567 代表 4.567V

    initialize_system();

    while (1) {
        temp_calc_value = g_adc_raw_value; // 从volatile变量读取一次，避免多次读取
        temp_calc_value = (temp_calc_value * 5000) / 255;
        voltage_mv = (WORD)temp_calc_value;

        ET0 = 0; 
        
        g_display_digits[0] = voltage_mv / 1000;       // 提取千位 -> 个位 X.
        g_display_digits[1] = (voltage_mv / 100) % 10; // 提取百位 -> 小数第一位 .X
        g_display_digits[2] = (voltage_mv / 10) % 10;  // 提取十位 -> 小数第二位 .XX
        g_display_digits[3] = voltage_mv % 10;         // 提取个位 -> 小数第三位 .XXX

        ET0 = 1; 
    }
}
```


#### **实验思考题解答**

**1. 思考如果要采集多处的电压，应该如何设计。**
答：要采集多路电压，需要利用ADC0809的8个模拟输入通道(IN0-IN7)。设计修改如下：
*   **硬件修改**: 将ADC0809的地址选择引脚 `ADDA`, `ADDB`, `ADDC` 连接到单片机的三个I/O口（例如P2.4, P2.5, P2.6），而不是直接接地。
*   **软件修改**: 在启动A/D转换之前，先通过控制这三个I/O口输出不同的电平组合，来选择要转换的通道。例如：
    *   要选择通道3 (IN3)，则设置 (ADDC, ADDB, ADDA) = (0, 1, 1)。
    *   要选择通道5 (IN5)，则设置 (ADDC, ADDB, ADDA) = (1, 0, 1)。
    *   设置好通道地址后，再向 `START` 引脚发送启动信号。

**2. 编程控制八通道轮流采样，经过转换后显示。**
答：可以设计一个循环，依次对8个通道进行采样和显示。
*   **程序逻辑**:
    1.  在 `while(1)` 循环中，增加一个 `for` 循环，变量 `channel` 从0到7。
    2.  在 `for` 循环内部：
        a.  **选择通道**: 将 `channel` 的值写入连接 `ADDA, ADDB, ADDC` 的I/O口。
        b.  **启动转换**: 发送 `START` 脉冲。
        c.  **等待转换完成**: 查询 `EOC` 引脚。
        d.  **读取结果**: 使能 `OE` 并从P0口读取数据。
        e.  **数据处理与显示**: 将读取的数据进行计算，并更新数码管的显示缓冲区。可以在数码管上同时显示通道号和电压值（例如，第一位显示通道号，后三位显示电压）。
        f.  **延时**: 加入一个短暂的延时，以便人眼能看清每个通道的读数，然后再切换到下一个通道。

---

### **实验二：3.14 DAC0832 数模转换实验分析**

这个实验的目的是让单片机输出一个变化的数字量给 DAC0832 芯片，由DAC芯片将其转换为模拟电压信号，从而产生一个锯齿波。
3.14原理图设计
1.在Proteus 中绘制单片机最小系统，包括主控芯片、晶振电路和复位电路。
2.添加DAC0832 芯片，片选CS 端口接电阻上拉，标号连接到单片机P2.7 口，WR 端口标号连接到
P3.6 口，两个GND 接地，VREF 接2.5V 高电平，VCC 和ILE 接高电平，WR2 和XFER、OUT2 同
时接地，输入端DI0-DI7 连接到P0 口，P0 口需要排阻上拉。
3.添加LM358 芯片，-端口连接到0832 的OUT1 端，+端口接地，1 端口作为反馈信号接回0832 的
RFB 端。8、4 引脚分别接+12 和-12 电源。
微控制器仿真实验实训平台(FB-EDU-MCU-F)
108
4.再添加一个LM358，分别按下图电路连接，构成3 个不同的放大电路，接入示波器的A、B、C 端
口，观察3 个数模转换后不同的波形。

| | 硬件连接表 | | | ---- | ---- | ---- | | | MCU - AT89S52 | 数模转换 | 8 位 LED/示波器 | | | P27 | CS | | | | P36 | WR | | | | P00~P07 | DB0~DB7 | | | | | DA 输出 | D1 |

#### **程序流程图**

这张图也分为主函数和另一个功能函数。

**1. 主函数 (Main Function) - 负责产生锯齿波**
*   **初始化部分 (执行一次):**
    1.  `0832片选CS置0`: 将 `/CS` 引脚置为低电平，选中DAC0832芯片。
    2.  `0832写信号, WR置0`: 将 `/WR` (即/WR1) 引脚也置为低电平。当 `/CS` 和 `/WR1` 都为低电平时，DAC0832处于“直通模式”，P0口的数据会直接送入DAC寄存器并开始转换，这大大简化了控制时序。

*   **主循环 (while(1)) - 持续执行:**
    1.  `for循环255次, 计数增加255次后计数归0`: 这实际上是一个 `for (i = 0; i <= 255; i++)` 的循环。
    2.  **(隐含的动作)**: 在 `for` 循环的每一次迭代中，程序将当前的计数值 `i` 输出到 `P0` 口。
    3.  `产生锯齿波`: 由于 `i` 从0线性增加到255，然后突变回0，所以P0口输出的数字量也是一个数字锯齿波。DAC0832将这个数字量实时转换为模拟电压，因此其输出端 `Vout` 就会产生一个平滑上升、然后瞬间跌落的模拟锯齿波电压。

**2. 方波发生函数 (Square Wave Function) - 另一个波形示例**
这是一个独立的函数，演示如何产生方波。
1.  `输出低电平`: 向P0口写入 `0x00`，DAC输出最低电压。
2.  `短延时`: 保持低电平一段时间。
3.  `输出高电平`: 向P0口写入 `0xFF` (255)，DAC输出最高电压。
4.  **(隐含的动作)**: 之后应该再有一个延时，然后循环整个过程，才能形成连续的方波。

#### **硬件连接分析**

*   **MCU 与 DAC0832**:
    *   `P0口` 作为数据总线，向DAC提供8位数字量。
    *   `P2.7 (/CS)` 和 `P3.6 (/WR)` 作为控制线。如前所述，将它们都置低可以简化操作。
    *   `DA输出` 连接到示波器或电压表，用于观察生成的模拟波形。

#### **参考代码 (适用于AT89S52)**

```c
#include <reg52.h>

// 类型定义
typedef unsigned char uchar;

// DAC0832 控制引脚定义
sbit DAC_CS = P2^7;
sbit DAC_WR = P3^6;

// 简单的延时函数，用于控制锯齿波的频率
void delay_us(unsigned int us) {
    while(us--);
}

void main() {
    uchar i;

    // 初始化DAC0832为直通模式
    DAC_CS = 0; // 片选，一直有效
    DAC_WR = 0; // 写使能，一直有效，P0数据直接进入DAC寄存器

    while(1) {
        // 循环输出0到255，产生锯齿波
        for (i = 0; i < 255; i++) {
            P0 = i; // 将计数值送到P0口
            // delay_us(10); // 可选：加入微小延时来降低波形频率
        }
        // 当i=255时，for循环结束，下一次循环i从0开始，
        // 从而实现从最大值到0的跳变。
        // 为确保达到最高点，可以加一句
        P0 = 255;
        // delay_us(10);
    }
}
```

**此部分需要作答，和上面的函数是一个完整的函数：如果要使用DAC0832编写程序产生一个锯齿波、三角波、方波，正弦波四种波形显示，该如何修改程序。**
答：可以编写三个独立的函数，分别用于产生三种波形，然后在主循环中合理使用用它们。

*   **参考程序结构**:
    ```c
    void generate_sawtooth() {
        for (int i = 0; i <= 255; i++) {
            P0 = i; // 输出到DAC
            delay_us(50); // 微秒级延时控制频率
        }
    }

    void generate_triangle() {
        // 上升沿
        for (int i = 0; i <= 255; i++) {
            P0 = i;
            delay_us(50);
        }
        // 下降沿
        for (int i = 255; i >= 0; i--) {
            P0 = i;
            delay_us(50);
        }
    }

    void generate_square() {
        P0 = 0x00; // 输出低电平
        delay_ms(10); // 毫秒级延时控制频率
        P0 = 0xFF; // 输出高电平
        delay_ms(10);
    }

    void main() {
        // 初始化DAC (CS=0, WR=0)
        ...
        while(1) {
            // 产生一段锯齿波 (例如，持续1秒)
            for(int j=0; j<50; j++) generate_sawtooth();

            // 产生一段三角波 (例如，持续1秒)
            for(int j=0; j<25; j++) generate_triangle();

            // 产生一段方波 (例如，持续1秒)
            for(int j=0; j<50; j++) generate_square();
        }
    }
    ```
    这个波形是轮流的，请尝试书写完整的一整个实验的代码



```C
#include <reg52.h>

#define BYTE unsigned char
#define WORD unsigned int

sbit adc_clk   = P3^1;
sbit adc_eoc   = P3^0;
sbit adc_cs    = P2^7;
sbit adc_start = P3^6;
sbit adc_oe    = P3^7;

volatile WORD g_adc_raw_value = 0;
volatile BYTE g_display_digits[4];

BYTE code led_glyph_table[] = {
    0xC0, 0xF9, 0xA4, 0xB0,
    0x99, 0x92, 0x82, 0xF8,
    0x80, 0x90
};

void initialize_system(void) {
    adc_cs    = 0;
    adc_oe    = 1;
    adc_start = 1;

    TMOD = 0x01;
    TH0  = 0xFC;
    TL0  = 0x66;
    ET0  = 1;
    TR0  = 1;
    EA   = 1;
}

void timer0_isr_routine(void) interrupt 1 {
    static BYTE scan_cursor = 0;

    adc_clk = ~adc_clk;

    if (adc_eoc == 1) {
        adc_oe = 0;
        g_adc_raw_value = P0;
        adc_oe = 1;

        adc_start = 0;
        adc_start = 1;
    }

    P1 = 0xFF;
    P2 |= 0x0F;

    P2 = (P2 & 0xF0) | (~(1 << scan_cursor) & 0x0F);

    P1 = led_glyph_table[g_display_digits[scan_cursor]];

    if (scan_cursor == 0) {
        P1 &= 0x7F;
    }

    scan_cursor++;
    if (scan_cursor == 4) {
        scan_cursor = 0;
    }

    TH0 = 0xFC;
    TL0 = 0x66;
}

void main(void) {
    unsigned long temp_calc_value;
    WORD voltage_mv;

    initialize_system();

    while (1) {
        temp_calc_value = g_adc_raw_value;
        temp_calc_value = (temp_calc_value * 5000) / 255;
        voltage_mv = (WORD)temp_calc_value;

        ET0 = 0;
        
        g_display_digits[0] = voltage_mv / 1000;
        g_display_digits[1] = (voltage_mv / 100) % 10;
        g_display_digits[2] = (voltage_mv / 10) % 10;
        g_display_digits[3] = voltage_mv % 10;

        ET0 = 1;
    }
}
```
