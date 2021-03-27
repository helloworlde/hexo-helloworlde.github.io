---
title: Go 调用 Windows 的系统 API，检测是否锁屏
date: 2019-09-15 21:56:13
tags:
    - Go
categories: 
    - Go
---

# Go 调用 Windows 的系统 API，检测是否锁屏

> 因为应用需要根据当前电脑是否处于活跃状态来执行不同的动作，所以需要获取电脑当前活跃的窗口判断是否处于锁屏
> 可以通过调用Windows 的库来执行相应的API

```go
import (
    "log"
    "syscall"
)

func main() {
    const successCallMessage = "The operation completed successfully."
    // 加载类库
    user32 = syscall.NewLazyDLL("user32.dll")
    // 创建新的调用进程
    getForegroundWindow = user32.NewProc("GetForegroundWindow")
    // 调用相应的函数
    activeWindowId, _, err := getForegroundWindow.Call()

    if err != nil && err.Error() != successCallMessage {
        log.Println(err)
    }
    log.Println("activeWindowId:", activeWindowId)
}
```

当调用成功后时，会返回三个结果，第一个是当前活跃的窗口 ID，当 ID 为 0 时，就说明处于锁屏状态；第三个参数是操作信息，如果成功内容就是`The operation completed successfully.`

这个函数没有入参，所以直接通过`Call()`调用，函数的详细信息可以参考微软提供的API [GetForegroundWindow function](https://docs.microsoft.com/zh-cn/windows/win32/api/winuser/nf-winuser-getforegroundwindow)

其他的函数调用也是一样，不同的是传入的参数和返回的结果，但调用过程是一样的

#### 参考文章

- [Programming reference for Windows API](https://docs.microsoft.com/zh-cn/windows/win32/api/)
- [GetForegroundWindow function](https://docs.microsoft.com/zh-cn/windows/win32/api/winuser/nf-winuser-getforegroundwindow)
- [WindowsDLLs](https://github.com/golang/go/wiki/WindowsDLLs)
