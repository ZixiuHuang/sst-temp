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

```c++
// 版权所有 2009-2024 NTESS。根据与 NTESS 的合同 DE-NA0003525 的条款，
// 美国政府保留对该软件的某些权利。
//
// 版权所有 (c) 2009-2024，NTESS
// 保留所有权利。
//
// 部分内容版权归其他开发者所有：
// 有关详细信息，请参阅发行版顶层目录中的 CONTRIBUTORS.TXT 文件。
//
// 该文件是 SST 软件包的一部分。有关许可信息，请参阅发行版顶层目录中的 LICENSE 文件。

#ifndef _BASICLINKS_H
#define _BASICLINKS_H

/*
 * 概要：本示例演示了创建端口和管理链接的不同方式。
 *
 * 覆盖的概念：
 *  - 事件处理程序
 *  - 轮询链接
 *  - 可选端口
 *  - 向量端口
 *  - 统计
 *
 * 描述
 *  该组件有两个端口和一个端口向量。“port_handler”端口使用事件处理程序管理事件到达。“port_polled”端口使用时钟函数进行轮询。
 *  “port_vector%d”端口向量是一组名为 port_vector0、port_vector1 等的端口。这些端口使用类似“port_handler”端口的事件处理程序进行处理。
 *  该组件还为每个端口提供统计数据，统计该端口接收到的字节数。所有端口都期望接收类型为“simpleElementExample::basicEvent”的事件。
 *
 * 模拟
 *  组件配置每个链接并创建时钟，以便可以轮询轮询端口。
 *  1. 每个事件的事件大小在 0 和 eventSize 之间随机选择。
 *  2. 组件使用统计数据来统计接收到的有效负载字节数。
 *
 */

#include <sst/core/component.h>
#include <sst/core/link.h>
#include <sst/core/rng/marsaglia.h>

namespace SST {
namespace simpleElementExample {

// 组件继承自 SST::Component
class basicLinks : public SST::Component
{
public:

/*
 *  SST 注册宏将组件注册到 SST 核心并记录它们的参数、端口等。
 *  SST_ELI_REGISTER_COMPONENT 是必需的，文档宏仅在相关时是必需的。
 */
    // 将此组件注册到元素库中
    SST_ELI_REGISTER_COMPONENT(
        basicLinks,                         // 组件类
        "simpleElementExample",             // 组件库（用于 Python/库查找）
        "basicLinks",                       // 组件名称（用于 Python/库查找）
        SST_ELI_ELEMENT_VERSION(1,0,0),     // 组件版本（与 SST 版本无关）
        "基础：管理链接示例",                // 描述
        COMPONENT_CATEGORY_UNCATEGORIZED    // 类别
    )

    // 记录此组件接受的参数
    // { "参数名称", "描述", "默认值或必需时为 NULL" }
    SST_ELI_DOCUMENT_PARAMS(
        { "eventsToSend", "此组件应发送多少事件。", NULL},
        { "eventSize",    "每个事件的有效负载大小，以字节为单位。", "16"}
    )

    // 记录此组件具有的端口
    // {"端口名称", "描述", { "端口可以处理的事件类型列表"} }
    SST_ELI_DOCUMENT_PORTS(
        {"port_handler",    "链接到另一个组件。此端口使用事件处理程序捕获传入事件。", { "simpleElementExample.basicEvent", ""} },
        {"port_polled",     "链接到另一个组件。此端口使用轮询来捕获传入事件。", { "simpleElementExample.basicEvent", ""} },
        {"port_vector%(portnum)d", "链接到另一个组件。这些端口演示如何从一个端口名称创建端口向量。连接 port_vector0、port_vector1 等。", {"simpleElementExample.basicEvent"} },
    )
    
    // 记录此组件提供的统计数据
    // { "统计名称", "描述", "单位", 启用级别 }
    SST_ELI_DOCUMENT_STATISTICS( 
        {"EventSize_PortHandler", "记录在 port_handler 端口接收到的每个事件的有效负载大小", "字节", 1},
        {"EventSize_PortPolled", "记录在 port_polled 端口接收到的每个事件的有效负载大小", "字节", 1},
        {"EventSize_PortVector", "记录在 port_vector 端口接收到的每个事件的有效负载大小", "字节", 1},
    )

    // 可选，因为没有需要记录的内容 - 请参见 SubComponent 示例以获取更多信息
    SST_ELI_DOCUMENT_SUBCOMPONENT_SLOTS( )

// 类成员

    // 构造函数。组件接收一个唯一的 ID 和在 Python 输入中分配的一组参数。
    basicLinks(SST::ComponentId_t id, SST::Params& params);
    
    // 析构函数
    ~basicLinks();

private:
    // 事件处理程序，在处理程序链接上接收到事件时调用
    void handleEvent(SST::Event *ev);

    // 事件处理程序，在链接向量元素之一上接收到事件时调用
    void handleEventWithID(SST::Event *ev, int linknum);

    // 时钟处理程序，在每个时钟周期调用
    virtual bool clockTic(SST::Cycle_t);

    // 参数
    int64_t eventsToSend;
    int eventSize;
    int lastEventReceived;

    // 随机数生成器
    SST::RNG::MarsagliaRNG* rng;

    // SST 输出对象，用于打印、错误消息等。
    SST::Output* out;

    // 链接
    SST::Link* linkHandler;
    SST::Link* linkPolled;
    std::vector<SST::Link*> linkVector;
    
    // 统计
    Statistic<uint64_t>* stat_portHandler;
    Statistic<uint64_t>* stat_portPolled;
    std::vector<Statistic<uint64_t>*> stat_portVector;

};

} // namespace simpleElementExample
} // namespace SST

#endif /* _BASICLINKS_H */
```

```c++
// 这个 include 是 ***必须的*** 
// 对于所有 SST 实现文件来说
#include "sst_config.h"

#include "basicLinks.h"
#include "basicEvent.h"

using namespace SST;
using namespace SST::simpleElementExample;

/* 
 * 在构造过程中，示例组件应该为仿真做准备
 * - 读取参数
 * - 配置链接
 * - 注册时钟
 * - 初始化 RNG
 * - 注册统计数据
 */
basicLinks::basicLinks(ComponentId_t id, Params& params) : Component(id) {

    // SST 输出对象
    // 初始化
    // - 没有前缀 ("")
    // - 详细信息设置为 1
    // - 没有掩码
    // - 输出到 STDOUT (Output::STDOUT)
    out = new Output("", 1, 0, Output::STDOUT);

    // 从 Python 输入获取参数
    bool found;
    eventsToSend = params.find<int64_t>("eventsToSend", 0, found);

    // 如果没有找到参数，则以退出代码 -1 结束仿真。
    // 告诉用户出错的原因（在输入中没有找到 'eventsToSend' 参数） 
    // 以及哪个组件生成了错误（getName()）
    if (!found) {
        out->fatal(CALL_INFO, -1, "Error in %s: the input did not specify 'eventsToSend' parameter\n", getName().c_str());
    }

    // 此参数控制消息的大小
    // 如果用户没有设置它，则默认参数为 16（字节）
    eventSize = params.find<int64_t>("eventSize", 16);

    // 告诉仿真在我们准备好之前不要结束
    registerAsPrimaryComponent();
    primaryComponentDoNotEndSim();

    // 设置我们的时钟。仿真器将以 1GHz 的频率调用 'clockTic'
    registerClock("1GHz", new Clock::Handler<basicLinks>(this, &basicLinks::clockTic));

    // 此仿真将在我们发送了 'eventsToSend' 个事件并在每个链接上接收到一个 'LAST' 事件时结束
    lastEventReceived = 0;

    // 初始化随机数生成器
    rng = new SST::RNG::MarsagliaRNG(11, 272727);

    
    /*
     *  这个例子的链接
     */
    
    // 1. 这些链接共享
    linkHandler = configureLink("port_handler", new Event::Handler<basicLinks>(this, &basicLinks::handleEvent));
    sst_assert(linkHandler, CALL_INFO, -1, "Error in %s: Link configuration for 'port_handler' failed\n", getName().c_str());


    // 2. 共享事件处理程序的链接。
    // 我们可以选择指定一个 ID，以便我们可以确定事件到达的链接
    // 此外，这还演示了可变数量的端口。有关如何指定这种端口，请参见 SST_ELI_DOCUMENT_PORTS
    std::string linkprefix = "port_vector";
    std::string linkname = linkprefix + "0";
    int portnum = 0;
    while (isPortConnected(linkname)) {
        SST::Link* link = configureLink(linkname, new Event::Handler<basicLinks, int>(this, &basicLinks::handleEventWithID, portnum));
        
        if (!link)
            out->fatal(CALL_INFO, -1, "Error in %s: unable to configure link %s\n", getName().c_str(), linkname.c_str());

        linkVector.push_back(link);
        
        // 构建下一个要检查的名称
        portnum++;
        linkname = linkprefix + std::to_string(portnum);
    }

    // 报告我们找到的链接数量。
    out->output("Component '%s' found %zu links for the vector of links\n", getName().c_str(), linkVector.size());

    // 3. 轮询链接
    linkPolled = configureLink("port_polled");
    sst_assert(linkPolled, CALL_INFO, -1, "Error in %s: Link configuration for 'port_polled' failed\n", getName().c_str());


    /*
     *  此示例的统计数据
     *  我们将统计每个链接上接收到的字节数
     */
    
    // 统计在 port_handler 链接上接收到的字节数
    stat_portHandler = registerStatistic<uint64_t>("EventSize_PortHandler");

    // 统计每个 port_shr 链接上接收到的字节数
    // 这种注册统计数据的版本注册了一个具有多个唯一 "SubID" 字符串（在这种情况下为索引）的统计名称
    // 每个 SubID 都单独统计
    for (size_t i = 0; i < linkVector.size(); i++) {
        Statistic<uint64_t>* stat = registerStatistic<uint64_t>("EventSize_PortVector", std::to_string(i));
        stat_portVector.push_back(stat);
    }

    // 统计在 port_polled 链接上接收到的字节数
    stat_portPolled = registerStatistic<uint64_t>("EventSize_PortPolled");

}


/*
 * 析构函数，清理我们的输出和 RNG
 */
basicLinks::~basicLinks()
{
    delete out;
    delete rng;
}


/* 
 * 无 ID 的事件处理程序
 * 如果多个端口使用此处理程序，我们将无法区分
 * 不同链接上到达的事件
 */
void basicLinks::handleEvent(SST::Event *ev)
{
    basicEvent *event = dynamic_cast<basicEvent*>(ev);
    
    if (event) {

        // 检查这是否是我们邻居将发送给我们的最后一个事件
        if (event->last) {
            lastEventReceived++;
        } else {
            // 记录有效负载的大小
            stat_portHandler->addData(event->payload.size());
        }

        // 接收方有责任删除事件
        delete event;

    } else {
        out->fatal(CALL_INFO, -1, "Error! Bad Event Type received by %s!\n", getName().c_str());
    }
}

/*
 * 带 ID 的事件处理程序
 * 传递给此函数的 'id' 是在构造函数中创建 EventHandler 时注册的常量
 */
void basicLinks::handleEventWithID(SST::Event *ev, int id) {

    basicEvent *event = dynamic_cast<basicEvent*>(ev);
 
    if (event) {
        
        if (event->last) {
            lastEventReceived++;
        } else {
            // 记录有效负载的大小
            stat_portVector[id]->addData(event->payload.size());
        }

        delete event;
    } else {
        out->fatal(CALL_INFO, -1, "Error! Bad Event Type received by %s on link ID %d\n", getName().c_str(), id);
    }
}


/* 
 * 时钟完成三件事:
 * 1. 在随机链接上发送事件，直到我们发送了一定数量的事件。然后在每个链接上发送一个 "LAST" 事件。
 * 2. 轮询轮询链接以检查是否接收到事件
 * 3. 检查退出条件并在此组件完成时通知仿真器
 */
bool basicLinks::clockTic( Cycle_t cycleCount)
{
    // 如果需要，发送一个事件
    if (eventsToSend > 0) {
        basicEvent *event = new basicEvent();
        
        // 使用 RNG 选择一个介于 1 和 eventSize 之间的有效负载大小
        uint32_t size = 1 + (rng->generateNextUInt32() % eventSize);

        // 创建一个大小为 size 的虚拟有效载荷
        for (int i = 0; i < size; i++) {
            event->payload.push_back(1);
        }
        
        eventsToSend--;

        // 使用 RNG 选择一个目标
        uint32_t dest = (rng->generateNextUInt32() % (2 + linkVector.size()));
        if (dest == 0) {
            linkHandler->send(event);
        } else if (dest == 1) {
            linkPolled->send(event);
        } else {
            linkVector[dest - 2]->send(event);
        }
        
        // 这是我们将发送的最后一个事件
        // 通知我们的邻居我们已经完成
        if (eventsToSend == 0) {
            for (int i = 0; i < linkVector.size(); i++) {
                basicEvent *event = new basicEvent();
                event->last = true;

                linkVector[i]->send(event);
            }

            basicEvent* event0 = new basicEvent();
            event0->last = true;
            linkHandler->send(event0);

            basicEvent* event1 = new basicEvent();
            event1->last = true;
            linkPolled->send(event1);
        }
    }


    // 检查轮询链接上是否有事件要接收
    while (SST::Event* ev = linkPolled->recv()) {
        // 静态转换对于发生多次的转换来说更快，但错误的可能性更大
        basicEvent* event = static_cast<basicEvent*>(ev);
        
        // 检查这是否是我们邻居将发送给我们的最后一个事件
        if (event->last) {
            lastEventReceived++;
        } else {
            // 记录有效负载的大小
            stat_portPolled->addData(event->payload.size());
        }
        
        // 接收方有责任删除事件
        delete event;
    }


    // 检查是否满足退出条件
    // 1. 我们已经发送了所有的事件
    // 2. 我们已经接收到所有预期的事件
    if (eventsToSend == 0 && lastEventReceived == (2 + linkVector.size())) {
        
        // 告诉 SST 可以结束仿真了（当所有主要组件都同意时，仿真将结束）
        primaryComponentOKToEndSim(); 

        // 返回 true 表示应禁用此时钟处理程序
        return true;
    }

    // 返回 false 表示不应禁用时钟处理程序
    return false;
}
```

```C++
#ifndef _BASIC_SIM_LIFECYCLE_H
#define _BASIC_SIM_LIFECYCLE_H

/*
 * 这个组件示例演示了 SST 的“生命周期”函数的使用。
 *
 * 这些组件可以在一个环中实例化，每个组件都有一个右链接和一个左链接。
 * 组件相互告知它们希望接收的事件数量，并且每个组件向另一个组件发送请求的事件数量。
 * 在仿真结束时，组件通知彼此这些信息并打印出来。
 * 
 * 仿真生命周期
 * 1) 构造
 *      组件确保它们的链接都已连接
 *      组件读取参数
 * 2) 初始化
 *      组件发现环中其他组件的名称
 * 3) 设置
 *      组件报告它们发现的组件的名称
 *      组件发送初始事件以启动仿真。
 *      这是必需的，因为这是一个纯事件驱动的仿真，因此需要一个事件来启动它。
 * 4) 运行
 *      组件发送和接收消息
 *      组件向左发送事件
 *          - 如果组件接收到一个针对自己的事件，它会删除该事件，并在尚未发送足够事件的情况下发送一个新事件
 *          - 如果组件接收到一个针对其他组件的事件，它会将事件转发给左侧组件
 *      仿真在系统中没有剩余事件时结束
 * 5) 完成
 *      组件向它们的左邻居告别，向右邻居道别
 * 6) 结束
 *      组件打印在 complete() 期间谁告诉了他们什么
 * 7) 销毁
 *      组件清理内存
 *
 * 如果 SST 接收到 SIGINT 或 SIGTERM，每个组件在终止前报告剩余要发送的事件数量
 * 如果 SST 接收到 SIGUSR2，每个组件报告剩余要发送的事件数量并继续运行
 *
 * 涉及的概念：
 *  - 使用 init(), setup(), complete() 和 finish()
 *  - 使用 printStatus() 和 emergencyShutdown()
 */

// SSTSnippet::component-header::start
#include <sst/core/component.h>
#include <sst/core/link.h>

// SSTSnippet::component-header::pause
namespace SST {
namespace simpleElementExample {

// 组件继承自 SST::Component
// SSTSnippet::component-header::start
class basicSimLifeCycle : public SST::Component
{
public:
// SSTSnippet::component-header::pause

/*
 *  SST 注册宏将组件注册到 SST 核心，并
 *  记录它们的参数、端口等。
 *  SST_ELI_REGISTER_COMPONENT 是必需的，文档记录宏
 *  仅在相关时才需要
 */
    // 将此组件注册到元素库中
    SST_ELI_REGISTER_COMPONENT(
        basicSimLifeCycle,              // 组件类
        "simpleElementExample",         // 组件库（用于 Python/库查找）
        "basicSimLifeCycle",            // 组件名称（用于 Python/库查找）
        SST_ELI_ELEMENT_VERSION(1,0,0), // 组件版本（与 SST 版本无关）
        "演示 SST 仿真生命周期的组件", // 描述
        COMPONENT_CATEGORY_PROCESSOR    // 类别
    )

    // 记录此组件接受的参数
    SST_ELI_DOCUMENT_PARAMS(
        { "eventsToSend",   "此组件应向环中的其他组件发送多少事件。", NULL},
        { "verbose",        "设置为 true 时打印此组件接收的每个事件。", "false"}
    )

    // 记录此组件具有的端口
    // {"端口名称", "描述", { "端口可以处理的事件类型列表" } }
    SST_ELI_DOCUMENT_PORTS(
        {"left",  "与另一个组件的左链接", { "simpleElementExample.basicLifeCycleEvent" } },
        {"right", "与另一个组件的右链接", { "simpleElementExample.basicLifeCycleEvent" } }
    )
    
    // SST_ELI_DOCUMENT_STATISTICS 和 SST_ELI_DOCUMENT_SUBCOMPONENT_SLOTS 未声明，因为未使用


// 类成员

    // 构造函数。组件接收一个唯一的 ID 和在 Python 输入中分配的一组参数。
// SSTSnippet::component-header::start
    basicSimLifeCycle(SST::ComponentId_t id, SST::Params& params);
// SSTSnippet::component-header::pause
    
    // 析构函数
// SSTSnippet::component-header::start
    virtual ~basicSimLifeCycle();

// SSTSnippet::component-header::pause
    // 在 SST 的 init() 生命周期阶段由 SST 调用
    virtual void init(unsigned phase) override;

    // 在 SST 的 setup() 生命周期阶段由 SST 调用
    virtual void setup() override;
    
    // 在 SST 的 complete() 生命周期阶段由 SST 调用
    virtual void complete(unsigned phase) override;

    // 在 SST 的 finish() 生命周期阶段由 SST 调用
    virtual void finish() override;

    // 如果 SST 检测到 SIGINT 或 SIGTERM 或发生致命错误时调用
    virtual void emergencyShutdown() override;

    // 如果 SST 接收到 SIGUSR2 时调用
    virtual void printStatus(Output& out) override;

    // 事件处理器，在接收到事件时调用
    void handleEvent(SST::Event *ev);


// SSTSnippet::component-header::start
private:
    // 参数
    unsigned eventsToSend;
// SSTSnippet::component-header::pause
    bool verbose;
// SSTSnippet::component-header::start

    // 组件状态
    unsigned eventsReceived;                // 接收到的事件数量
    unsigned eventsForwarded;               // 转发的事件数量
    unsigned eventsSent;                    // 发送的事件数量（发起的）
    std::set<std::string> neighbors;        // 环上所有邻居的集合
    std::set<std::string>::iterator iter;   // eventRequests Map 中下一个要发送的组件

    // 在 finish 阶段报告的附加状态
    std::string leftMsg, rightMsg;

    // SST 输出对象，用于打印、错误消息等
    SST::Output* out;

    // 链接
    SST::Link* leftLink;
    SST::Link* rightLink;
};
// SSTSnippet::component-header::end

} // namespace simpleElementExample
} // namespace SST

#endif /* _BASIC_SIM_LIFECYCLE_H */
```

```c++
// Copyright 2009-2024 NTESS. Under the terms
// of Contract DE-NA0003525 with NTESS, the U.S.
// Government retains certain rights in this software.
//
// Copyright (c) 2009-2024, NTESS
// All rights reserved.
//
// Portions are copyright of other developers:
// See the file CONTRIBUTORS.TXT in the top level directory
// of the distribution for more information.
//
// This file is part of the SST software package. For license
// information, see the LICENSE file in the top level directory of the
// distribution.


// 这个包含是 ***必须的*** 
// 对于所有的 SST 实现文件
#include "sst_config.h"

#include "basicSimLifeCycle.h"
#include "basicSimLifeCycleEvent.h"


using namespace SST;
using namespace SST::simpleElementExample;

/* 
 * 生命周期阶段 #1: 构造
 * - 配置输出对象
 * - 确保两个链接都已连接并配置
 * - 读取参数
 * - 初始化内部状态
 */
basicSimLifeCycle::basicSimLifeCycle(ComponentId_t id, Params& params) : Component(id) 
{
    // SST 输出对象
    // 初始化 
    // - 无前缀 ("")
    // - 详细级别设置为 1
    // - 无掩码
    // - 输出到标准输出 (Output::STDOUT)
    out = new Output("", 1, 0, Output::STDOUT);
    
    out->output("阶段: 构造, %s\n", getName().c_str());

    // 从 Python 输入中获取参数
    bool found;
    eventsToSend = params.find<unsigned>("eventsToSend", 0, found);

    // 如果参数未找到，结束模拟并返回退出代码 -1。
    // 告知用户如何修复错误（在输入中设置 'eventsToSend' 参数）
    // 以及哪个组件生成了错误（getName()）
    if (!found) {
        out->fatal(CALL_INFO, -1, "错误在 %s: 输入未指定 'eventsToSend' 参数\n", getName().c_str());
    }

    // 详细参数以进一步控制输出
    verbose = params.find<bool>("verbose", false);

    // 使用回调函数配置我们的链接，该回调函数将在事件到达时调用
    leftLink = configureLink("left", new Event::Handler<basicSimLifeCycle>(this, &basicSimLifeCycle::handleEvent));
    rightLink = configureLink("right", new Event::Handler<basicSimLifeCycle>(this, &basicSimLifeCycle::handleEvent));

    // 确保我们成功配置了链接
    // 失败通常意味着用户未在输入文件中连接端口
    sst_assert(leftLink, CALL_INFO, -1, "错误在 %s: 左链接配置失败\n", getName().c_str());
    sst_assert(rightLink, CALL_INFO, -1, "错误在 %s: 右链接配置失败\n", getName().c_str());

    // 注册为主要组件并防止模拟结束，直到我们接收到所有需要的事件
    registerAsPrimaryComponent();
    primaryComponentDoNotEndSim();

    // 初始化我们的事件计数变量
    eventsReceived = 0;
    eventsForwarded = 0;
    eventsSent = 0;
}

/*
 * 生命周期阶段 #2: 初始化
 * 环中的组件将发现它们的邻居名称并同意使用每个邻居的平均 eventsToSend 数量。
 * 
 * - 发送我们的 eventsToSend 参数和名称到环中
 * - 记录来自其他组件的信息
 * - 在 init() 期间，'sendUntimedData' 函数必须使用而不是 'send'
 */
void basicSimLifeCycle::init(unsigned phase) 
{
    out->output("阶段: 初始化(%u), %s\n", phase, getName().c_str());
    
    // 只在阶段 0 发送我们的信息
    if (phase == 0) {
        basicLifeCycleEvent* event = new basicLifeCycleEvent(getName(), eventsToSend);
        leftLink->sendUntimedData(event);
    }

    // 检查是否接收到事件。recvUntimedData 如果没有可用事件则返回 nullptr
    while (SST::Event* ev = rightLink->recvUntimedData()) {

        basicLifeCycleEvent* event = dynamic_cast<basicLifeCycleEvent*>(ev);
        if (event) {
            if (verbose)
                out->output("    %" PRIu64 " %s 收到 %s\n", getCurrentSimCycle(), getName().c_str(), event->toString().c_str());

            if (event->getStr() == getName()) { // 事件环绕到环路并返回到此组件
                delete event;
            } else { // 事件来自另一个组件
                neighbors.insert(event->getStr());
                eventsToSend += event->getNum();
                leftLink->sendUntimedData(event);
            }

        } else {
            out->fatal(CALL_INFO, -1, "错误在 %s: 在 init() 期间收到一个事件，但它不是预期的类型\n", getName().c_str());
        }
    }
}

/*
 * 生命周期阶段 #3: 设置
 * - 发送第一个事件
 *   这个发送应该使用 link->send 函数，因为它是为模拟而设计的，而不是
 *   预模拟（init）或后模拟（complete）
 * - 初始化 'iter' 变量以指向 eventRequests 映射中的下一个组件
 */
void basicSimLifeCycle::setup() 
{
    out->output("阶段: 设置, %s\n", getName().c_str());
    
    // 使用每个组件的 eventsToSend 参数的平均值来同意 eventsToSend
    // 然后，在模拟期间总共发送的事件是我们的邻居数 * 每个邻居发送的事件数
    eventsToSend /= (neighbors.size() + 1); // 加一因为我不在邻居列表中

    out->output("    %s 将发送 %u 个事件给每个其他组件。\n", getName().c_str(), eventsToSend);

    eventsToSend *= neighbors.size(); // 总共要发送的事件数

    // 健全性检查
    if (neighbors.empty()) {
        out->output("    我的名字是 %s。这里很孤独，我没有邻居。\n", getName().c_str());
        primaryComponentOKToEndSim();
        return;
    } else if (eventsToSend == 0) {
        out->output("    我的名字是 %s。我有邻居，但我们谁都不想说话。\n", getName().c_str());
        primaryComponentOKToEndSim();
        return;
    }

    // 由于所有集合的顺序相同，将我们的起始邻居错开到我们之后的一个
    iter = neighbors.upper_bound(getName());
    if (iter == neighbors.end()) iter = neighbors.begin();

    // 发送第一个事件
    leftLink->send(new basicLifeCycleEvent(*iter));
    
    // 记录我们发送了这个事件
    eventsSent++;

    // 更新 iter 为下一次发送做准备
    iter++;
    if (iter == neighbors.end()) iter = neighbors.begin();

    out->output("阶段: 运行, %s\n", getName().c_str());
}


/* 
 * 生命周期阶段 #4: 运行
 *
 * 在运行阶段，SST 将在接收到链接上的事件时调用此事件处理程序
 * - 记录我们收到一个事件（eventsForwarded 或 eventsReceived）
 * - 如果事件是针对我们的，则删除事件
 * - 如果事件是针对其他人的，则转发事件
 * - 发送一个新事件
 */
void basicSimLifeCycle::handleEvent(SST::Event *ev)
{
    basicLifeCycleEvent *event = dynamic_cast<basicLifeCycleEvent*>(ev);
    
    if (event) {
        if (verbose) out->output("    %" PRIu64 " %s 收到 %s\n", getCurrentSimCycle(), getName().c_str(), event->toString().c_str());
       
        // 确定事件是否是针对我们的，并相应处理
        if (event->getStr() == getName()) {
            eventsReceived++;
            delete event;
            
            if (eventsReceived == eventsToSend) 
                primaryComponentOKToEndSim();

            // 如果我们需要，发送一个新事件
            if (eventsSent != eventsToSend) {
                leftLink->send(new basicLifeCycleEvent(*iter));
                eventsSent++;
                iter++;
                if (iter == neighbors.end()) iter = neighbors.begin();
            } 

        } else {
            /* 否则，事件不是针对我们的。转发它。 */
            eventsForwarded++;
            leftLink->send(event);
        }
        
    } else {
        out->fatal(CALL_INFO, -1, "错误在 %s: 在模拟期间收到一个事件，但它不是预期的类型\n", getName().c_str());
    }
}

/*
 * 生命周期阶段 #5: 完成
 *
 * 在完成阶段，向我们的左邻居说再见并向我们的右邻居告别
 */
void basicSimLifeCycle::complete(unsigned phase)
{
    out->output("阶段: 完成(%u), %s\n", phase, getName().c_str());
    
    if (phase == 0) {
        std::string goodbye = "来自 " + getName() + " 的再见";
        std::string farewell = "来自 " + getName() + " 的告别";
        leftLink->sendUntimedData( new basicLifeCycleEvent(goodbye) );
        rightLink->sendUntimedData( new basicLifeCycleEvent(farewell) );
    }

    // 检查左链接上的事件
    while (SST::Event* ev = leftLink->recvUntimedData()) {
        basicLifeCycleEvent* event = dynamic_cast<basicLifeCycleEvent*>(ev);
        if (event) {
            if (verbose) out->output("    %" PRIu64 " %s 收到 %s\n", getCurrentSimCycle(), getName().c_str(), event->toString().c_str());
            leftMsg = event->getStr();
            delete event;
        } else {
            out->fatal(CALL_INFO, -1, "错误在 %s: 在 complete() 期间收到一个事件，但它不是预期的类型\n", getName().c_str());
        }
    }

    // 检查右链接上的事件
    while (SST::Event* ev = rightLink->recvUntimedData()) {
        basicLifeCycleEvent* event = dynamic_cast<basicLifeCycleEvent*>(ev);
        if (event) {
            if (verbose) out->output("    %" PRIu64 " %s 收到 %s\n", getCurrentSimCycle(), getName().c_str(), event->toString().c_str());
            rightMsg = event->getStr();
            delete event;
        } else {
            out->fatal(CALL_INFO, -1, "错误在 %s: 在 complete() 期间收到一个事件，但它不是预期的类型\n", getName().c_str());
        }
    }
}

/*
 * 生命周期阶段 #6: 结束
 * 在 finish() 阶段，输出我们在 complete() 期间收到的信息
 */
void basicSimLifeCycle::finish() 
{
    out->output("阶段: 结束, %s\n", getName().c_str());
    
    out->output("    我的名字是 %s，我发送了 %u 条消息。我收到了 %u 条消息并转发了 %u 条消息。\n"
            "    我的左邻居说：%s\n"
            "    我的右邻居说：%s\n",
            getName().c_str(), eventsSent, eventsReceived, eventsForwarded,
            leftMsg.c_str(), rightMsg.c_str());
}

/*
 * 生命周期阶段 #7: 析构
 * 清理我们的输出对象
 * SST 将删除链接
 */
basicSimLifeCycle::~basicSimLifeCycle()
{
    // 组件信息已被核心删除，因此 getName() 将不起作用
    out->output("阶段: 析构\n");
    delete out;
}

/*
 * 紧急关闭
 * 尝试向模拟发送 SIGTERM
 */
void basicSimLifeCycle::emergencyShutdown() {
    out->output("呃哦，我的名字是 %s，我必须退出。我发送了 %u 条消息。\n", getName().c_str(), eventsSent);
}

/* 
 * 打印状态
 * 尝试向模拟发送 SIGUSR2
 */
void basicSimLifeCycle::printStatus(Output& sim_out) {
    sim_out.output("%s 报告。我发送了 %u 条消息，收到了 %u 条，并转发了 %u 条。\n", 
            getName().c_str(), eventsSent, eventsReceived, eventsForwarded);
}
```

```c++
// 版权所有 2009-2024 NTESS。根据与 NTESS 签订的合同 DE-NA0003525 的条款，
// 美国政府保留对本软件的某些权利。
//
// 版权所有 (c) 2009-2024, NTESS
// 保留所有权利。
//
// 部分版权归其他开发者所有：
// 有关更多信息，请参见分发目录顶层的文件 CONTRIBUTORS.TXT。
//
// 本文件是 SST 软件包的一部分。有关许可信息，
// 请参见分发目录顶层的 LICENSE 文件。

#ifndef _BASIC_SIM_LIFECYCLE_EVENT_H
#define _BASIC_SIM_LIFECYCLE_EVENT_H

#include <sst/core/event.h>

namespace SST {
namespace simpleElementExample {

/*
 * basicSimLifeCycle 示例的事件
 * 根据模拟的阶段，有时发送字符串，有时发送无符号整数
 * 在所有阶段使用相同的事件以简化代码
 */

class basicLifeCycleEvent : public SST::Event
{
public:
    // 构造函数
    basicLifeCycleEvent() : SST::Event(), str(""), num(0) { } /* 该构造函数是为了序列化而必需的 */
    basicLifeCycleEvent(std::string val) : SST::Event(), str(val), num(0) { }
    basicLifeCycleEvent(std::string sval, unsigned uval) : SST::Event(), str(sval), num(uval) { }
    basicLifeCycleEvent(unsigned val) : SST::Event(), str(""), num(val) { }
    
    // 析构函数
    ~basicLifeCycleEvent() { }

    std::string getStr() { return str; }
    unsigned getNum() { return num; }

    std::string toString() const override {
        std::stringstream s;
        s << "basicLifeCycleEvent. String='" << str << "' Number='" << num << "'";
        return s.str();
    }

private:
    std::string str;
    unsigned num;

    // 序列化
    void serialize_order(SST::Core::Serialization::serializer &ser)  override {
        Event::serialize_order(ser);
        ser & str;
        ser & num;
    }

    // 注册此事件为可序列化
    ImplementSerializable(SST::simpleElementExample::basicLifeCycleEvent);
};

} // namespace simpleElementExample
} // namespace SST

#endif /* _BASIC_SIM_LIFECYCLE_EVENT_H */
```

