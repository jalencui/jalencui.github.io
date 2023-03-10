---
title: QCef-Ime-JSInject
author: Jalen Cui
comments: true
commonts: true
categories: Cef
date: 2023-03-08 22:47:56
tags:
---


# 预备知识
* Communications: between host and web
    + [__Preloading__](https://www.electronjs.org/docs/latest/tutorial/tutorial-preload): supported by Electron only
        > Preload scripts are injected __BEFORE__ a web page loads in the renderer, similar to a Chrome extension's content scripts. To add features to your renderer that require privileged access, you can define global objects through the contentBridge API.
        - __Guard current context__: Embedding google login or facebook login page, for example, we need to isolate their content (global setting, global variables etc.). Otherwise, we may encounter various error info, for example, jquery related error or something else. And `Preload` is the only way to taskle this situation.
    + [__Execute Javascript__](https://cefview.github.io/QCefView/zh/docs/reference/QCefView): supported by almost all framework
        > Executes javascript code in specified frame at any time.
        - __Inject JS Code__: In this way, we usually do something to adjust user interface, or fix some irregular behavior in our host. For example, constraint user to type chinese, japanese to __INPUT__, and this is the only way to fix this in OSR mode.
* Html:  `<input/>` event related
    + Enter:
        - `keydown` -> `keypress` -> `change` -> `keyup`
    + Alphabet: 
        -  `keydown` -> `keypress` -> `change` -> `keyup`
    + Ime:
        - `compositionstart` -> `compositionupdate` -> `input` -> ... -> `input` -> `compositionend`   
    + Ime: filter in `input` event
        - `compositionstart` -> `compositionupdate` -> `input` -> ... -> `input` -> `compositionend`
* Cef: develop tools
    + Mode:
        - Popup Window: this is the only effective way if __OSR__ enabled
        - Child Window: like all other browsers' default behavior
        - Windowless: off-screen rendering
    + Prerequisite:
        - OnPreKeyEvent:
            + CefViewBrowserClient: from CefViewBrowserClient_KeyboardHandler
            + CCefClientDelegate: from CCefCientDelegate_KeyboardHandler
# 简介  

桌面端嵌入类 `webview` 的产品十分常见(开发周期极短, 开箱即用). 从某种程度上来讲，桌面软件扮演了浏览器的角色。对于常规 web 页面, 浏览器的行为在这种 Hybrid 模式的桌面软件中必定发生不符合预期的表现:  

* 登录注册:
    + Fackbook: 需要保证相对纯净的 jquery 上下文, 才能正常走完登录的流程
    + Google: 需要伪造合法的 User Agent, 才能通过页面的验证
* 输入法: only __IME__ introduced
    + 联想相关: 联想框弹出位置问题
    + IME: 中文输入法问题
        - IE 浏览器: 可通过 属性 `ime-mode` 属性屏蔽 `composition` 相关动作
        - Chrone 系列： 不支持 `ime-mode`
            + 屏蔽系统输入法 
            + JS 注入, 以改变 __HTML__ 默认行为  

本文主要介绍 web 页面内 `input` 标签屏蔽中文输入的相关方法. 考虑到 Cef 支持两种渲染模式(__OSR__, __NON-OSR__), 这里介绍两种不同的方式处理中文输入问题：屏蔽系统输入法, JS 注入改变默认行为.
# 准备阶段
* 双端通讯: 主进程与渲染进程间通信的方式，无外乎通过全局变量(window.XXXX). Electron 开发中就是直接的对 `window` 进行赋值. QCef 则提供了 `setBridgeObjectName` 接口. 二者本质上没有区别.
    + 进程通信:
```cpp
    QCefConfig::setBridgeObjectName("CallBridge");
```
    + HTML 回调:
```cpp 
    R"(
        window.addEventListener('load', ()=>{
            setTimeout(()=>{
                var inputs = document.getElementsByClassName('XXXXXX'); // or getElementById
                if(inputs.length > 0){
                    // method 1
                    inputs[0].addEventListener("XXXXX", (event)=>{
                        // 改变 HTML 默认行为, 达到屏蔽中文的目的
                    });
                    // method 2
                    inputs[0].addEventListener("YYYYY", (event)=>{
                        // 回调至主进程, 以屏蔽输入法
                    })
                }
            }, 200); // delay 200ms in case too early to find no <input/>
        });
    )"
```
* 屏蔽系统输入法: 这是相对比较合理的处理方式. 此处贴出相关逻辑处理 
```cpp
void XXX::setImeEnabled(bool enabled)
{
    HWND wnd = getWindowHandle(); // user defined
    if(enabled)
    {
        m_himc = ::ImmGetContext(wnd);
        if(m_himc)
        {
            ImmAssociateContext(wnd, nullptr);
            ImmReleaseContext(wnd, m_himc);
        }
    }else{
        if(m_himc){
            ImmDestroyContext(m_himc);
            m_himc = nullptr;
        }
        m_himc = ImmCreateContext();
        if(m_himc)
        {
            ImmAssociateContext(wnd, m_himc);
            ImmReleaseContext(wnd, m_himc);
        }
    }
}
```
* JS 注入: OSR 模式下, 业务层无法拿到当前输入法对应的窗口句柄(HWND). 为了保证 Web 页面多端行为一致性, 要求相应开发修改前端逻辑是不合适的.

# 代码实现
两种方式可以屏蔽用户输入中文密码的问题:  
* 屏蔽中文输入法:
通过注入代码的形式, 绑定 `<input/>` 焦点事件. 当获取焦点时, 通过捕获 `focus` 事件的方式间接通知 host 屏蔽当前窗口对应的输入法，即: 调用 `setImeEnabled`. 当失去焦点时, 则通知 host 恢复当前窗口的输入法状态.
```cpp
void YYY::injectToLogin(){
    QString jsCode = R"(
        window.addEventListener('load', ()=>{
            setTimeout(()=>{
                var inputs = document.getElementsByClassName('XXXXXX'); // or getElementById
                if(inputs.length > 0){ // method 2
                    inputs[0].addEventListener("focus", (event)=>{
                        window.CallBridge.invokeMethod("inputFocus", '{"focus": true}');
                    })
                    inputs[0].addEventListener("focusout", (event)=>{
                        window.CallBridge.invokeMethod("inputFocus", '{"focus": false}');
                    })
                }
            }, 200); // delay 200ms in case too early to find no <input/>
        });
    )"
}
```
* Js 注入: 
当采用这种方式过滤中文字符时，会造成用户无法通过 ENTER 键触发登录的问题. 由 __预备知识__ 可知, `keydown` 是最早触发的事件. 通过捕获 `keydown` 事件, 我们可以在用户输入完密码并按下 __ENTER__ 键的第一时刻模拟登录按钮的单击事件.
```cpp
void YYY::injectToLogin(){
    QString jsCode = R"(
        window.addEventListener('load', ()=>{
            setTimeout(()=>{
                var inputs = document.getElementsByClassName('XXXXXX'); // or getElementById
                if(inputs.length > 0){ // method 1
                    inputs[0].addEventListener("keydown", (event)=>{ // fix: simulate click event trigger by ENTER
                        var key = event.which;
                        if(key === 13)
                        {
                            inputs[0].blur();
                            var loginBtn = document.getElementById('YYYYYY')
                            if(loginBtn){
                                const event = new MouseEvent('click', {
                                    view: window,
                                    bubbles: true,
                                    cancelable: true
                                });
                                loginBtn.dispatchEvent(event);
                            }
                        }
                    })
                    inputs[0].addEventListener("input", (event)=>{
                        event.target.value = event.target.value.replace(/[\u4e00-\u9fa5]/g, '');
                    })
                }
            }, 200); // delay 200ms in case too early to find no <input/>
        });
    )"
}
```
# To be continue ...
