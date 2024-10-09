Original 杨文波 CPP与系统软件联盟
_2022年04月15日 20:00_
本文摘录自资深软件架构师杨文波老师「与大师为伍：走进Scott Meyers的C++世界」直播实录。

**01** **前言**
在纯软件的编程实践中，学习许多语言的第一个例子是 Hello World 打印程序。嵌入开发中的 Hello World 是 blinky，点亮 LED 灯珠的程序。这个程序虽小，细想却也并不简单。MCU 身处于半导体的世界中，由数以十万计的与非门等组成，而与非门是由 PN 结组成，blinky 程序要让这样的一套集成电路通过 GPIO 反过来驱动外围物理世界的一个 PN 结，使它发光。blinky 这个例子也引出了嵌入式场景中的一类硬件——MMIO（内存映射的输入输出设备），具体来说是 GPIO，理解了 GPIO 的建模，可以推广到许多其他的外围设备。Scott Meyers 的嵌入式 C++ 最佳实践课程中总结了 C++ 建模 MMIO 的一套习语。下面我将分享，如何从 blinky 这个例子入手来学习它。

**02** **C建模 MMIO 的常见做法**
首先回顾在嵌入式 C 编程中 MMIO 的常见实现，在此我们选取了较有代表性的两家嵌入式 MCU 方案，乐鑫 ESP32 和 NXP LPC55。
乐鑫 ESP 系列，是较为流行的基于蓝牙 WiFi 一体模组的 IoT 方案。它的官方开发框架 Espressif IoT Development Framework 在 github 上的星数颇高，有着活跃的社区：
esp-idf (_https://github.com/espressif/esp-idf)_
它的 blinky 是这样实现的：

```cpp
#elif CONFIG_BLINK_LED_GPIOstatic void blink_led(void){    /* Set the GPIO level according to the state (LOW or HIGH)*/    gpio_set_level(BLINK_GPIO, s_led_state);}static void configure_led(void){    ESP_LOGI(TAG, "Example configured to blink GPIO LED!");    gpio_reset_pin(BLINK_GPIO);    /* Set the GPIO as a push/pull output */    gpio_set_direction(BLINK_GPIO, GPIO_MODE_OUTPUT);}#endifvoid app_main(void){    /* Configure the peripheral according to the LED type */    configure_led();    while (1) {        ESP_LOGI(TAG, "Turning the LED %s!", s_led_state == true ? "ON" : "OFF");        blink_led();        /* Toggle the LED state */        s_led_state = !s_led_state;        vTaskDelay(CONFIG_BLINK_PERIOD / portTICK_PERIOD_MS);    }}
```

程序先是配置了一个 GPIO，之后在 while 循环中让 GPIO 拉高拉低，“干活” 的函数是 `gpio_set_level()`。观察它的实现：

```cpp
static esp_err_t gpio_output_enable(gpio_num_t gpio_num){    GPIO_CHECK(GPIO_IS_VALID_OUTPUT_GPIO(gpio_num), "GPIO output gpio_num error", ESP_ERR_INVALID_ARG);    gpio_hal_output_enable(gpio_context.gpio_hal, gpio_num);    esp_rom_gpio_connect_out_signal(gpio_num, SIG_GPIO_OUT_IDX, false, false);    return ESP_OK;}// ... ...esp_err_t gpio_set_level(gpio_num_t gpio_num, uint32_t level){    GPIO_CHECK(GPIO_IS_VALID_OUTPUT_GPIO(gpio_num), "GPIO output gpio_num error", ESP_ERR_INVALID_ARG);    gpio_hal_set_level(gpio_context.gpio_hal, gpio_num, level);    return ESP_OK;}int gpio_get_level(gpio_num_t gpio_num){    return gpio_hal_get_level(gpio_context.gpio_hal, gpio_num);
```

注意到宏 `GPIO_CHECK` 在许多函数中都反复出现。

```
#define GPIO_CHECK(a, str, ret_val) ESP_RETURN_ON_FALSE(a, ret_val, GPIO_TAG, "%s", str)
```

```
#define GPIO_IS_VALID_OUTPUT_GPIO(gpio_num) (((1ULL << (gpio_num)) & SOC_GPIO_VALID_OUTPUT_GPIO_MASK) != 0)
```

```
/** * Macro which can be used to check the condition. If the condition is not 'true', it prints the message * and returns with the supplied 'err_code'. */#define ESP_RETURN_ON_FALSE(a, err_code, log_tag, format, ...) do {                             \        if (unlikely(!(a))) {                                                                   \            ESP_LOGE(log_tag, "%s(%d): " format, __FUNCTION__, __LINE__, ##__VA_ARGS__);        \            return err_code;                                                                    \        }                                                                                       \    } while(0)
```

重复的调用违反了 DRY（Don't Repeat Yourself）的原则：`GPIO_CHECK` 宏包装的代码反复检查 GPIO 的 `port` 号是不是在合理的范围内，如果不是，会在运行期打印错误信息并返回一个错误。然而在运行期，每个配置 GPIO 的函数都会调用它，而每次调用操作 GPIO 的函数如 `gpio_set_level` 也都会调用它。

进入到 `gpio_ll_set_level()` 中，发现 GPIO 的 `port` 号又被检查了一次:

```
/** * @brief  GPIO set output level * * @param  hal Context of the HAL layer * @param  gpio_num GPIO number. If you want to set the output level of e.g. GPIO16, gpio_num should be GPIO_NUM_16 (16); * @param  level Output level. 0: low ; 1: high */#define gpio_hal_set_level(hal, gpio_num, level) gpio_ll_set_level((hal)->dev, gpio_num, level)
```

```
/** * @brief  GPIO set output level * * @param  hw Peripheral GPIO hardware instance address. * @param  gpio_num GPIO number. If you want to set the output level of e.g. GPIO16, gpio_num should be GPIO_NUM_16 (16); * @param  level Output level. 0: low ; 1: high */static inline void gpio_ll_set_level(gpio_dev_t *hw, gpio_num_t gpio_num, uint32_t level){    if (level) {        if (gpio_num < 32) {            hw->out_w1ts = (1 << gpio_num);        } else {            HAL_FORCE_MODIFY_U32_REG_FIELD(hw->out1_w1ts, data, (1 << (gpio_num - 32)));        }    } else {        if (gpio_num < 32) {            hw->out_w1tc = (1 << gpio_num);        } else {            HAL_FORCE_MODIFY_U32_REG_FIELD(hw->out1_w1tc, data, (1 << (gpio_num - 32)));        }    }}
```

在 LED 一亮一暗的过程中，gpio_num 到底会被检查多少次呢？而 32 又是什么意思？这种检查显然会带来运行期开销，而设计意图在对同一参数的反复而略有不同的检查中也显得有些模糊。

再看 NXP LPC55S6x 中的实现。NXP LPC55S6x 主要目标应用是工业 IoT 及楼宇自动化等，它基于 ARM Cortex M33 并在芯片内集成了 RAM 和 ROM 和常见的外设。我手中刚好有一块第三方 LPC55S69 开发板：OKdo-E1，从链接\_(https://www.okdo.com/getting-started/get-started-with-okdo-e1-board/)\_ 中可以下载到相关的资料，包括 NXP 官方的 IDE。稍后我也会用它演示用 C++ 为 MMIO 建模。

`main.c` 的实现方式是类似的:

```
/*! * @brief Main function */int main(void){   //... ...;    while (1)    {        /* Delay 1000 ms */        SysTick_DelayTicks(500U);        GPIO_PortToggle(GPIO, BOARD_LED_PORT, 1u << BOARD_LED_PIN);    }}
```

```
/*! * @brief Reverses current output logic of the multiple GPIO pins. * * @param base GPIO peripheral base pointer(Typically GPIO) * @param port GPIO port number * @param mask GPIO pin number macro */static inline void GPIO_PortToggle(GPIO_Type *base, uint32_t port, uint32_t mask){    base->NOT[port] = mask;}
```

这里 `GPIO_PortToggle` 的实现挺直接，在 `GPIO_Type` 类型的指针指向的 `NOT` 这个位置赋个值，就完成了，没有做任何检查。

结合 `GPIO_Type` 结构体的定义：

```
/** GPIO - Register Layout Typedef */typedef struct {  __IO uint8_t B[2][32];                           /**< Byte pin registers for all port GPIO pins, array offset: 0x0, array step: index*0x20, index2*0x1 */       uint8_t RESERVED_0[4032];  __IO uint32_t W[2][32];                          /**< Word pin registers for all port GPIO pins, array offset: 0x1000, array step: index*0x80, index2*0x4 */       uint8_t RESERVED_1[3840];  __IO uint32_t DIR[2];                            /**< Direction registers for all port GPIO pins, array offset: 0x2000, array step: 0x4 */       uint8_t RESERVED_2[120];  __IO uint32_t MASK[2];                           /**< Mask register for all port GPIO pins, array offset: 0x2080, array step: 0x4 */       uint8_t RESERVED_3[120];  __IO uint32_t PIN[2];                            /**< Port pin register for all port GPIO pins, array offset: 0x2100, array step: 0x4 */       uint8_t RESERVED_4[120];  __IO uint32_t MPIN[2];                           /**< Masked port register for all port GPIO pins, array offset: 0x2180, array step: 0x4 */       uint8_t RESERVED_5[120];  __IO uint32_t SET[2];                            /**< Write: Set register for port. Read: output bits for port, array offset: 0x2200, array step: 0x4 */       uint8_t RESERVED_6[120];  __O  uint32_t CLR[2];                            /**< Clear port for all port GPIO pins, array offset: 0x2280, array step: 0x4 */       uint8_t RESERVED_7[120];  __O  uint32_t NOT[2];                            /**< Toggle port for all port GPIO pins, array offset: 0x2300, array step: 0x4 */       uint8_t RESERVED_8[120];  __O  uint32_t DIRSET[2];                         /**< Set pin direction bits for port, array offset: 0x2380, array step: 0x4 */       uint8_t RESERVED_9[120];  __O  uint32_t DIRCLR[2];                         /**< Clear pin direction bits for port, array offset: 0x2400, array step: 0x4 */       uint8_t RESERVED_10[120];  __O  uint32_t DIRNOT[2];                         /**< Toggle pin direction bits for port, array offset: 0x2480, array step: 0x4 */} GPIO_Type;
```

`port` 的类型是 `uint32_t`，而硬件显然支持不了 4,294,967,296 个端口，实际只能支持到 2。注释中说，`NOT[2]` 对应的硬件的 mask 可以同时支持多个管脚，但对于点灯的应用来说，我们一次只希望反转一个管脚。库不做任何检查，参数相关的行为正确性就只能靠调用方自己来保证。如果调用方“不小心”调用 `GPIO_PortToggle(GPIO, 3, 1u << BOARD_LED_PIN)`，编译也会正常通过，但运行时也许什么都不会发生，也许会发生类似“打开电灯开关，空调会一起开了”这样的事。

以上的两段代码，是嵌入式共享库和半导体 SDK 中的常见实现方式，它们作为半导体的库而言是合格的，能完成基本功能，但用于工业级或更高要求的应用，还不够理想。前一种以运行期反复的检查来确保用户代码以各种方式使用库的接口时，不会出现严重的问题。重复的代码造成了运行时间和二进制尺寸的浪费。后一种库不做什么检查，一根指针传下去直接操作寄存器，由调用方来保证正确性。总结一下 C 语言抽象硬件容易出现的问题：

- 一次变化带来多处改动，违背 DRY 原则；
- 为了保证安全，增加许多重复的检查，牺牲运行速度，增加代码尺寸；
- 如果减少检查，则将安全风险和责任都交给了调用方，而调用方往往并不具备对硬件的深入细致的认识，难以做好；

在编程实践中，为了弥补语言层面类型系统的不足，我们不得不借助于语言外的工具、框架以及工程师的“细心”，而这些语言外的机制都增加了工程复杂度和维护难度。C 语言在面向对象机制上的欠缺也让我们难以轻松表达一些硬件特性，比如：只写寄存器，硬件是否支持动态配置，模块的供电和时钟等。Linux 内核为了抽象 GPIO，设计了由众多指针和函数指针搭建的结构体作为抽象框架，同时维护了超过 150 种 GPIO，但是在小型嵌入式框架和工程中往往未必负担得起这种大而全的方式。

**03** **C++ 建模 MMIO**

下面结合 LPC55 的实际例子，探讨 Scott Meyers 总结的 C++ MMIO 设计习语。习语，大约等同于框架而非规定，它告诉我们将建模的硬件设备所需的特殊处理放在哪里。

建模之前，我们总结一下，什么是 MMIO。MMIO 出现在许多嵌入式 SoC 或 MCU 中，它将 IO 设备映射到程序地址空间的固定地方，通常：

- 输入寄存器和输出寄存器分开。
- 控制/状态寄存器和数据寄存器分开。
- 不同的状态寄存器比特表达各种信息，如准备就绪情况或设备终端是否使能。

而它所映射的内存跟一般的内存相比，常常会有这样的特点：

- 原子的读/写可能需要显式同步。
- 单个比特有时会是只读的，而有时会是只写。
- 清零某个比特可能需要给它赋 1 。
- 一个状态寄存器可能控制或者对应超过一个数据寄存器。比如, 比特 0-3 控制一个数据寄存器，比特 4-7 控制另一个。

使用 C++，能让 MMIO 设备看起来像是有着天然接口的对象。

首先，把控制寄存器写成 `private` 数据成员，这意味着其中的细节，尤其是上文中总结的这段特殊内存的一些特点和细节调用方无需关注。而调用方需要使用的操作抽象为 `public` 成员函数。在点灯的例子中，我们只用到了翻转和读端口的操作，所以暂且只写两个接口。

还有两处细节：特殊内存前面加上了 `volatile` 的限定，正如CP.200所指出的\_(https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#cp200-use-volatile-only-to-talk-to-non-c-memory)\_ ，表明这里将会访问不遵循 C++ 内存模型的硬件；另外，所有的函数都内联，所以在速度上会等同于前文 NXP 风格的驱动。

代码看起来是这样：

```cpp
class  GpioControlReg {   public:      GpioControlReg()      {         printf("do something");      }      enum class Port_t : uint8_t      {         PORT0,         PORT1      };      enum class Level_t : uint8_t      {         LOW,         HIGH      };      enum class Pin_t : uint8_t      {         PIN6 = 6,         PIN9 = 9      };      inline void PortToggle(Port_t port, Pin_t pin)      {         base.NOT[(uint32_t)port] = uint32_t(1u << (uint32_t)pin);      }      inline Level_t GPIO_PinRead(Port_t port, Pin_t pin)      {         return (Level_t)base.B[(uint8_t)port][(uint8_t)pin];      }   private:      volatile GPIO_Type base;};
```

进入了 C++ 的世界，我们马上就可以把 `uint32_t` 换成强类型枚举，这样 `port`、`pin` 和 读出的电平都有了类型， 可以让编译器帮我们来检查。只要调用方尊重 C++ 的类型系统，就能避免访问越界或是进行没意义的访问。接着我们看看调用方怎么调用这个类。观察 `GPIO_Type` 的布局，发现其中包含了若干个 RESERVED_X 这样的空当，加起来有 10 几 KB，这样大段的内存不适合放在一般的 RAM 中，因为嵌入式系统中总的内存一般也只有几十 KB。这里就要用到一个 C++ 的语言特性：布置 `new`（placement `new`）。

C++ 中，`new` 表达式 `T *p = new T` 做两件事情：

- 调用某个 `operator new` 函数，以确定将 T 对象放在哪里。
- 调用合适的 T 构造函数。

请注意，`operator new` 的工作从根本上来说不是分配内存，而是要确定**某个对象应该去哪儿**。通常，这会造成动态的内存分配，比如调用 `malloc`。但有的时候我们知道对象该放在哪里，比如我们想要把对象放在某个 MMIO 地址，或者是把对象构造到某段内存缓冲区中去。这样的 `operator new` 实现大致是这样：

```cpp
void* operator new(std::size_t, void *ptrToMemory){ return ptrToMemory; }
```

这就是布置 `new`（placement `new`），是得到广泛支持的一种标准形式。一个像 `T *p = new T` 这样的表达式会调用两个函数，`operator new` 和构造函数。前者要这样传参：

```cpp
T *p = new(op new args) T;
```

后者要这样传参：

```
T *p = new T(ctor args);
```

如果两者都做自然是：

```
T *p = new (op new args) T(ctor args);
```

我们就能以这个形式在布置 `new` 创建的对象上使用任何构造函数。在 LPC55 blinky 的例子中，这样用它：

```
//...#include <new>//...GpioControlReg* const pcr = new(reinterpret_cast<void*>(GPIO_BASE))GpioControlReg{};while (1){   using Port = GpioControlReg::Port_t;   using Pin = GpioControlReg::Pin_t;   using Level = GpioControlReg::Level_t;   /* Delay 1000 ms */   SysTick_DelayTicks(200U);   if (pcr->GPIO_PinRead(Port::PORT1, Pin::PIN9) == Level::HIGH)   {      putchar('.');      //GPIO_PortToggle(GPIO, BOARD_LED_PORT, 1u << BOARD_LED_PIN);      pcr->PortToggle(Port::PORT1, Pin::PIN6);   }}// ...
```

LPC55 官方 SDK（GNU libstdc++） 的 `new` 实现是这样：

```
_GLIBCXX_NODISCARD inline void* operator new(std::size_t, void* __p) _GLIBCXX_USE_NOEXCEPT{ return __p; }
```

从汇编代码的角度，这似乎只是做了一次赋值，但经过了布置 `new` 得到的是有 C++ 类型和语义的 pcr 指针，编译器就可以对它进行约束和检查，调用方就能基于它构建各种更高抽象层次的对象。此外，布置 `new` 还会调用构造函数，其中可以放入 GPIO 初始化相关代码，例如对于时钟和电源域的控制，应用相关的一些检查等。

这样的抽象显然还不完整，使用 GPIO 的上层代码不应当知道/记住 GPIO 的端口和管脚。对于一般的应用，上层在初始化的时候设置好这些细节，之后以 LED、按键、阀门等面向对象的方式来使用它们。于是我们可以写出这样的代码：

```
class Led {   public:      inline Led(GpioControlReg* cr, GpioControlReg::Port_t port, GpioControlReg::Pin_t pin) :          m_pcr(cr),         m_port(port),         m_pin(pin)   {}      inline void Toggle()      {         m_pcr->PortToggle(m_port, m_pin);  //GpioControlReg::Pin_t::PIN6      }   private:      GpioControlReg* const m_pcr;      const GpioControlReg::Port_t m_port;      const GpioControlReg::Pin_t m_pin;};
```

这样来调用它：

```
Led led(pcr, Port::PORT1, Pin::PIN6);Key key(pcr, Port::PORT1, Pin::PIN9);if (key.Read() == Level::HIGH){   led.Toggle();}
```

下面我们将这段代码在真实硬件上跑跑看，看看它的行为和开销。在 MCUXpresso IDE 上将优化打到 `-O3`，编译通过后下载到 OKdo-E1 中在调试器中观察。果然，编译器已经领会了我们的意图，`Led` 和 `Key` 的构造函数被完全优化掉了，而 `led.Toggle()` 变成了一行汇编，这没法更短了。所以我们的 GPIO 不但有了更贴切的抽象，也做到了“你不用的东西，你就不需要付出代价；你使用的东西，你手工写代码也不会更好”：

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在工程实际中，类的层级和接口会更为复杂，还有许多因素和细节要考虑。比如，如何考虑多个设备共享的内存映射区域的生命周期；如要支持多态，是要通过虚函数和继承的方式，还是通过元编程；如何避免用户的误用，将对象布置到了不是 MMIO 的内存区域；如何进一步通过泛型编程减少重复代码等，这些话题在 Scott Meyers 的课程讲义中会做进一步探讨。

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

欢迎有兴趣的读者到gitee\_(https://gitee.com/yangwenboolean/cpp_mmio_example)\_下载示例代码并在 OKdo-E1 或者 LPC55 官方 EVK 上进行尝试。现在ARM GCC 以及其他商业编译器都已经广泛支持了现代 C++，更欢迎读者在手头的嵌入式开发板上尝试并分享您的发现。

**04**

**总结**

blinky 这样一个小例子，折射出 C++ 在嵌入式领域的优势。尝试总结几条：

- “语言结构到硬件设备的直接映射”。不仅是映射 CPU 和内存等硬件，直接映射各种外部设备硬件对于嵌入式更有特殊意义。

- “零开销抽象”。同样的业务逻辑，C++ 实现在时空开销上往往能优于于 C。由于语言层面能给编译器更全面的约束信息，有时甚至能做到“负开销”。

- 嵌入式在软件工业中总体上属“后浪”技术，C++ 语言及社区在系统编程领域的积累和优势，能让嵌入式工程师直接利用和借鉴高性能云计算等前沿领域已经验证的技术。

**直播预告**

4月16日晚8点，Boolan 首席软件专家李建忠老师带大家一起探讨\*\*《C++ 系统工程师进阶的“道”和“术”》\*\*：

1、面对庞大复杂的C++，如何升级打怪？

2、C++系统工程师进阶的几个关键点是什么？

3、良好的系统软件设计素养如何建立？

4、如何训练掌握C++的核心思维模型？

![](https://wx.qlogo.cn/finderhead/FrdAUicrPIibf3FFaW0UMG0HxcjnxPNyojXDicFOIwnqvLJG2q5iaOhPkA/0)

**首席C++架构师**

，

04月16日 20:00 直播

已结束

C++ 系统工程师进阶的“道”和“术”

视频号

!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**李建忠**

**Boolan 首席软件专家**

Boolan首席软件专家，全球C++及系统软件技术大会主席。对面向对象、设计模式、软件架构、技术创新有丰富经验和深入研究。主讲《设计模式纵横谈》，《面向对象设计》等课程，影响近百万软件开发人员，享有盛誉。曾于 2005年-2010年期间担任微软最有价值技术专家，区域技术总监。拥有近二十年软件技术架构与产品经验。

Read more

Reads 1132

修改于2022年04月16日

​
