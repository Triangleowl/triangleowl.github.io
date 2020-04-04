---
layout: post
title: 第一个window程序
author: Triangleowl
categories: Windows
permalink: /:categories/:title
---
### 整体思路
编写window程序的步骤如下：
1. 定义一个 `WNDCLASS`,并对其成员赋值；
2. 调用 `RegisterClass` 注册定义的 `WNDCLASS`；
3. 调用 `CreateWindow` 创建一个 window；
4. 调用 `ShowWindow` 显示创建的 window
5. 接受键盘、鼠标数据  


完整代码见[源码]


### 定义一个 `WNDCLASS`
```C
typedef struct {
    UINT style;
    WNDPROC lpfnWndProc;
    int cbClsExtra;
    int cbWndExtra;
    HINSTANCE hInstance;
    HICON hIcon;
    HCURSOR hCursor;
    HBRUSH hbrBackground;
    LPCTSTR lpszMenuName;
    LPCTSTR lpszClassName;
} WNDCLASS, *PWNDCLASS;
```
根据 `MSDN` 介绍，不必对`WNDCLASS`的每个成员都定义，只需要对三个成员定义即可.

![WNDCLASS介绍](https://github.com/Triangleowl/picture/blob/master/first-window/WNDCLASS.png?raw=true "MSDN手册")


### 注册`WNDCLASS`
调用`RegisterClass`即可。


### 创建window
`CreateWindow`的参数比较多，无用的参数赋值为0或者NULL即可。
```C
HWND CreateWindow(          LPCTSTR lpClassName,
    LPCTSTR lpWindowName,
    DWORD dwStyle,
    int x,
    int y,
    int nWidth,
    int nHeight,
    HWND hWndParent,
    HMENU hMenu,
    HINSTANCE hInstance,
    LPVOID lpParam
);
```


### 显示 window
调用`ShowWindow`和`UpdateWindow`即可。

### 编写 `WindowProc`函数
为了能够交互，窗口应该能接受鼠标和键盘消息。按照要求，`WindowProc`函数的原型如下：

```C
LRESULT CALLBACK WindowProc(          HWND hwnd,
    UINT uMsg,
    WPARAM wParam,
    LPARAM lParam
);
```

具体的编程查询`MSDN`或者`MicroSoft`的[在线文档]




[源码]:https://github.com/Triangleowl/101/blob/master/first-window/window.cpp "源码链接"

[在线文档]:https://docs.microsoft.com/en-us/windows/win32/learnwin32/your-first-windows-program "第一个window程序"