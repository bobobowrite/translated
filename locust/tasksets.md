# TaskSet类

当你对一个层级结构的网站进行性能测试的时候, 构建同样层次结构的测试部件将会是很有帮助的.
为此, Locust提供了TaskSet类. TaskSet是任务的集合, 在其中定义的任务像直接定义在User类中一样运行.

注意:

    TaskSet是一个极少情况下使用的高级特性. 大多数情况, 通常的Python循环
    和控制语句可以实现的事情,你最好避免使用它. 
    这里有几个陷阱, 其中最经常出现的就是忘记调用*self.interrupt()*

```
    from locust import User, TaskSet, constant
    
    class ForumSection(TaskSet):
        wait_time = constant(1)

        @task(10)
        def view_thread(self):
            pass
        
        @task
        def create_thread(self):
            pass
        
        @task
        def stop(self):
            self.interrupt()
    
    class LoggedInUser(User):
        wait_time = constant(5)
        tasks = {ForumSection:2}
        
        @task
        def my_task(self):
            pass
```

一个TaskSet也可以使用@task注解, 定义为User或者TaskSet的内部类:

```
    class MyUser(User):
        @task
        class MyTaskSet(TaskSet):
            ...
```

一个TaskSet中的任务也可以是TaskSet, 允许嵌套任意数量的层次.
这让我们能够更真实地模拟用户行为.

例如我们可以按如下的结构定义TaskSet:

```
    - Main user behaviour
      - Index page
      - Forum page
        - Read thread
          - Reply
        - New thread
        - View next page
      - Browse categories
        - Watch movie
        - Filter movies
      - About page
```

一个User线程运行时会选取一个TaskSet类来执行, 创建这个TaskSet的一个实例, 
然后交给其执行. 其执行过程是选取所拥有的一个Task执行,而后休眠一个特定的时长,
该时长是由User的*wait_time*函数定义的
(除非在TaskSet类中直接声明一个*wait_time*函数来取代), 然后选取一个新的任务...如此往复循环.

TaskSet实例中具有指向所属User的引用*self.user*. 它还有一个用以快捷访问所属User
中的client属性的引用. 所以你可以使用*self.client.request()*方法生成一个request请求,
就如同你的task直接定义在一个HttpUser里一样.


## 中断一个TaskSet

需要知道的关于TaskSet的一件重要的事情就是它们自身将一直循环执行所拥有的任务. 
要向它们的所属的User或者TaskSet交回执行权, 需要被开发者显式调用方法*TaskSet.interrupt()*方法.

在下方示例中, 如果我们没有调用*self.interrupt()*停止任务, 
模拟的用户一旦执行Forum任务集合将不会停止:

```
    class RegisteredUser(User):
        @task
        class Forum(TaskSet):
            @task(5)
            def view_thread(self):
                pass
            
            @task(1)
            def stop(self):
                self.interrupt()
        
        @task
        def frontpage(self):
            pass
```

我们可以连同任务权重一起使用interrupt函数, 定义一个模拟用户停止访问论坛页面的概率.


## TaskSet和User类中任务的区别

属于TaskSet下的任务和直接定义在User下的任务的一个不同就是, 它们执行时
传入的一个参数(装饰器声明任务时使用的参数*self*)是引用TaskSet实例而不是User实例.
User实例可以在一个TaskSet实例中通过属性直接引用使用,
TaskSet同时还拥有访问User实例中client属性的便捷方式.


## 引用User实例或者父级TaskSet实例

一个TaskSet实例具有user属性指向它所属的User实例, 和parent属性指向它的
父级TaskSet实例.


## Tags和TaskSets

可以使用*@tag<locust.tag>*装饰器为TaskSet标记, 这和在普通任务上打标
记一样, 但有几点细微的区别值得提到. 给一个TaskSet标记会自动应用在其
下的所有任务中. 此外, 如果你给嵌入在TaskSet中的任务打标记时, Locust会
执行该任务即使TaskSet没被标记.


## SequentialTaskSet类

类*SequentialTaskSet <locust.SequentialTaskSet>*
是一个其下任务可以按定义的顺序执行的TaskSet.
在TaskSet中嵌套使用SequentialTaskSet是可以的, 反之亦然.

例如, 下例中代码将会顺序请求URL/1-/4, 然后循环执行:

```
    def function_task(taskset):
        taskset.client.get("/3")
    
    class SequenceOfTasks(SequentialTaskSet):
        @task
        def first_task(self):
            self.client.get("/1")
            self.client.get("/2")
        
        # you can still use the tasks attribute to specify a list of tasks
        tasks = [function_task]
        
        @task
        def last_task(self):
            self.client.get("/4")
```

注意, 如果只是想顺序地发生一些请求, 你是不需要SequentialTaskSet的.
只需要简单地使用定义在一个用户中单独的一个任务中即可.
