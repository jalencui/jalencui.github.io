---
title: QCef-Ime-JSInject
author: Jalen Cui
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
        - Popup Window: this is the only effective way if OSR enabled
        - Child Window: like all other browsers' default behavior
        - Windowless: off-screen rendering
    + Prerequisite:
        - OnPreKeyEvent:
            + CefViewBrowserClient: from CefViewBrowserClient_KeyboardHandler
            + CCefClientDelegate: from CCefCientDelegate_KeyboardHandler
# 简介


# 准备阶段


# 代码实现


# To be continue ...