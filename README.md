# iOS-runloop
        runLoop，正如其名，表示一直运行着的循环。
        一般来说，一个线程只能执行一个任务，执行完就会推出，如果我们需要一种
    机制，让线程能随时处理时间但并不退出，而runLoop就是这样一个机制；而这种
    机制的关键在于：如何管理消息/消息，如何让线程在没有处理消息的时候休眠以
    避免资源占用，在有消息的时刻立刻被唤醒。
        所以，runLoop实际上就是一个对象，这个对象管理了其需要处理的时间和消息
    ，并提供了一个入口函数来执行上面的逻辑。线程执行了这个函数之后，就会一直
    处于“接收消息-等待-处理”的循环中，知道这个循环结束，函数返回。
        ios提供了两个这样的对象：NSRunLoop和CFRunLoopRef。
    
##一、线程与runLoop
###1.线程任务的类型
        ① 直线线程：该线程执行的任务是一条直线；
        ② 圆形线程：该线程是一个圆，不断循环，知道通过某种方式截止，ios中，圆
    形线程就是通过runLoop实现的。
##2.线程与runLoop的关系
        ① runLoop和线程紧密相连，可以说，runLoop是为了线程而生的，没有线程，
    就没有runLoop存在的必要；
        ② 每个线程都有其对应的runLoop对象；
        ③ 主线程的runLoop是默认启动的，而其他线程的runLoop是默认没有启动的；
##二、RunLoop输入事件来源
        runLoop接收的输入事件来自两种来源：输入源和定时源；
###1.输入源
        传递异步事件，通常消息来自其他线程或程序。输入源传递异步消息给相应的处
    理程序，并调用runUntilDate方法来退出；
        当你创建输入源，需要将其分配给runLoop的一个或多个模式，模式只会在特定
    事件影响监听的源。
        以下是输入源的类型：
        ① 基于端口的输入源：基于端口的输入源是有内核自动发送；
            cocoa和Core Foundation内置支持使用端口相关的对象和函数来创建基于端
        口的源。
            例如：在Core Foundation中，使用端口相关的函数来创建端口和runLoop源；
        ② 自定义输入源：自定义源需要人工从其他线程发送。
            Core Foundation中可以使用CFRunLoopSourceRef等来创建源，也可以使用
        回调函数来配置源。Core Foundation会在配置源的不同地方调用回调函数，处理
        输入事件，在源从runLoop移除的时候清理它；
        ③ Cocoa上的selector源
###2.定时源
        定时源在预设的时间点同步传递消息，这些消息都会在特定事件或者重复的时间
    间隔，定时源则传递消息个处理线程，不会立即退出runLoop。
        定时器并不是实时机制，定时器和你的runLoop的特定模式相关，如果定时器所在
    的模式当前未被runLoop监视，那么定时器将不会开始，知道runLoop运行在响应的模式
    下。
###3.runLoop观察者
        源是在合适的同步或异步事件发生时触发，而runLoop观察者则是在runLoop本身
    运行的特定时候触发，你可以使用runLoop观察者为处理某一特定事件或是进入休眠的
    程序做准备。可以将runLoop观察者和以下事件关联：
        ① runLoop入口
        ② runLoop何时处理一个定时器；
        ③ runLoop何时处理一个输入源；
        ④ runLoop何时进入睡眠状态；
        ⑤ runLoop何时被唤醒，但在唤醒之前要处理的事件；
        ⑥ runLoop终止；
        在创建的时候，也可以指定runLoop观察者可以只用一次或者循环使用，若只用一
    次，那么它在启动后，会把自己从runLoop中移除，而循环的观察者不会。
###4.runLoop事件队列
        每次运行runLoop，线程的runLoop会自动处理之前未处理的消息，并通知相关的
    观察者，具体的顺序如下：
        ① 通知观察者runLoop已经启动；
        ② 通知观察者任何即将要开始的定时器；
        ③ 通知观察着任何即将启动的非基于端口的源；
        ④ 启动任何准备好的非基于端口的源；
        ⑤ 如果基于端口的源准备好并处于等待状态，立即启动，并进入步骤⑨；
        ⑥ 通着观察者线程进入休眠；
        ⑦ 将线程至于休眠知道任意下面的事件发生：
            A.某一时间到达基于端口的源；
            B.定时器启动；
            C.runLoop设置的时间已经超过；
            D.runLoop被显示唤醒；
        ⑧ 通知观察者线程将被唤醒；
        ⑨ 处理未处理的事件
            A.如果用户定义的定时器启动，处理定时器并重启runLoop，进入步骤2
            B.如果输入源启动，传递响应的消息；
            C.如果runLoop被显示唤醒，而且时间还没超过，重启runLoop，进入步骤2
        ⑩ 通知观察者runLoop结束。
        
        ps：
            ① 如果事件到达，消息会被传递给响应的处理程序来处理，runLoop处理完
        当次事件后，runLoop就会推出，而不管之前预定的时间到了没有。
            ② 可以重新启动runLoop来等待下一事件；
            ③ 如果线程中有需要处理的源，但是响应的事件没有到来的时候，线程就会
        休眠等待相应事件的发生。
###5.runLoop的使用
        仅当在为你的程序创建辅助线程的时候，你才需要显示运行一个runLoop
        对于辅助线程，你需要判断一个runLoop是否是必须的。如果是必须的，那么你要
    自己配置并启动它，你不需要再任何情况下都去启动一个线程的runLoop。runLoop在你
    要和线程有更多的交互时才需要，比如以下情况：
        ① 使用端口或者自定义输入源来和其他线程通信；
        ② 使用线程的定时器；
        ③ Cocoa中使用任何performSelector的方法；
        ④ 使线程周期性工作。
