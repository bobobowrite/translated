## 编写locust文件

现在让我们看一个更接近真实测试的例子:

```
    import time
    from locust import HttpUser, task, between

    class QuickstartUser(HttpUser):
        wait_time = between(1, 5)

        @task
        def hello_world(self):
            self.client.get("/hello")
            self.client.get("/world")

        @task(3)
        def view_items(self):
            for item_id in range(10):
                self.client.get(f"/item?id={item_id}", name="/item")
                time.sleep(1)

        def on_start(self):
            self.client.post("/login", json={"username":"foo", "password":"bar"})
```

我们分解来看一下这个例子

```
    import time
    from locust import HttpUser, task, between
```

一个locust文件是一个普通的Python脚本文件, 它可以从其它文件或者包中导入代码.

```
    class QuickstartUser(HttpUser):
```

这里我们定义一个类来模拟被测系统的用户. 这个类继承自*HttpUser <locust.HttpUser>*
, 而获得一个*client*属性. *client*是*HttpSession <locust.clients.HttpSession>*的实例, 
能够对我们要测试的目标系统发送HTTP请求.
当测试启动的时候, locust会为模拟的每一个用户产生一个实例, 每一个用户将使用自己的协程执行.

一个有效的locust文件至少需要具有一个继承自*User <locust.User>*的类.

```
    wait_time = between(1, 5)
```

我们定义了一个*wait_time*属性, 可以让被模拟的用户每个task(见下文)执行
后等待1到5秒钟.

```
    @task
    def hello_world(self):
        ...
```

被装饰器*@task*修饰的方法是locust文件的核心内容.
对每个运行的用户, locust创建一个协程去调用这些方法.

```
    @task
    def hello_world(self):
        self.client.get("/hello")
        self.client.get("/world")

    @task(3)
    def view_items(self):
    ...
```

这里我们通过装饰器*@task*声明了两个任务方法, 其中一个给了更高的权重(3) .
当*QuickstartUser*运行时, 会择取一个任务——在本例中要么是*hello_world*, 要么是*view_items*——来执行.
任务时随机选取的, 但是它们有不同的权重. 以上的设置会让locust以相对选取
*hello_world*3倍的可能性选取*view_items*执行.
当一个任务执行完毕后, User会休眠等待一段时间(本例中是1到5秒钟). 等待时
间结束后, User会选取一个新的任务执行, 然后这样不断循环.

注意只有被装饰器*@task*修饰的方法才会被择取执行, 所以你可以任意定义其
它内部辅助方法.

```
    self.client.get("/hello")
```

*self.client*属性产生的HTTP请求能够被locust记录.
要了解如何生成其它类型请求, 验证响应等等, 见
[Using the HTTP Client](#client-attribute-httpsession)

注意:

    HttpUser不是一个真正的浏览器, 因此它不会解析HTML响应来加载资源或者渲染页面.
    但是它可以跟踪cookie.

```
    @task(3)
    def view_items(self):
        for item_id in range(10):
            self.client.get(f"/item?id={item_id}", name="/item")
            time.sleep(1)
```

在*view_items*任务中, 我们使用一个查询参数发出了10个不同的URL.
为了防止locust分成10种分别的统计——因为统计是通过不同的URL分组的——
我们使用一个*name*参数来将这些请求统计在一个名为*/item*的条目下.

```
    def on_start(self):
        self.client.post("/login", json={"username":"foo", "password":"bar"})
```

另外我们还声明了一个*on_start*方法. 
每一个模拟的用户启动时会调用名为*on_start*的方法.

## User类

一个User类代表一类用户(或者一群成群飞行的蝗虫).
Locust将为每一个模拟的用户产生一个该User类的实例.
User类中定义了一些通用属性.

### wait_time属性 

User类的*wait_time <locust.User.wait_time>*方法非常简单易用地为每个任
务的执行引入了暂停. 如果这个属性没有被指定, 则一个任务结束后下一个任务
会立即执行.

* *constant <locust.wait_time.constant>* 用以设置固定的延时

* *between <locust.wait_time.between>* 用以设置在最小和最大值之间随机选择一个时长

例如, 让每个用户执行每个任务都等待0.5到10秒钟之间的一个随机时间:

```
    from locust import User, task, between

    class MyUser(User):
        @task
        def my_task(self):
            print("executing my_task")

        wait_time = between(0.5, 10)
```

* *constant_throughput <locust.wait_time.constant_throughput>*
用以设置一个适当的等待时间来确保任务每秒钟执行(最多)X次

* *constant_pacing <locust.wait_time.constant_pacing>*
用以设置一个适当的等待时间来确保任务每X秒(最多)执行一次(这是和*constant_throughput*数学上相反的).

注意:

    例如, 如果你想让Locust在高峰负载时每秒钟迭代运行500任务,
    你可以使用wait_time = constant_throughput(0.1)和5000数量用户.
    等待时长只能限制吞吐量, 而不是运行新的用户来达到这个目标. 所以,
    在我们的示例中, 如果任务迭代时间超过10秒钟, 吞吐量将会少于500.

    等待时长在任务执行"后"起作用, 所以如果你有较高的用户产生速率或者
    负载高峰, 你可能在负载爬升过程中停止迈向你的目标.

    等待时长作用于"任务"而不是请求. 例如, 如果你设"wait_time = constant_throughput(2),
    并且在任务中产生两个请求, 你的请求速率(RPS)将会达到每个用户4个请求.

你也可以在你的类中直接定义自己的wait_time方法.
例如, 下面的User类将会暂停1秒, 然后2秒, 然后3秒, 如此这样...

```
    class MyUser(User):
        last_wait_time = 0

        def wait_time(self):
            self.last_wait_time += 1
            return self.last_wait_time

        ...
```

## weight和fixed_count属性

如果在文件中不只一个用户类, 并且在命令行中也没有明确指定哪一个用户类的
话, Locust将为每一个用户类产生同样数量的用户.
你也可以在命令行中指定使用同一个文件中具体哪些用户类.

```
    $ locust -f locust_file.py WebUser MobileUser
```

如果你希望对某个特点类型模拟更多用户, 你可以为这些用户类设置一个权重属性.
比如说, Web用户是移动用户的3倍:

```
    class WebUser(User):
        weight = 3
        ...

    class MobileUser(User):
        weight = 1
        ...
```

同样你可以设置*fixed_count <locust.User.fixed_count>*属性.
在这个例子中, 权重属性将被忽略, 而产生确切数量的用户.
这些用户被首先产生. 在下面的例子中, 只有一个AdminUser实例被生成,
它更精确的控制产生独立于总用户量的请求数量来实现特定的工作. 

```
    class AdminUser(User):
        wait_time = constant(600)
        fixed_count = 1
        
        @task
        def restart_app(self):
            ...

    class WebUser(User):
        ...
```

## host属性

host属性是一个URL前缀(比如"http://google.com").
通常是在locust启动时候, 用*--host*选项, 指定在locust的Web UI或者命令行模式下使用.

如果在用户类中声明了一个host属性, 它将在命令行(*--host*)或者web请求中没有指定的
情况下起作用.

## tasks属性

一个用户类可以将方法注解上*@task <locust.task>*变成任务, 也可以使用
*tasks*属性来指定, 更多细节查看下面的"tasks-attribute"

## environment属性

正在运行的用户中都有一个*environment <locust.env.Environment>*属性, 用以和environment交互, 
也可以使用environment持有的*runner <locust.runners.Runner>*属性, 从任务方法内部停止runner等等:

```
    self.environment.runner.quit()
```

如果是单机运行模式, 这将停止整个运行. 如果是运行在多节点的一个, 这只会
停止当前这个节点.


## on_start和on_stop方法

Users(和*TaskSets <tasksets>*)能够声明一个*on_start <locust.User.on_start>*方法和*on_stop <locust.User.on_stop>*方法.
一个用户在启动运行时会调用它的*on_start <locust.User.on_start>*方法,
在结束运行时会调用*on_stop <locust.User.on_stop>*方法.
当一个模拟用户开始执行它的TaskSet时, TaskSet的*on_start <locust.TaskSet.on_start>*方法被调用;
当用户停止执行TaskSet或者方法*interrupt() <locust.TaskSet.interrupt>*
被调用时或者用户被杀死时, TaskSet的*on_stop <locust.TaskSet.on_stop>*方法被调用.

## Tasks

当一个负载测试启动时, 对应每一个被模拟用户会创建一个User类的实例. 它们
运行在自己的协程里.
这些用户运行时, 它们选取任务执行, 休眠一会, 然后再选取一个新的任务执行,
如此这样循环.

任务都是普通的python可调用对象. 如果我们负载测试一个拍卖网站, 这些任务
可以做一些像"加载起始页", "搜索一些商品", "发起一个拍卖"等事情.

## @task装饰器

给一个用户添加任务的最简单方式就是使用*task <locust.task>*装饰器.

```
    from locust import User, task, constant

    class MyUser(User):
        wait_time = constant(1)

        @task
        def my_task(self):
            print("User instance (%r) executing my_task" % self)
```

*@task*有一个权重参数来控制任务的执行比率.
在下面的例子中, task2有两倍于task1被选择的可能性.

```
    from locust import User, task, between

    class MyUser(User):
        wait_time = between(5, 15)

        @task(3)
        def task1(self):
            pass

        @task(6)
        def task2(self):
            pass

```

## tasks attribute

为一个用户定义任务的另一种方式是设置*tasks <locust.User.tasks>*属性.
*tasks*属性可以是任务的列表或者字典, 其中的任务是python可调用对象或者*TaskSet <tasksets>*类.
任务是普通的python函数, 它接收一个单一的参数, 为运行这个任务的用户实例.
这里有一个例子, 就是使用普通的python函数声明为一个用户的任务:

```
    from locust import User, constant

    def my_task(user):
        pass

    class MyUser(User):
        tasks = [my_task]
        wait_time = constant(1)
```

如果tasks属性被置为一个列表,每次一个任务从*tasks*属性中被选取执行.
但如果*tasks*是一个字典——键为可调用对象, 值为int型——任务被以整型值为概率随机选择执行.
所以一个任务看起来是这样的:

    {my_task: 3, another_task: 1}

*my_task*将会3倍于*another_task*的可能性执行.

内部的上面的字典实际上会被扩展成一个像下面这样的列表:

    [my_task, my_task, my_task, another_task]

然后Python的*random.choice()*用来从列表中选取任务.

## @tag装饰器

使用*@tag <locust.tag>*装饰器标记任务, 你就可以使用*--tags*和*--exclude-tags*参数挑选被执行的任务.
参考以下例子:

```
    from locust import User, constant, task, tag

    class MyUser(User):
        wait_time = constant(1)

        @tag('tag1')
        @task
        def task1(self):
            pass

        @tag('tag1', 'tag2')
        @task
        def task2(self):
            pass

        @tag('tag3')
        @task
        def task3(self):
            pass

        @task
        def task4(self):
            pass
```

如果你使用*--tags tag1*启动测试, 只有*task1*和*task2*将会执行.
如果你使用*--tags tag2 tag3*启动, 只有*task2*和*task3*将会执行.

*--exclude-tags*是以相反的方式工作. 所以, 如果你使用*--exclude-tags tag3*, 只有*task1*, *task2*和*task4*将会执行
Exclusion always
排除要高于包含关系, 所以如果有一个任务既有被包含又被排除的标记, 它将不会执行.

## Events

如果你想在你的测试中运行一些装配代码, 通常把它们在模块级别放在你的locust文件中就足够了.
但是有时候你需要在运行中的特定时刻做一些处理. 为了满足这种需求, Locust提供了事件钩子.

## test_start和test_stop

如果你需要在启动或者结束负载测试时运行一些代码,
应该使用*test_start <locust.event.Events.test_start>*和*test_stop <locust.event.Events.test_stop>*
事件. 你可以在你的locust文件模块级别为这些事件设置监听器.

```
    from locust import events

    @events.test_start.add_listener
    def on_test_start(environment, **kwargs):
        print("A new test is starting")

    @events.test_stop.add_listener
    def on_test_stop(environment, **kwargs):
        print("A new test is ending")
```

## init

*init*事件将在每个Locust进程开始的时候被触发.
这在分布式模式下尤其有用, 在这种模式下每一个worker进程(不是每一个user)都需要进行一些初始化工作.
例如, 我们假设你有一些全局状态会在所有被当前进程大量产生的user中所用到:

```
    from locust import events
    from locust.runners import MasterRunner

    @events.init.add_listener
    def on_locust_init(environment, **kwargs):
        if isinstance(environment.runner, MasterRunner):
            print("I'm on master node")
        else:
            print("I'm on a worker or standalone node")
```

## 其它事件

查看*使用事件钩子扩展Locust <extending_locust>*获取更多其它事件和例子, 以了解如何使用它们.

## HttpUser类

类*HttpUser <locust.HttpUser>*是最经常使用的*User <locust.User>*类.
它添加了一个可以用来产生HTTP请求的*client <locust.HttpUser.client>*属性.

```
    from locust import HttpUser, task, between

    class MyUser(HttpUser):
        wait_time = between(5, 15)

        @task(4)
        def index(self):
            self.client.get("/")

        @task(1)
        def about(self):
            self.client.get("/about/")
```

## client属性 / HttpSession

属性*client <locust.HttpUser.client>*
是类*HttpSession <locust.clients.HttpSession>*的实例. 
*HttpSession*是类*requests.Session*的子类和包装器.
它有很完善的文档, 并被广泛熟知.
*HttpSession*所增加的就是向Locust报告请求结果(成功/失败, 响应时间, 响应长度, 名字).

它包含所有HTTP方法:
*get <locust.clients.HttpSession.get>*, *post <locust.clients.HttpSession.post>*, *put <locust.clients.HttpSession.put>*.
像*requests.Session*一样, 它在请求之间维护cookies, 所以它可以很容易地用来登录Web网站.


产生一个POST请求, 获取响应数据, 在第二个请求中重用会话cookie:
```
    response = self.client.post("/login", {"username":"testuser", "password":"secret"})
    print("Response status code:", response.status_code)
    print("Response text:", response.text)
    response = self.client.get("/my-profile")
```

HttpSession捕获Session抛出的所有*requests.RequestException*异常(由连接
错误, 超时等类似原因产生的), 而不会返回*status_code*为0和*content*为None的虚假的响应对象

## 验证响应结果

可以认为如果HTTP响应码为OK(<400)请求就是成功的.
但做一些额外的验证是很有用的.

你可以通过使用*catch_response*参数, *with*语句上下文和调用
*response.failure()*标记一个请求是失败的.

```
    with self.client.get("/", catch_response=True) as response:
        if response.text != "Success":
            response.failure("Got wrong response")
        elif response.elapsed.total_seconds() > 0.5:
            response.failure("Request took too long")
```

你也可以即使请求响应码是错误的时候, 标记一个请求是成功的:

```
    with self.client.get("/does_not_exist/", catch_response=True) as response:
        if response.status_code == 404:
            response.success()
```

你甚至可以通过抛出一个异常然后在with语句块外捕获它来完全避免记录一个请求.
或者你可以像如下的示例那样抛出一个*locust exception <exceptions>*, 让Locust捕获它:

```
    from locust.exception import RescheduleTask
    ...
    with self.client.get("/does_not_exist/", catch_response=True) as response:
        if response.status_code == 404:
            raise RescheduleTask()
```

## REST/JSON APIs

这个例子展示了如何调用一个REST API并且验证响应结果:

```
    from json import JSONDecodeError
    ...
    with self.client.post("/", json={"foo": 42, "bar": None}, catch_response=True) as response:
        try:
            if response.json()["greeting"] != "hello":
                response.failure("Did not get expected value in greeting")
        except JSONDecodeError:
            response.failure("Response could not be decoded as JSON")
        except KeyError:
            response.failure("Response did not contain expected key 'greeting'")
```

locust插件有一个现成的类来测试REST API:
[RestUser](https://github.com/SvenskaSpel/locust-plugins/blob/master/examples/rest_ex.py).


## 请求分组

网站中有一些页面的URL含有动态参数是很常见的.
所以在统计中将这些URL分类成组是需要的.
可以通过给*HttpSession's <locust.clients.HttpSession>*不同的请求方法传递一个*name*参数来实现.

示例:

```
    # Statistics for these requests will be grouped under: /blog/?id=[id]
    for i in range(10):
        self.client.get("/blog?id=%i" % i, name="/blog?id=[id]")
```

可能会有一些情况无法向请求函数传递一个参数, 比如与封装一个请求会话的库
或者SDK交互时.给请求分组的另一个可选方式是设置*client.request_name*属性.

```
    # Statistics for these requests will be grouped under: /blog/?id=[id]
    self.client.request_name="/blog?id=[id]"
    for i in range(10):
        self.client.get("/blog?id=%i" % i)
    self.client.request_name=None
```

如果你想使用微模板形成一系列的多分组, 你可以使用*client.rename_request()*上下文管理器.

```
    @task
    def multiple_groupings_example(self):
        # Statistics for these requests will be grouped under: /blog/?id=[id]
        with self.client.rename_request("/blog?id=[id]"):
            for i in range(10):
                self.client.get("/blog?id=%i" % i)

        # Statistics for these requests will be grouped under: /article/?id=[id]
        with self.client.rename_request("/article?id=[id]"):
            for i in range(10):
                self.client.get("/article?id=%i" % i)
```

## HTTP代理设置 

为提供性能, 我们通过设置*requests.Session*的*trust_env*属性为False, 配置请求在测试环境中不去查找HTTP代理设置.
如果你不想这样, 你可以手动设置*locust_instance.client.trust_env*为True. 
查看[documentation of requests](https://requests.readthedocs.io/en/master/api/#requests.Session.trust_env)
获取更多细节.

## 连接池

每一个*HttpUser <locust.HttpUser>*类创建一个新的*HttpSession <locust.clients.HttpSession>*,
每一个用户实例有它自己的连接池. 这和真实用户如何与一个web服务器交互类似.

但是, 如果你想在所有用户之间共享连接, 你可以使用单一的一个池管理器.
设置*pool_manager <locust.HttpUser.pool_manager>*类属性为*urllib3.PoolManager*的一个实例就可以实现.

```
    from locust import HttpUser
    from urllib3 import PoolManager

    class MyUser(HttpUser):
        # All users will be limited to 10 concurrent connections at most.
        pool_manager = PoolManager(maxsize=10, block=True)
```

更多配置选项参看[urllib3 documentation](https://urllib3.readthedocs.io/en/stable/reference/urllib3.poolmanager.html).

## TaskSets

TaskSets是一种对web网站/系统层次化构建测试的方法.
你可以在<tasksets>章节读到更多详细内容.

## 如何结构化你的测试代码

始终记得locustfile.py只是被Locust导入的普通的Python模块是重要的.
在这个模块中, 你可以任意地从其它python代码中导入, 这就像你在任意Python
程序中做的那样.

当前的工作目录会自动被添加到python的*sys.path*中.
所以位于这个工作目录的file/module/packages都可以用python*import*语句导入.

对于小规模的测试, 把所有测试代码写在一个*locustfile.py*文件中是合适的.
当时对于大型的测试, 你可能更希望能将代码拆分到多个文件和目录中.

如何安排测试代码的结构完全取决于你的设计.
但是我们建议你遵循Python的最佳实践.
这里有一个设想的Locust工程文件结构的例子:

* Project root

  * ``common/``

    * ``__init__.py``
    * ``auth.py``
    * ``config.py``
  * ``locustfile.py``
  * ``requirements.txt`` (External Python dependencies is often kept in a requirements.txt)

一个拥有多个locust文件的项目, 也可以让它们分开放在子目录中:

* Project root

  * ``common/``

    * ``__init__.py``
    * ``auth.py``
    * ``config.py``
  * ``my_locustfiles/``

    * ``api.py``
    * ``website.py``
  * ``requirements.txt``


按上述项目结构, 你的locust文件可以如下导入通用模块:

    import common.auth
