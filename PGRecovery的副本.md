PGRecovery流程分析

**概述**

当PG完成了Peering过程后，处于Active状态的PG就已经可以对外提供服务了。如果该PG的各个副本上有不一致的对象，就需要进行修复。修复过程有两种：Recovery和Backfill；

**前置知识**

1. 线程池和工作队列

线程池封装了pthread接口函数，对外提供线程的入口函数位Thread::entry()。Thread只是消费者的抽象，WorkThread才是线程池具体消费者，它实现了Thread函数接口，入口函数位worker；WorkThread主要执行分三步：首先调用_void_dequeue()方法获取队列元素，然后通过__void_process()处理队列原素，最后使用_void_process_finish函数进行收尾工作。

目前Ceph处理Op采用相同的工作队列即osd::op_shardedwq，后续会将RecoveryOp处理分开，由单独的recovery队列处理。

2. Bufferlist与序列化

Ceph中的bufferlist设计主要包括三个内buffer::raw(bufferraw)、buffer::ptr（bufferptr）、buffer::list(bufferlist)，这三个类都定义在common/buffer.h中，都是buffer的内部类，而buffer类本身没有任何内容，只起到了一个命名空间的作用。

+ buffer::raw对应一段真实的物理内存，负责维护这段物理内存的引用计数nref和释放操作
+ buffer::ptr对应Ceph中的一段被使用过的内存，buffer::raw的一部分或者全部
+ buffer::list 表示一个ptr列表(std::list)，相当于N过ptr构成一个更大的虚拟连续内存

常用集合数据结构的序列化已经由Ceph实现，位于include/encoding.h中，包括以下集合类型：pair, triple，list, set, vector, map, multimap，hash_map, hash_set,dequePGQueable

3. Boost库相关
   1. boost::Variant 

Boost::Variant是定义在boost/variant.hpp中的模板类，它的功能类似与union。Variant是一个模板，所以在使用时必须传入至少一个类型参数。Variant实例所存储的数据类型只能是传入类型中的一种。

​         2.  boost::apply_visitor()与boost::static_visitor

boost::apply_visitor()是boost中提供的函数，该函数的第一个参数必须是boost::static_visitor类型的子类，第二个参数是boost::Variant类型。boost::static_visitor的子类中必须提供operator()()的重载，分别用于处理boost::variant中传入的不同类型的参数。在使用时boost::apply_visitor会自动根据第二个参数类型来调用operator()()。

Example:

```c++
#include <iostream>
#include <boost/variant.hpp>
#include <string>

using namespace std;

struct Var: public boost::static_visitor<>
{
    void operator()(const double &t) 
    {
        cout << "The type is double, the value is: " << t << endl;
    }
    void operator()(const int &t) 
    {
        cout << "The type is int, the value is: " << t << endl;
    }
    void operator()(const string &t)
    {
        cout << "The type is string, the value is: " << t << endl;
    }
};


int main(int argc, char* argv[])
{
    Var v;
    boost::variant<double, int, string> var;
    var = 356;
    boost::apply_visitor(v, var);
    var = 3.14;
    boost::apply_visitor(v, var);
    var = "hello world";
    boost::apply_visitor(v, var);

    return 0;
}
```

Result:

```c++
The type is int, the value is: 356
The type is double, the value is: 3.14
The type is string, the value is: hello world
```

3. boost::optional

optional类位于boost/optional.hpp中，包装了“可能阐释无效值”的对象，实现了“未初始化”的概念。函数并不能总是返回有意义的结果，有时候函数可能返回“无意义”的值，一般来说我们通常使用一个不再正常解空间的一个哨兵来表示无意义的概念，如NULL，-1，end()或者EOF.然后对于不同的应用场合，这些哨兵并不是通用的，而且有时候可能不存在这种解空间之外的哨兵。optional很像一个仅能存放一个元素的容器，它实现了"未初始化"的概念：如果元素未初始化，那么容器就是空的，否则，容器内就是有效的，已经初始化的值。optional的真实接口很复杂，因为它要能包装任何的类型。

Ref:https://zhuanlan.zhihu.com/p/337180080

**代码流程分析**

PGQueable

***

PGQueable类使用了boost的variant类型去定义了成员变量 qvariant，variant类型支持多种方式访问，其中一种就是通过访问者模式来访问。内部结构体 RunVis继承了boost::static_visitor，实现一个variant实例的访问器，且重载了不同类型的函数调用方法"()"；比如Recovery过程实现的重载操作符函数为：void PGQueueable::RunVis::operator()(const PGRecovery &op)，该函数通过调用do_recovery()方法，进入到Recovery过程。

```c++
class PGQueueable {
    typedef boost::variant<
    OpRequestRef,
    PGSnapTrim,
    PGScrub
    > QVariant;   // 定义队列处理的三种请求
    
    QVariant qvariant;
    int cost;
    unsigned priority;
    utime_t start_time;
    entity_inst_t owner;
    epoch_t map_epoch;
    struct RunVis : public boost::static_visitor<> {
        OSD *osd;
        PGRef &pg;
        ThreadPool::TPHandle &handle;
        RunVis(OSD *osd, PGRef &pg, ThreadPool::TPHandle &handle)
            : osd(osd), pg(pg), handle(handle) {}
        void operator()(OpRequestRef &op);
        void operator()(PGSnapTrim &op);
        void operator()(PGScrub &op);
    };
    
public:
    // cppcheck-suppress noExplicitConstructor
    PGQueueable(OpRequestRef op)    // 处理OpRequest
        : qvariant(op), cost(op->get_req()->get_cost()),
          priority(op->get_req()->get_priority()),
          start_time(op->get_req()->get_recv_stamp()),
          owner(op->get_req()->get_source_inst(),
          map_epoch(e),_type(op->_type))
    {}
    PGQueueable(       // 处理PGSnapTrim
        const PGSnapTrim &op, int cost, unsigned priority, utime_t start_time,
        const entity_inst_t &owner)
        : qvariant(op), cost(cost), priority(priority), start_time(start_time),
          owner(owner) ,map_epoch(e),_type(QosOpType::snaptrimop)
    PGQueueable(       // 处理PGScrub
        const PGScrub &op, int cost, unsigned priority, utime_t start_time,
        const entity_inst_t &owner)
        : qvariant(op), cost(cost), priority(priority), start_time(start_time),
          owner(owner),map_epoch(e),_type(QosOpType::scrubop)
    PGQueueable(       // 处理PGrecovery
        const PGRecovey &op, int cost, unsigned priority, utime_t start_time,
        const entity_inst_t &owner)
        : qvariant(op), cost(cost), priority(priority), start_time(start_time),
          owner(owner) ,map_epoch(e),_type(QosOpType::recoveryop)
...
    void run(OSD *osd, PGRef &pg, ThreadPool::TPHandle &handle) {
        RunVis v(osd, pg, handle);
        boost::apply_visitor(v, qvariant);
    }
...
};
```

RecoveryOpWQ

***

RecoveryOpWQ是用于专门处理RecoveryOp的工作队列，仿照ShardedOpWQ编写的，核心处理函数_process

```c++

```

最后调用qi->run(old,pg,tp_handle)函数，实际上会执行OSD::do_recovery()函数



接着调用PrimaryLogPG::start_recovey_ops()开始recovery操作

```c++

```







