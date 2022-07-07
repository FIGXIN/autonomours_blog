- [apollo介绍之Cyber框架(十) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/91322837)

关于cyber的代码也看了很长一段时间，之前一直想写一篇关于cyber的介绍，怎奈迟迟没有动笔，一是cyber的篇幅实在过长，二是有些方面也没有完全看懂。终于下定决心来写一篇cyber的长文已经到了11月份，cyber有很多我们值得学习的地方，就算是仅仅出于技术原因，我也非常推荐学习下cyber的源码。

由于cyber有些流程很长，导致看代码的时候喜欢画很复杂的流程图，**实际上把图画长难，把图画小更难**，百度apollo cyber的框图都很短小，但都直击要害，说出每个模块的具体功能。下面是简单的对比[[1\]](https://zhuanlan.zhihu.com/p/91322837#ref_1)。

![img](https://pic4.zhimg.com/80/v2-faa4c3cd6166e2d7f6aaaff8e9ebd63b_720w.jpg)

长流程图

![img](https://pic2.zhimg.com/80/v2-2947ce303e7082e1d15f7e343785083d_720w.jpg)

## Cyber实现的功能

cyber提供的功能概括起来包括2方面：

1. **消息队列** - 主要作用是接收和发送各个节点的消息，涉及到消息的发布、订阅以及消息的buffer缓存等。
2. **实时调度** - 主要作用是调度处理上述消息的算法模块，保证算法模块能够实时调度处理消息。

除了这2方面的工作，cyber还需要提供以下2部分的工作：

1. **用户接口** - 提供灵活的用户接口
2. **工具** - 提供一系列的工具，例如bag包播放，点云可视化，消息监控等

![img](https://pic3.zhimg.com/80/v2-47c413a2d2c108ce3d4354a439d6e4e2_720w.jpg)

总结起来就是，**cyber是一个分布式收发消息，和调度框架，同时对外提供一系列的工具和接口来辅助开发和定位问题**。其中cyber对比ROS来说有很多优势，唯一的劣势是cyber相对ROS没有丰富的算法库支持。



下面我们开始分析整个cyber的代码流程。

## cyber入口

cyber的入口在"cyber/mainboard"目录中：

```text
├── mainboard.cc           // 主函数
├── module_argument.cc     // 模块输入参数
├── module_argument.h
├── module_controller.cc   // 模块加载，卸载
└── module_controller.h
```

mainboard中的文件比较少，也很好理解，我们先从"[mainboard.cc](https://link.zhihu.com/?target=http%3A//mainboard.cc/)"中开始分析：

```text
int main(int argc, char **argv) {
  google::SetUsageMessage("we use this program to load dag and run user apps.");
  
  // 注册信号量，当出现系统错误时，打印堆栈信息
  signal(SIGSEGV, SigProc);
  signal(SIGABRT, SigProc);
  
  // parse the argument
  // 解析参数
  ModuleArgument module_args;
  module_args.ParseArgument(argc, argv);

  // initialize cyber
  // 初始化cyber
  apollo::cyber::Init(argv[0]);
  
  // start module
  // 加载模块
  ModuleController controller(module_args);
  if (!controller.Init()) {
    controller.Clear();
    AERROR << "module start error.";
    return -1;
  }
  
  // 等待cyber关闭
  apollo::cyber::WaitForShutdown();
  // 卸载模块
  controller.Clear();
  AINFO << "exit mainboard.";

  return 0;
}
```

上述是"[mainboard.cc](https://link.zhihu.com/?target=http%3A//mainboard.cc/)"的主函数，下面我们重点介绍下具体的过程。



**打印堆栈**

在主函数中注册了信号量"SIGSEGV"和"SIGABRT"，当系统出现错误的时候（空指针，异常）等，这时候就会触发打印堆栈信息，也就是说系统报错的时候打印出错的堆栈，方便定位问题[[2\]](https://zhuanlan.zhihu.com/p/91322837#ref_2)。

```text
  // 注册信号量，当出现系统错误时，打印堆栈信息
  signal(SIGSEGV, SigProc);
  signal(SIGABRT, SigProc);
```

打印堆栈的函数在"SigProc"中实现，而打印堆栈的实现是通过"backtrace"实现[[3\]](https://zhuanlan.zhihu.com/p/91322837#ref_3)。

```text
// 打印堆栈信息
void ShowStack() {
  int i;
  void *buffer[STACK_BUF_LEN];
  int n = backtrace(buffer, STACK_BUF_LEN);
  char **symbols = backtrace_symbols(buffer, n);
  AINFO << "=============call stack begin:================";
  for (i = 0; i < n; i++) {
    AINFO << symbols[i];
  }
  AINFO << "=============call stack end:================";
}
```



**解析参数**

解析参数是在"ModuleArgument"类中实现的，主要是解析加载DAG文件时候带的参数。

```text
void ModuleArgument::ParseArgument(const int argc, char* const argv[]) {
  // 二进制模块名称
  binary_name_ = std::string(basename(argv[0]));
  // 解析参数
  GetOptions(argc, argv);

  // 如果没有process_group_和sched_name_，则赋值为默认值
  if (process_group_.empty()) {
    process_group_ = DEFAULT_process_group_;
  }

  if (sched_name_.empty()) {
    sched_name_ = DEFAULT_sched_name_;
  }

  // 如果有，则设置对应的参数
  GlobalData::Instance()->SetProcessGroup(process_group_);
  GlobalData::Instance()->SetSchedName(sched_name_);
  AINFO << "binary_name_ is " << binary_name_ << ", process_group_ is "
        << process_group_ << ", has " << dag_conf_list_.size() << " dag conf";

  // 打印dag_conf配置，这里的dag是否可以设置多个？？？
  for (std::string& dag : dag_conf_list_) {
    AINFO << "dag_conf: " << dag;
  }
}
```



**模块加载**

在"ModuleController"实现cyber模块的加载，在"ModuleController::Init()"中调用"LoadAll()"来加载所有模块，我们接着看cyber是如何加载模块。

\1. 首先是找到模块的路径

```text
    if (module_config.module_library().front() == '/') {
      load_path = module_config.module_library();
    } else {
      load_path =
          common::GetAbsolutePath(work_root, module_config.module_library());
    }
```

\2. 通过"class_loader_manager_"加载模块，后面我们会接着分析"ClassLoaderManager"的具体实现，加载好对应的类之后在创建对应的对象，并且初始化对象（调用对象的Initialize()方法，也就是说所有的cyber模块都是通过Initialize()方法启动的，后面我们会接着分析Initialize具体干了什么）。

这里的"classloader"其实类似java中的classloader，即java虚拟机在运行时加载对应的类，并且实例化对象。

cyber中其实也是实现了类型通过动态加载并且实例化类的功能，好处是可以动态加载和关闭单个cyber模块(定位，感知，规划等)，也就是**在dreamview中的模块开关按钮，实际上就是动态的加载和卸载对应的模块**。

```text
    // 通过类加载器加载load_path下的模块
    class_loader_manager_.LoadLibrary(load_path);

    // 加载模块
    for (auto& component : module_config.components()) {
      const std::string& class_name = component.class_name();
      // 创建对象
      std::shared_ptr<ComponentBase> base =
          class_loader_manager_.CreateClassObj<ComponentBase>(class_name);
      // 调用对象的Initialize方法
      if (base == nullptr || !base->Initialize(component.config())) {
        return false;
      }
      component_list_.emplace_back(std::move(base));
    }

    // 加载定时器模块
    for (auto& component : module_config.timer_components()) {
      const std::string& class_name = component.class_name();
      std::shared_ptr<ComponentBase> base =
          class_loader_manager_.CreateClassObj<ComponentBase>(class_name);
      if (base == nullptr || !base->Initialize(component.config())) {
        return false;
      }
      component_list_.emplace_back(std::move(base));
    }
```

上述就是cyber mainboard的整个流程，**cyber main函数中先解析dag参数，然后根据解析的参数，通过类加载器动态的加载对应的模块，然后调用Initialize方法初始化模块**。



下面我们会接着分析ClassLoaderManager

## 类加载器(class_loader)

类加载器的作用就是动态的加载动态库，然后实例化对象。我们先来解释下，首先apollo中的各个module都会编译为一个动态库，拿planning模块来举例子，在"planning/dag/planning.dag"中，会加载：

```text
module_config {
  module_library : "/apollo/bazel-bin/modules/planning/libplanning_component.so"
```

也就是说，apollo中的模块都会通过类加载器以动态库的方式加载，然后实例化，之后再调用Initialize方法初始化。也就是说，我们讲清楚下面3个问题，也就是讲清楚了类加载器的原理。

1. cyber如何加载apollo模块？
2. 如何实例化模块?
3. 如何初始化模块?



**目录结构**

类加载器的实现在"cyber/class_loader"目录中，通过"Poco/SharedLibrary.h"库来实现动态库的加载，关于Poco动态库的加载可以[参考]([Class Poco::SharedLibrary](https://link.zhihu.com/?target=https%3A//pocoproject.org/docs/Poco.SharedLibrary.html))

```text
├── BUILD                  // 编译文件
├── class_loader.cc        // 类加载器
├── class_loader.h
├── class_loader_manager.cc  // 类加载器管理
├── class_loader_manager.h
├── class_loader_register_macro.h  // 类加载器注册宏定义
└── utility
    ├── class_factory.cc           // 类工厂
    ├── class_factory.h
    ├── class_loader_utility.cc    // 类加载器工具类
    └── class_loader_utility.h
```



**类加载器(ClassLoader)**

我们先从"class_loader.h"开始看起，首先我们分析下"class_loader"实现的具体方法：

```text
class ClassLoader {
 public:
  explicit ClassLoader(const std::string& library_path);
  virtual ~ClassLoader();

  // 库是否已经加载
  bool IsLibraryLoaded();
  // 加载库
  bool LoadLibrary();
  // 卸载库
  int UnloadLibrary();
  // 获取库的路径
  const std::string GetLibraryPath() const;
  // 获取累名称
  template <typename Base>
  std::vector<std::string> GetValidClassNames();
  // 实例化类对象
  template <typename Base>
  std::shared_ptr<Base> CreateClassObj(const std::string& class_name);
  // 类是否有效
  template <typename Base>
  bool IsClassValid(const std::string& class_name);

 private:
  // 当类删除
  template <typename Base>
  void OnClassObjDeleter(Base* obj);

 private:
  // 类的路径
  std::string library_path_;
  // 类加载引用次数
  int loadlib_ref_count_;
  // 类加载引用次数锁
  std::mutex loadlib_ref_count_mutex_;
  // 类引用次数
  int classobj_ref_count_;
  // 类引用次数锁
  std::mutex classobj_ref_count_mutex_;
};
```

可以看到类加载器主要是提供了加载类，卸载类和实例化类的接口。实际上加载类和卸载类的实现都比较简单，都是调用"utility"类中的实现，我们暂时先放一边，先看下实例化对象的实现。

```text
template <typename Base>
std::shared_ptr<Base> ClassLoader::CreateClassObj(
    const std::string& class_name) {
  // 加载库
  if (!IsLibraryLoaded()) {
    LoadLibrary();
  }

  // 根据类名称创建对象
  Base* class_object = utility::CreateClassObj<Base>(class_name, this);

  // 类引用计数加1
  std::lock_guard<std::mutex> lck(classobj_ref_count_mutex_);
  classobj_ref_count_ = classobj_ref_count_ + 1;
  // 指定类的析构函数
  std::shared_ptr<Base> classObjSharePtr(
      class_object, std::bind(&ClassLoader::OnClassObjDeleter<Base>, this,
                              std::placeholders::_1));
  return classObjSharePtr;
}
```

可以看到创建类的时候，类引用计数加1，并且绑定类的析构函数(OnClassObjDeleter)，删除对象的时候让类引用计数减1。

```text
template <typename Base>
void ClassLoader::OnClassObjDeleter(Base* obj) {
  if (nullptr == obj) {
    return;
  }

  std::lock_guard<std::mutex> lck(classobj_ref_count_mutex_);
  delete obj;
  --classobj_ref_count_;
}
```



我们先简单的分析下ClassLoaderManager，最后再分析utility。

**ClassLoaderManager**

类加载器管理实际上是管理不同的classloader，而不同的libpath对应不同的classloader。ClassLoaderManager主要的数据结构其实如下：

```text
std::map<std::string, ClassLoader*> libpath_loader_map_;
```

其中"libpath_loader_map_"为map结构，在"LoadLibrary"的时候赋值，**key为library_path，而value为ClassLoader**.

```text
bool ClassLoaderManager::LoadLibrary(const std::string& library_path) {
  std::lock_guard<std::mutex> lck(libpath_loader_map_mutex_);
  if (!IsLibraryValid(library_path)) {
    // 赋值
    libpath_loader_map_[library_path] =
        new class_loader::ClassLoader(library_path);
  }
  return IsLibraryValid(library_path);
}
```

也就是说"ClassLoaderManager"对ClassLoader进行保存和管理。