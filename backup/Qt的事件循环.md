## 什么是事件循环
假设有3个线程A、B和C：
线程A，一个死循环，等待其他线程的事件通知，大部分时间阻塞着；
线程B、C，需要执行功能时，只需要把事件通知发给线程A，线程A就会解除阻塞，执行B、C想要执行的功能；
这样，不管线程B、C需要执行的功能多么耗时，都可以立即响应，因为它们只要把事件ID发给线程A就结束了；
所谓事件循环，就是有一个主线程，保存了一个事件队列（其他线程需要做的事情），循环遍历队列执行对应的功能，在队列为空时候，阻塞等待。

## Qt中的使用
每一个Qt程序，都会有这样的代码
```
int main(int argc, char *argv[])
{
	QApplication a(argc, argv);
	...
	return a.exec()
}
```
很明显，QApplication::exec()就会让程序进入事件循环，等待各种事件响应（鼠标、键盘等等），进行处理。
官方文档提到：
- 1、exec()会进入主事件循环，退出需要使用exit()
- 2、在exec()执行之前，不能发生任何用户交互（因为没有事件循环，也就没线程干活）


## Qt源码跟踪
接下来，根据下源码，看看执行exec()后，程序如何进入到事件循环。
- 1、QApplication::exec()
- 2、QGuiApplication::exec()
- 3、QCoreApplication::exec()
- 4、QEventLoop::exec(ProcessEventsFlags flags = AllEvents)
  程序在此处进入死循环：
  ```
  while (!d->exit.loadAcquire())
        processEvents(flags | WaitForMoreEvents | EventLoopExec);
  ```
- 5、QEventLoop::processEvents(ProcessEventFlags flags = AllEvents)
- 6、Qt是跨平台的，基于不同的系统有不同的事件分发器
  - QEventDispatcherUNIX::processEvents(QEventLoop::ProcessEventsFlags flags)
  - QEventDispatcherGlib::processEvents(QEventLoop::ProcessEventsFlags flags)
  - QEventDispatcherWin32::processEvents(QEventLoop::ProcessEventsFlags flags)
  - QEventDispatcherWinRT::processEvents(QEventLoop::ProcessEventsFlags flags)
  - QEventDispatcherCoreFoundation::processEvents(QEventLoop::ProcessEventsFlags flags)
  - QCocoaEventDispatcher::processEvents(QEventLoop::ProcessEventsFlags flags)
- 7、跟踪linux平台下的源码，即QEventDispatcherUNIX::processEvents()的处理:
  其中片段如下，继续跟踪qt_safe_poll可以发现，在此处会进入阻塞（linux的poll系统调用），通过IO复用响应不同的读写事件。
  ```
  switch (qt_safe_poll(d->pollfds.data(), d->pollfds.size(), tm)) {
      case -1:
          perror("qt_safe_poll");
          break;
      case 0:
          break;
      default:
          nevents += d->threadPipe.check(d->pollfds.takeLast());
          if (include_notifiers)
              nevents += d->activateSocketNotifiers();
          break;
      }
  ```

创建事件分发器（找了好久，记录一下）
- 1、QEventLoop::EventLoop(QObject *parent)
- 2、QThreadData::ensureEventDispatcher()
- 3、QThreadData::createEventDispatcher()
- 4、QThreadPrivate::createEventDispatcher()
qthread_unix.cpp中的实现：
```
QAbstractEventDispatcher *QThreadPrivate::createEventDispatcher(QThreadData *data)
{
    Q_UNUSED(data);
#if defined(Q_OS_DARWIN)
    bool ok = false;
    int value = qEnvironmentVariableIntValue("QT_EVENT_DISPATCHER_CORE_FOUNDATION", &ok);
    if (ok && value > 0)
        return new QEventDispatcherCoreFoundation;
    else
        return new QEventDispatcherUNIX;
#elif !defined(QT_NO_GLIB)
    const bool isQtMainThread = data->thread.loadAcquire() == QCoreApplicationPrivate::mainThread();
    if (qEnvironmentVariableIsEmpty("QT_NO_GLIB")
        && (isQtMainThread || qEnvironmentVariableIsEmpty("QT_NO_THREADED_GLIB"))
        && QEventDispatcherGlib::versionSupported())
        return new QEventDispatcherGlib;
    else
        return new QEventDispatcherUNIX;
#else
    return new QEventDispatcherUNIX;
#endif
}

```
qthread_win.cpp中的实现：
```
QAbstractEventDispatcher *QThreadPrivate::createEventDispatcher(QThreadData *data)
{
    Q_UNUSED(data);
#ifndef Q_OS_WINRT
    return new QEventDispatcherWin32;
#else
    return new QEventDispatcherWinRT;
#endif
}
```