上下文本地变量
==============

你可能已经意识到在一个请求的处理过程中, 你将有一些数据需要传递到多个函数中使用.
那么你会觉得使用全局变量存取要比层层参数传递便捷得多.
但是在Python的Web应用中使用全局变量是非线程安全的, 
这会造成不同的请求处理过程之间互相干扰数据.

因此, 在一个请求的处理过程中, 你应该使用上下文本地变量而不是全局变量来存
取通用的数据. 一个上下文本地变量能够全局性地定义或者导入, 但是它包含的
数据是对当前线程, 异步任务或者协程来说是特定的. 你不会意外获取或者覆盖
了另一个处理过程的数据.

当前Python内置存储每个上下文本地数据的方法是*contextvars*模块.
上下文本地变量为每个线程, 异步任务或者协程存储各自的数据.
它替代了较早的只适用于处理线程情况的*threading.local*.

Werkzeug提供了对Python内置类*contextvars.ContextVar*的封装, 使之更容易使用.


代理对象
========

*LocalProxy*允许像直接使用一个普通对象一样使用上下文本地变量,
不需要再调用*ContextVar.get()*并进行相应的检查处理.

如果上下文本地变量被设置了, 则*LocalProxy*可以查找到它, 并可以像这个对象
本身一样被使用. 如果没有被设置, 大部分操作会抛出*RuntimeError*异常.

    from contextvars import ContextVar
    from werkzeug.local import LocalProxy

    _request_var = ContextVar("request")
    request = LocalProxy(_request_var)

    from werkzeug.wrappers import Request

    @Request.application
    def app(r):
        _request_var.set(r)
        check_auth()
        ...

    from werkzeug.exceptions import Unauthorized

    def check_auth():
        if request.form["username"] != "admin":
            raise Unauthorized()

服务器端的处理者使用*request*会指向每一个特定请求的数据, 而不会在多个
请求之间发生数据混淆. 你可以像使用一个真实的*Request*对象那样使用*request*.

如果变量没有被设置, *bool(proxy)*将会返回*False*.
如果你想直接使用对象而不是代理, 你可以通过*LocalProxy._get_current_object*方法获取它.


栈和命名空间
============

类*contextvars.ContextVar*一次存储一个数据.
你会发现需要存储一个堆栈的元素或者一个命名空间下的多个属性数据.
这种情况下需要使用列表或者字典, 但是在上下文变量中使用它们需要更小心,
注意更多的细节. Werkzeug提供了*LocalStack*来封装一个列表, 以及*Local*
来封装一个字典.

这会带来不小的性能损失. 因为列表和字典是可变的, 类*LocalStack*和类
*Local*需要做更多的工作来确保数据不会被嵌套的上下文共享.
请尽可能设计你的程序使用*LocalProxy*封装上下文变量.


释放数据
========

之前的*Local*实现用的是内置数据结构, 在上下文结束时无法自动释放.
而要使用下面的方法来手动释放数据.

警告:

    在主流实现中, 上下文中的数据应该被Python自动化管理, 而不需要手动释放.
    这里保留手动控制的能力, 但是未来可能会被移除.

*release_local*
