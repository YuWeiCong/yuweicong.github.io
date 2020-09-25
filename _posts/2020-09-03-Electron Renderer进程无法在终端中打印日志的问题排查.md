---
layout:     post
title:      Electron Renderer进程无法在终端中打印日志的问题排查
date:       2020-09-03
author:     Hope
header-img: img/google-chrome-logo.jpg
catalog: true
tags:
    - Electron
    - Chromium
    - LOG
---
# 结论
1. 要打印Browser 进程日志时，需添加 `--enable-logging` 命令行参数
2. 要打印Renderer 进程日志时，需添加 `--enable-logging` 和 `--enable-sandbox` （或者在electron app中调用[app.enableSandbox()](https://github.com/electron/electron/blob/master/docs/api/sandbox-option.md)方法）


# 背景
最近有同事反馈，在electron中调试官方的[electron-quick-start](https://www.electronjs.org/docs/tutorial/first-app#trying-this-example)时，看不到Renderer进程进程的日志。这么好的问题，怎能放过呢？下面记录排查过程。

# 复现现象
在[renderer_main.cc](https://source.chromium.org/chromium/chromium/src/+/master:content/renderer/renderer_main.cc)中RendererMain()函数中加入下面代码：
```
base::RepeatingTimer* timer = new base::RepeatingTimer();
timer->Start(
    FROM_HERE, base::TimeDelta::FromSeconds(1), base::BindRepeating([]() {
      LOG(WARNING) << "RendererMain test log";
    }));
```
然后重新编译electron，并用如下命令运行electron-quick-start。
```
out/default/electron ~/electron-quick-start/ --enable-logging
```
果然，Renderer 进程没有任何日志输出。为啥子呢？

# 排查过程
## 排查LOG相关设置
用 GDB 调试 Renderer进程的LOG代码，发现LOG会把日志打印到stderr中的。具体代码如下：[logging.cc](https://source.chromium.org/chromium/chromium/src/+/master:base/logging.cc;l=794?q=logging.cc&ss=chromium%2Fchromium%2Fsrc)
```
if (ShouldLogToStderr(severity_)) {
    ignore_result(fwrite(str_newline.data(), str_newline.size(), 1, stderr));
    fflush(stderr);
}
```
都调用了fflush(stderr)，为啥还没有输出了？为了验证最终fwrite是否真正有写入数据，便使用了strace来进行跟踪相关的内核调用。命令如下：
```
# xxx是 Renderer 进程的 pid
strace -e trace=write -s 200 -f -p xxx
``` 
发现，是有相关的内核调用的，而且也能看到日志内容（似乎也是一种解决办法）。

## 排查 stderr 无法显示日志
stdin、stdout、stderr本质上也是一个fd。<br>
![image](/img/2020-09-03-standard-stream.png)<br>
如果 stderr 被重定向了文件或者其他地方，那就不会输出到屏幕中了。使用如下linux 命令查看 stderr 的状态。
```
# xxx是 Renderer 进程的 pid
ls -al /proc/xxx/fd
# 下面是查找结果的一部分
lr-x------ 1 hope hope 64 Sep  3 11:54 0 -> /dev/null
lrwx------ 1 hope hope 64 Sep  3 11:54 1 -> /dev/pts/11
l-wx------ 1 hope hope 64 Sep  3 11:54 2 -> /dev/null
```
果然，stderr被重定向到了/dev/null。但是stdout还是能正常输出到屏幕的。
故把上面的`LOG(WARNING)`改为 `std::cout`，验证是否能显示日志。
```
LOG(WARNING) << "RendererMain test log";
# 改为
std::cout << "RendererMain test log" <<std::endl;
```
一切如预期，日志能打印出来了。那在什么地方，stderr或者STDERR_FILENO被重定向到/dev/null的呢？唉，这个问题一直没有能答案。

## electron 自带的app
偶然间发现，通过如下命令启动的electron 自带的app，是能在屏幕上显示Renderer进程的日志。
```
out/default/electron --enable-logging
```
通过与启动 electron-quick-start的命令行参数对比，发现命令行中多了`--enable-sandbox`参数，便加上该参数尝试。
```
out/default/electron ~/electron-quick-start/ --enable-logging --enable-sandbox
```
呃，Renderer进程日志能正常显示出来了。但为啥起作用了呢？一直没有找到原因。

# 其他解决办法：
1. 在[electron_main.delegate.cc](https://github.com/electron/electron/blob/master/shell/app/electron_main_delegate.cc)初始化LOG时，设置日志打在debug.log中，通过tail命令实时查看。代码如下：
```
logging::LoggingSettings settings;
settings.logging_dest = logging::LOG_TO_ALL;
base::FilePath log_filename;
base::PathService::Get(base::DIR_EXE, &log_filename);
log_filename = log_filename.AppendASCII("debug.log");
settings.log_file_path = log_filename.value().c_str();
settings.lock_log = logging::LOCK_LOG_FILE;
settings.delete_old = logging::DELETE_OLD_LOG_FILE;
logging::InitLogging(settings);
```
2. 通过SetLogMessageHandler来hook日志的处理，然后使用std::cout把日志打印出来。不过这样会缺少关键信息，比如日志中的时间戳、进程号等

# 参考文档
- [标准流-维基百科](https://zh.wikipedia.org/wiki/%E6%A8%99%E6%BA%96%E4%B8%B2%E6%B5%81#%E6%A8%99%E6%BA%96%E8%BC%B8%E5%85%A5_(stdin)ele)