# SLF4j
# logback
1. logback 由两个模块组成：logback-core 和 logback-class

# logback-core
## Appender
1. appender 这个单词的意思就是：输出源 or 输出目的地，在 logback 中 Appender 组件确实是这个意思，就是将用户的日志内容输出到某个目的地，比如：控制台、日志文件、电子邮件等。
2. Appender 接口有三个方法，其中，最主要的方法便是 `void doAppend(E event) throws LogbackException;`，另外两个方法可以先忽略。而 doAppend 方法的作用就是 "输出"，也可以说是 "写入"。
### 一级子类（重点关注 doAppend 方法逻辑）
#### AppenderBase
1. AppenderBase 是一个实现了 Appender 接口的抽象类，将 doAppend 方法委托给自己的抽象方法 `abstract protected void append(E eventObject);`，真正写入 eventObject 的操作交给了子类实现，这便是设计模式中的模板方法模式。
2. AppenderBase 在执行 append 方法前后，使用了布尔变量 guard 来保证同步。append 方法之前将 guard 置为 true，append 方法之后将 guard 置为 false。 若执行 doAppend 方法时，guard 为 true，则不执行 append 方法，直接返回。
3. 奇怪的是，这里的 guard 变量的定义是：`private boolean guard = false;`，也就是说它没有使用 volatile 关键字修饰，多线程环境下是不安全的，除非子类实现 append 方法时保证多线程安全！
#### UnsynchronizedAppenderBase
1. UnsynchronizedAppenderBase 也是一个实现了 Appdner 接口的抽象类，基本功能与 AppenderBase 类一致。区别在于 UnsynchronizedAppenderBase 中的 guard 变量是线程私有变量：`private ThreadLocal<Boolean> guard = new ThreadLocal<Boolean>();`，这样每个线程都可以独立控制自己的 guard 变量，但这对于 append 方法来说仍然无法保证多线程安全，多线程安全仍然需要由子类实现 append 方法时保证。
### 二级子类（重点关注 append 方法逻辑）
#### OutputStreamAppender
1. OutputStreamAppender 类继承自 UnsynchronizedAppenderBase 类，从类名可以看出它是将 Java IO 中最基本的 OutputStream 作为输出目的地。其 append 方法主要是委托给了自身另一个方法 `protected void subAppend(E event)`，为啥要再增加一个 subAppend 方法？主要是为了将实际的写入操作和 start、stop 这种生命周期相关逻辑区分开。
2. subAppend 方法主要逻辑是，先调用 Encoder 组件将 event 对象编码为字节数组，再将字节数组写入输出流 OutputStream。（毕竟，一个 event 对象实例是无法直接被写入到某个硬件设备的！）
3. 写入字节数组的方法如下：
![OutputStreamAppender#wirteBytes](https://logback-1259456425.cos.ap-beijing.myqcloud.com/20231108194416.png)
从代码中可以看出：在执行写入（ `outputStream.write(byteArray)`）操作前进行了加锁操作，保证了多线程安全。
### 三级子类
#### FileAppender
1. FileAppender 从类名可以看出它是将文件作为输出目的地。它增加了一个布尔变量 prudent，看注释是用来实现多进程同步写入操作的：
![FileAppender#setPrudent](https://logback-1259456425.cos.ap-beijing.myqcloud.com/20231109162523.png)
这个变量主要是在方法 writeOut 中使用，但我没有找到调用 writeOut 方法的地方，所以这段逻辑是无用的？暂时忽略它。
### 四级子类（重点关注 subAppend 方法逻辑）
#### RollingFileAppender
1. RollingFileAppender 从类名和继承关系可以看出它也是将文件作为输出目的地，但它有额外的功能：将日志文件按大小或者按日期滚动归档。这也是我们日常最常用的 Appender。
2. RollingFileAppender 为了实现自身功能，覆盖了父类 OutputStreamAppender 的 subAppend 方法，增加了日志文件滚动归档功能，真正的写入操作还是委托给了父类 OutputStreamAppender 的 subAppend 方法：
![RollingFileAppender#subAppend](https://logback-1259456425.cos.ap-beijing.myqcloud.com/20231109173652.png)
从上图中，也可以看出什么时候进行滚动归档是取决于 TriggeringPolicy 组件的。而从 rollover 方法的逻辑中也可以看出具体的滚动归档操作是由组件 RollingPolicy 组件实现的：
![RollingFileAppender#rollover](https://logback-1259456425.cos.ap-beijing.myqcloud.com/20231109175608.png)
![RollingFileAppender#attemptRollover](https://logback-1259456425.cos.ap-beijing.myqcloud.com/20231109175729.png)
这里我们不再继续深入分析 TriggeringPolicy 和 RollingPolicy 组件了，这两个组件在其他小节进行专项分析。

## TriggeringPolicy
## RollingPolicy





从这里可以看到一个日志框架中包含很多组件，每个组件包含一部分独立的逻辑，这实现了良好的逻辑解耦。同时，每个组件可以独立发展，做到了正交性设计。
