---
title: rlottie -- Animation Parser on CPP Client
author: Jalen Cui
comments: true
categories: CPP
date: 2023-02-18 09:15:11
tags:
---


# 预备知识
* Concurrency In Action: __single-producer, single-consumer__
    + Lock-free concurrent data structure: 
        > lock-free data structures rely on the use of __atomic operations__ and the associated __memory-ordering__ guarantees in order to ensure that data becomes __visible__ to other threads in the __correct order__.  
    
    + Guildllines: 
        > - use `std::memory_order_seq_cst` for prototyping 
        > - use a lock-free memory reclamation scheme 
        > - simplify it to the context: 
          + only one thread calling `push()` at a time 
          + only one thread calling `pop()` at a time 
        > - watch out for the ABA problem
        > - identify busy-wait loops and help the other thread

* [Lottie](https://lottiefiles.com):
> A lottie is JSON-based animation file format that allows you to ship animations on any playform as easily as shipping static assets.  

    + Parse Lottie:
        - [LottieAnimation](https://doc.qt.io/qt-6/qml-qt-labs-lottieqt-lottieanimation.html): a bodymovin player for Qt 
        - [rlottie](https://github.com/Samsung/rlottie): a platform independent standalone c++ library for rendering vector based animations and art __IN REALTIME__

    + Tips on rlottie:
        - render mode: synchronize (`rendersync`) and asynchronzie (`render`)
        - cache policy: load from data with corresponding __key__ to distinguish different cache 
# 简介  

客户端中使用动画的业务场景大致相同, 如下范式可供参考:  
>本博主负责的 AR 直播 (1080p59.94) 项目中使用过类似范式： 16.6ms 内使用最少的 cpu 时间处理 4 路输出, 而每路每帧的 1080p 图片 `copy` 耗时 3ms 左右哦.

* desigen structure:
    + 设计 __单生产单消费者__ 数据结构: `lock_free_queue`
    + 分析业务场景并简化之，以适用于 `lock_free_queue`
* request render: 
    + 通过异步接口 `render()` 请求渲染一帧图片，并得到对应的 `future` 对象.  
    + 将 `future` 对象存至对应的 `lock_free_queue`: `render_queue`
* process request: 遍历所有 `render_queue` 中的所有 `future` 对象 (得益于良好的数据结构，业务逻辑无需锁) 
# 准备阶段  
本阶段就是要设计 (拿来主义) 对应的 __无锁__ 数据结构啦, 一切从简, 就简单贴出标准的接口.
```cpp
template<typename T>
class lock_free_queue
{
private:
    struct node
    {
        std::shared_ptr<T> data;
        ndoe* next;
        node():next(nullptr){}
    };
    std::atomic<ndoe*> head;
    std::atomic<node*> tail;
    node* pop_head(){
        node* const old_head = head.load();
        if(old_head == tail.load()) {                                     // <order> 1
            return nullptr;
        }
        head.store(old_head->next);
        return old_head;
    }
public: 
    lock_free_queue():head(new node), tail(head.load()){}
    lock_free_queue(const lock_free_queue& other)=delete;
    lock_free_queue& operator=(const lock_free_queue& other)=delete;
    ~lock_free_queue();

    std::shared_ptr<T> pop(){
        node* old_head=pop_head();
        if(!old_head) return std::shared_ptr<T>();
        std::shared_ptr<T> const res(old_head->data);                     // <order> 2 
        delete old_head;
        return res;
    }
    void push(T new_value){
        std::shared_ptr<T> new_data(std::make_shared<T>(new_value));
        node* p = new node;                                               // <order> 3
        node* const old_tail = tail.load();                               // <order> 4
        old_tail->data.swap(new_data);                                    // <order> 5
        old_tail->next = p;                                               // <order> 6
        tail.store(p);                                                    // <order> 7
    }
};
```

# 代码实现
本阶段仅仅列出 `rlottie` 接口相关的内容. 
>__预备知识__ 和 __简介__ 是理论和实践经验的陈述. 其中涉及的异步处理框架, 需要根据项目的业务场景和采用的设计模式 (MVC, MVVM) 进行定制. 
* parseLottie: `lottie` 初始化, 包括: Animation, Surface 对象的构造, 堆内存分配
```cpp
void parseLottie(std::string lottie_file)
{
     // load data
     auto lottieData = util::loadFromFile(lottie_file); // load lottie.json file from lottie_file
     // parse lottie and contruct `Animation`
     lottieAnimation = Animation::loadFromData(lottieData, lottie_file); // ****key: lottie_file****
     // alloc 1 frame buffer and construct `Surface`
     size_t bytesPerLine = cols * sizeof(unint32_t); // row bytes to iterate
     auto lottieBuffer = (uint32_t*)malloc(bytesPerline * rows);
     lottieSurface = Surface(lottieBuffer, cols, rows, bytesPerline);
}
```
* requestRender: 以 `lottie` 动画的帧率, 调用该接口完成异步渲染相关动作
```cpp
void requestRender()
{
    auto future = lottieAnimation->render(currentFrame, lottieSurface, true);
    currentFrame = (currentFrame + 1) % lottieAnimation->totalFrame();
    // lock-free queue
    lottieQueue->push(std::make_pair(key_to_index, future));
}
```

# 插件集成 -- Windows
* __Build rlottie__
    + Install [make](https://gnuwin32.sourceforge.net/packages/make.htm) on Windows: in case of `command not found` on `make -j 2`
    + Build libs for Windows: no need to flow [rlottie](https://github.com/Samsung/rlottie)
        - step 1: `cmake ..` produces Visual Studio Solution (default on Windows)
        - step 2: open _rlottie.sln_ and build it with Visual Studio
* __Integrate rlottie__
    + Header Files:
        - Visual Studio: setup _Additional Include Directories_
        - CMakeLists
            <table style="margin-bottom: 8px">
            <tbody>
            <tr style="border: none">
                <td style="background: black; border: none; width: 100vw; border-radius: 4px">
                    <ol style="margin:0">
                        <li align="left"><b>
                            <font size=2 color=steelblue>include_directories</font><font size=2 color=silver>&ensp;(</font>
                        </b></li>
                        <li align="left"><b>
                            <font size=2 color=mediumseagreen>&emsp;&emsp;${CMAKE_SOURCE_DIR}</font><font size=2 color=silver>/rlottie/</font><font size=2 color=skyblue>include</font>
                        </b></li>
                        <li align="left"><b>
                            <font size=2 color=silver>)</font>
                        </b></li>
                    </ol>
                </td>
            </tr>
            </tbody>
            </table>  

    + Libs
        - Visual Studio: setup _Additional Dependencies_
        - CMakeLists
            <table style="margin-bottom: 8px">
            <tbody>
            <tr style="border: none">
                <td style="background: black; border: none; width: 100vw; border-radius: 4px">
                    <ol style="margin:0">
                        <li align="left"><b>
                            <font size=2 color=steelblue>target_link_libraries</font><font size=2 color=silver>&ensp;(</font><font size=2 color=mediumseagreen>${PROJECT_NAME}</font>
                        </b></li>
                        <li align="left"><b>
                            <font size=2 color=skyblue>PRIVATE</font>
                        </b></li>
                        <li align="left"><b>
                            <font size=2 color=gainsboro>debug</font>
                        </b></li>
                        <li align="left"><b>
                            <font size=2 color=mediumseagreen>${CMAKE_SOURCE_DIR}</font><font size=2 color=silver>/rlottie/sdk/debug/rlottie.lib</font>
                        </b></li>
                        <li align="left"><b>
                            <font size=2 color=gainsboro>optimized</font>
                        </b></li>
                        <li align="left"><b>
                            <font size=2 color=mediumseagreen>${CMAKE_SOURCE_DIR}</font><font size=2 color=silver>/rlottie/sdk/release/rlottie.lib</font>
                        </b></li>
                        <li align="left"><b>
                            <font size=2 color=silver>)</font>
                        </b></li>
                    </ol>
                </td>
            </tr>
            </tbody>
            </table> 

    + Dlls
        - Visual Studio: copy them to _Output Directory_
        - CMakeLists
            <table style="margin-bottom: 8px">
            <tbody>
            <tr style="border: none">
                <td style="background: black; border: none; width: 100vw; border-radius: 4px">
                    <ol style="margin:0">
                        <li align="left"><b>
                            <font size=2 color=white>cmake_path(CONVERT</font>
                            <font size=2 color=peru>&ensp;"</font>
                            <font size=2 color=mediumseagreen>${CMAKE_SOURCE_DIR}</font>
                            <font size=2 color=white>/rlottie/sdk/$<CONFIG>/*.dll TO_NATIVE_PATH_LIST rlottie_dll_path)</font>
                        </b></li>
                            <li align="left"><b>
                            <font size=2 color=white>cmake_path(CONVERT</font><font size=2 color=peru>&ensp;"</font><font size=2 color=mediumseagreen>${CMAKE_RUNTIME_OUTPUT_DIRECTORY}</font><font size=2 color=peru>/"</font><font size=2 color=white>&ensp;TO_NATIVE_PATH_LIST runtime_output_ path)</font>
                        </b></li>
                        <li align="left"><b>
                            <font size=2 color=steelblue>add_custom_command</font><font size=2 color=white>(</font><font size=2 color=steelblue>TARGET</font><font size=2 color=mediumseagreen>${XXX_TARGET_NAME}</font><font size=2 color=white>&ensp;POST_BUILD</font>
                        </b></li>
                        <li align="left"><b>
                            <font size=2 color=steelblue>&emsp;&emsp;&emsp;&emsp;COMMAND</font><font size=2 color=white>&ensp;xcopy</font><font size=2 color=peru>&ensp;"</font><font size=2 color=mediumseagreen>${rlottie_dll_path}</font><font size=2 color=peru>"&ensp;"</font><font size=2 color=mediumseagreen>${runtime_output_path}</font><font size=2 color=peru>"</font><font size=2 color=white>&ensp;/y</font>
                        </b></li>
                        <li align="left"><b>
                            <font size=2 color=steelblue>&emsp;&emsp;&emsp;&emsp;WORKING_DIRECTORY</font><font size=2 color=mediumseagreen>${CMAKE_CURRENT_SOURCE_DIR}</font>
                        </b></li>
                        <li align="left"><b>
                            <font size=2 color=steelblue>&emsp;&emsp;&emsp;&emsp;COMMAND</font><font size=2 color=peru>&ensp;"Copying thirdparty files"</font><font size=2 color=steelblue>&ensp;VERBATIM</font>
                        </b></li>
                        <li align="left"><b>
                            <font size=2 color=white>&emsp;&emsp;&emsp;)</font>
                        </b></li>
                    </ol>
                </td>
            </tr>
            </tbody>
            </table>
