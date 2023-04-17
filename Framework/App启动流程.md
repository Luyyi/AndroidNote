# App启动流程

### 1. 冷启动分析

#### （1）启动APP进程

① 点击桌面App图标时，Launcher进程调用startActivity()，通过Binder通信，通知system_server。

② system_server收到消息，由AMS通过socket通信告诉Zygote进程生成一个子进程，这个子进程就是APP进程。

#### （2）启动APP主线程

① APP进程启动后，实例化ActivityThread，执行main函数。

② 创建ApplicationThread，Looper，Handler，开启主线程的消息循环。

#### （3）初始化Application

① ActivityThread的main函数，通过Binder向system_server的AMS发送attachApplication请求。

② system_server收到请求，AMS通过bindApplication，给主线程的Handler发送消息。

③ 主线程收到消息后，调用handleBindApplication，初始化Application；

#### （4）初始化Activity

① system_server在bindApplication后，会通过scheduleLaunchActivity向主线程发送消息。

② 主线程收到消息后，调用handleLaunchActivity、performLaunchActivity，通过反射创建Activity并初始化。

#### （5）UI布局绘制

① 主线程Handler初始化Activity，创建PhoneWindow,初始化DecorView，设置WindowManager来管理window，然后通知Activity已创建。

② Activity调用onCreate()，将布局添加到DecorView的ContentView中。

</br>

### 2. 一些说明

##### ① Zygote进程

Android系统启动时生成的进程，是Android系统的首个Java进程。

Zygote进程负责孵化system_server进程和App进程。

##### ② system_server进程

是Zygote进程孵化的第一个进程。

负责启动、管理整个Java Framework，系统里重要的服务都在这里开启。

##### ③ APP进程

由Zygote进程孵化。

Launcher进程是Zygote在应用层孵化的第一个进程，即桌面APP。

##### ④ AMS

ActivityManagerService，是system_server启动的系统服务，用于管理四大组件和应用进程的启动、切换、调度等。

##### ⑤ ApplicationThread

是ActivityThread的内部类，实现了Binder，AMS通过Binder经由ApplicationThread对APP进行控制。
