### SST 中的时间表示

在 SST中，时间表示为以下以下两者的组合。

* **时间基准（time base）**：表示时间的基本单位，封装在 `TimeConverter` 对象中；
* **计数值（count）**：表示基本单位的计数值，表示为无符号 64 位整数 `SimTime_t`；

故当前时间可表示为$count * time\_base$，例如，时间基准为 2 皮秒（ps），计数为 1000，那么表示当前的时间是$ 1000*2ps=2000ps=2ns$。

在SST框架中，有两个基本的时间抽象：Core-Time（核心时间）和Local-Time（本地时间）。

* **Core Time**：Core Time表示自模拟开始以来的事件，是绝对的时间尺度。Core-Time-Base（核心时间基准）是仿真器器中的最小时间间隔，默认值为 1 皮秒（ps），可以通过 `getCoreTimeBase()` 获取。当前Core Time由 `SimTime_t` 追踪，通过 `getCurrentSimCycle()` 获取。
* **Local Time**：Local Time是Component（或SubComponent）的本地时间视图，在无需获取绝对时间信息的情况下实现Component，Component不需要关心实际的时钟频率或模拟的绝对时间，只需关注它们自身的时钟周期。例如，一个 CPU Component可以只关心它每个时钟周期的操作，而不需要知道这些周期对应的实际时间是多少。Local-Time-Base（本地时间基准）通过 `TimeConverter` 对象捕获。

在SST框架中，Core Time和Local Time之间转换通过`TimeConverter`对象实现，有以下两个关键接口：

- **`convertToCoreTime()`**：将计数值count从Local Time转换为Core Time视角。例如，若Local-Count为250，Local-Time-Base为 1000 ps，Core-Time-Base为1ps，则返回Core-Count=$250*10^3(ps)/1(ps)=250000$。
- **`convertFromCoreTime()`**：将计数值count从Core Time转换为Local Time视角。例如，若Component的时钟频率为2GHz，Local-Time-Base则为 500ps，在 2ns模拟后调用 `getCurrentSimTime()` 返回 Local-Count=$2*10^3(ps)/500(ps)=4$.

Component（SubComponent）具有默认的Time Base，当未提供特定 `TimeConverter` 时使用。可以通过多种方法设置默认Time Base：

- **显式设置**：直接调用 `setDefaultTimeBase()`；
- **隐式设置**：通过 `registerClock()` 设置时钟周期时隐式设置。当调用 `registerClock()` 方法为Component设置时钟周期时，SST 会隐式地将该时钟周期用作默认Time Base。例如，假设设置Component的时钟为1GHz，则 SST 会将 1 纳秒（1GHz 的周期）作为该Component的默认Time Base；当创建Link连接时，如果没有显式设置Time Based，SST 使用Component的默认Time Base来设置Link的Time Base；在一般情况下，SubComponent会继承父Component的默认Time Base。

```c++
#ifndef _BASICCLOCKS_H
#define _BASICCLOCKS_H

/*
 * 概述：这个示例展示了如何注册时钟和处理器。
 *
 * 涉及的概念：
 *  - 时钟和时钟处理器
 *  - 时间转换器
 *  - 用于错误检查的单位代数
 *
 * 描述：
 *  组件注册了三个时钟，频率是可参数化的。
 *  构造函数使用 UnitAlgebra 来检查每个频率参数是否包含单位。
 *  Clock0 使用 mainTick 作为其处理器。Clock1 和 Clock2 使用 otherTick 作为其处理器
 *  并传递一个 id（分别为 1 和 2），用于标识哪个时钟调用了共享处理器。
 *
 * 模拟：
 *  模拟运行直到 Clock0 执行了 'clockTicks' 个周期。Clock0 在模拟期间会向
 *  STDOUT 打印十次消息（参见 'printInterval' 变量）。这个通知会打印
 *  Clock0 的当前周期，模拟时间，并使用 Clock1 和 Clock2 的时间转换器打印
 *  以 Clock1 和 Clock2 周期为单位的当前时间。
 *  每次调用处理器时，Clock1 和 Clock2 都会打印通知。经过十次滴答后，Clock1 和
 *  Clock2 会暂停自己。
 */

#include <sst/core/component.h>

namespace SST {
namespace simpleElementExample {

// 组件继承自 SST::Component
class basicClocks : public SST::Component
{
public:

/*
 *  SST 注册宏将组件注册到 SST 核心并
 *  文档化其参数、端口等。
 *  SST_ELI_REGISTER_COMPONENT 是必须的，文档化宏
 *  仅在相关时才需要
 */
    // 将此组件注册到元素库中
    SST_ELI_REGISTER_COMPONENT(
        basicClocks,                        // 组件类
        "simpleElementExample",             // 组件库（用于 Python/库查找）
        "basicClocks",                      // 组件名称（用于 Python/库查找）
        SST_ELI_ELEMENT_VERSION(1,0,0),     // 组件版本（与 SST 版本无关）
        "基础：管理时钟示例",              // 描述
        COMPONENT_CATEGORY_UNCATEGORIZED    // 类别
    )

    // 文档化此组件接受的参数
    // { "参数名称", "描述", "默认值或 NULL 如果是必需的" }
    SST_ELI_DOCUMENT_PARAMS(
        { "clock0",     "clock0 的频率或周期（带单位）",     "1GHz" },
        { "clock1",     "clock1 的频率或周期（带单位）",     "5ns" },
        { "clock2",     "clock2 的频率或周期（带单位）",     "15ns" },
        { "clockTicks", "要执行的 clock0 滴答数",    "500" }
    )

    // 可选，因为没有需要文档化的端口
    SST_ELI_DOCUMENT_PORTS()
    
    // 可选，因为没有需要文档化的统计信息
    SST_ELI_DOCUMENT_STATISTICS()

    // 可选，因为没有需要文档化的子组件 - 参见子组件示例获取更多信息
    SST_ELI_DOCUMENT_SUBCOMPONENT_SLOTS()

// 类成员

    // 构造函数。组件接收一个唯一 ID 和在 Python 输入中分配的一组参数。
    basicClocks(SST::ComponentId_t id, SST::Params& params);
    
    // 析构函数
    ~basicClocks();

private:
   
    // clock0 的时钟处理器
    bool mainTick(SST::Cycle_t cycle);

    // clock1 和 clock2 的时钟处理器
    bool otherTick(SST::Cycle_t cycle, uint32_t id);
    
    // 时间转换器 - 参见 sst-core 中的 timeConverter.h/.cc
    // 这些存储一个时钟间隔并可用于时间转换
    TimeConverter* clock1converter;     // clock1 的时间转换器
    TimeConverter* clock2converter;     // clock2 的时间转换器
    Clock::HandlerBase* clock2Handler;  // clock2 的处理器（clock2Tick）

    // 参数
    Cycle_t cycleCount;
    std::string clock0Freq;
    std::string clock1Freq;
    std::string clock2Freq;

    // SST 输出对象，用于打印、错误消息等
    SST::Output* out;
    
    // mainTick 中打印语句之间的周期数
    Cycle_t printInterval;
};

} // namespace simpleElementExample
} // namespace SST

#endif /* _BASICCLOCKS_H */

/*
这个示例展示了如何在 SST 中创建一个组件，注册多个时钟，并处理时钟滴答事件。通过使用时间转换器，可以在不同的时钟频率之间进行转换和比较，体现了 SST 的灵活性和强大功能。
*/
```



```c++

// 这个包含是所有SST实现的***必需***项
#include "sst_config.h"

#include "basicClocks.h"

using namespace SST;
using namespace SST::simpleElementExample;

/* 
 * 在构造函数中，basicClocks 组件应该为模拟做好准备
 * - 读取参数
 * - 注册时钟
 */
basicClocks::basicClocks(ComponentId_t id, Params& params) : Component(id) {

    // SST 输出对象
    out = new Output("", 1, 0, Output::STDOUT);

    // 从 Python 输入中获取参数
    // 时钟可以通过频率或周期指定
    clock0Freq = params.find<std::string>("clock0", "1GHz");
    clock1Freq = params.find<std::string>("clock1", "5ns");
    clock2Freq = params.find<std::string>("clock2", "15ns");
    cycleCount = params.find<Cycle_t>("clockTicks", "500");

    // 使用 UnitAlgebra 进行时钟参数的错误检查
    // 检查所有频率是否具有时间单位
    UnitAlgebra clock0Freq_ua(clock0Freq);
    UnitAlgebra clock1Freq_ua(clock1Freq);
    UnitAlgebra clock2Freq_ua(clock2Freq);
    
    if (! (clock0Freq_ua.hasUnits("Hz") || clock0Freq_ua.hasUnits("s") ) )
        out->fatal(CALL_INFO, -1, "Error in %s: Parameter 'clock0' must have units of Hz or s\n", getName().c_str());
    
    if (! (clock1Freq_ua.hasUnits("Hz") || clock1Freq_ua.hasUnits("s") ) )
        out->fatal(CALL_INFO, -1, "Error in %s: Parameter 'clock1' must have units of Hz or s\n", getName().c_str());
    
    if (! (clock2Freq_ua.hasUnits("Hz") || clock2Freq_ua.hasUnits("s") ) )
        out->fatal(CALL_INFO, -1, "Error in %s: Parameter 'clock2' must have units of Hz or s\n", getName().c_str());

    // 告诉模拟系统不要结束，直到我们准备好
    registerAsPrimaryComponent();
    primaryComponentDoNotEndSim();

    // 注册我们的时钟

    // 主时钟 (clock 0)
    // 时钟可以通过字符串或 UnitAlgebra 注册，这里我们使用字符串
    registerClock(clock0Freq, new Clock::Handler<basicClocks>(this, &basicClocks::mainTick));
    
    out->output("Registered clock0 at %s\n", clock0Freq.c_str());

    // 第二个时钟，这里我们使用 UnitAlgebra 注册
    // 时钟处理程序可以接受模板参数。在这个例子中，clock1 和 clock2 共享一个处理程序，但
    // 传递一个唯一的 ID 来区分
    // 我们还保存 registerClock 的返回值 (一个 TimeConverter) 以供后续使用 (参见 mainTick)
    clock1converter = registerClock(clock1Freq_ua, 
            new Clock::Handler<basicClocks, uint32_t>(this, &basicClocks::otherTick, 1));
    
    out->output("Registered clock1 at %s (which is %s or %s if we convert the UnitAlgebra to string)\n",
            clock1Freq.c_str(), clock1Freq_ua.toString().c_str(), clock1Freq_ua.toStringBestSI().c_str());

    // 最后一个时钟，类似于 clock1，处理程序有一个额外的参数，我们保存 registerClock 的返回值
    Clock::HandlerBase* handler = new Clock::Handler<basicClocks, uint32_t>(this, &basicClocks::otherTick, 2);
    clock2converter = registerClock(clock2Freq, handler);
    
    out->output("Registered clock2 at %s\n", clock2Freq.c_str());

    // 该组件每隔一段时间打印时钟周期和时间，因此根据模拟时间计算打印间隔
    printInterval = cycleCount / 10;
    if (printInterval < 1)
        printInterval = 1;
}


/*
 * 析构函数，清理我们的输出对象
 */
basicClocks::~basicClocks()
{
    delete out;
}


/* 
 * 主时钟 (clock0) 处理程序
 * 此处理程序每 'printInterval' 个周期打印所有时钟的时间和周期
 * 当 'cycleCount' 个周期过去后，此时钟触发模拟结束
 */
bool basicClocks::mainTick( Cycle_t cycles)
{
    // 每隔一段时间打印时钟周期
    // Clock0 周期：按照此时钟经过的周期数
    // Clock1 周期：按照 clock1 经过的周期数，使用 clock1converter 计算
    // Clock2 周期：按照 clock2 经过的周期数，使用 clock2converter 计算
    // Sim 周期：按照 SST 核心周期经过的周期数，从模拟器获取
    // Sim ns：经过的纳秒时间，从模拟器获取
    if (cycles % printInterval == 0) {
        out->output("Clock0 cycles: %" PRIu64 ", Clock1 cycles: %" PRIu64 ", Clock2 cycles: %" PRIu64 ", Sim cycles: %" PRIu64 ", Sim ns: %" PRIu64 "\n",
                cycles, getCurrentSimTime(clock1converter), getCurrentSimTime(clock2converter),
                getCurrentSimCycle(), getCurrentSimTimeNano());
    }

    cycleCount--;

    // 检查是否满足退出条件
    // 如果满足，通知模拟系统结束
    if (cycleCount == 0) {
        primaryComponentOKToEndSim();
        return true;
    } else {
        return false;
    }
}

/*
 * 其他时钟 (clock1 & clock2) 处理程序
 * 让两个时钟运行 10 个周期
 */
bool basicClocks::otherTick( Cycle_t cycles, uint32_t id )
{
    out->output("Clock #%d - Tick count: %" PRIu64 "\n", id, cycles);

    if (cycles == 10) {
        return true; // 10 个周期后停止调用此处理程序
    } else {
        return false; // 如果小于 10 个周期，继续调用此处理程序
    }
}
/*
这个示例展示了如何在 SST 中创建一个组件，注册多个时钟，并处理时钟滴答事件。通过使用 UnitAlgebra 进行参数检查，确保时钟参数的正确性。此外，还展示了如何在模拟过程中打印信息和控制模拟的结束条件。这个代码示例非常适合用于学习和理解 SST 中时钟和事件处理的基本概念。*/
```

