# 定制负载模型

有时一个完全定制化的负载测试无法通过简单的设置或者修改用户数量和产生速率来实现.

例如你可能需要产生一个负载高峰或者在特定的时间上升和下降负载. 通过使
用*LoadTestShape*类你可以全程完全控制用户数量和产生速率.

在你的locust文件中定义一个继承*LoadTestShape*的子类.
Locust可以自动发现这个类型并且使用.

你在这个类里定义一个*tick()*方法, 返回一个包含用户数量和产生速率的元组
(或者*None*来停止测试).
Locust大约每秒一次将会调用*tick()*方法.

你还可以调用这个类的*get_run_time()*方法, 来获取测试的运行时长.

## 例子

这个模型类将会以100为单位增加用户数量, 并且在10分钟后停止负载测试:

```
    class MyCustomShape(LoadTestShape):
        time_limit = 600
        spawn_rate = 20
        
        def tick(self):
            run_time = self.get_run_time()

            if run_time < self.time_limit:
                # User count rounded to nearest hundred.
                user_count = round(run_time, -2)
                return (user_count, spawn_rate)

            return None
```

进一步的功能在[这里](https://github.com/locustio/locust/tree/master/examples/custom_shape)的例子中会进一步演示, 包括:

- 生成一个双波模型
- 像K6的阶段时间
- 像Visual Studio的单步负载模式

一个能够为你定制负载模型提供帮助的更进一步的方法是
*get_current_user_count()*, 这个方法返回激活用户的总数.

这个方法可以用来阻止在足够数量的用户产生之前就进行了后续步骤.
这在每个用户初始化过程低速或者耗时不稳定时尤其有用.
如果这听起来像你的用例, 可以继续查看
[github](https://github.com/locustio/locust/tree/master/examples/custom_shape/wait_user_count.py)上的示例.

## 用不同的负载配置组合用户

如果你使用Web UI, 可以添加参数*---class-picker <class-picker>*来选择使用什么负载模型.
但是让你的User定义和LoadTestShape在不同的文件通常更加灵活.
例如, 如果你的高负载和低负载分别定义在high_load.py和low_load.py中:

```
    $ locust -f locustfile.py,low_load.py

    $ locust -f locustfile.py,high_load.py
```

## 限定每次产生的用户类型

加入*user_classes*元素让你能够更精细地控制:

```
    class StagesShapeWithCustomUsers(LoadTestShape):

        stages = [
            {"duration": 10, "users": 10, "spawn_rate": 10, "user_classes": [UserA]},
            {"duration": 30, "users": 50, "spawn_rate": 10, "user_classes": [UserA, UserB]},
            {"duration": 60, "users": 100, "spawn_rate": 10, "user_classes": [UserB]},
            {"duration": 120, "users": 100, "spawn_rate": 10, "user_classes": [UserA,UserB]},

        def tick(self):
            run_time = self.get_run_time()

            for stage in self.stages:
                if run_time < stage["duration"]:
                    try:
                        tick_data = (stage["users"], stage["spawn_rate"], stage["user_classes"])
                    except:
                        tick_data = (stage["users"], stage["spawn_rate"])
                    return tick_data

            return None
```

这个负载模型可以在开始的10秒钟产生10个UserA用户. 接下来的20秒钟产生
40个UserA或者UserB用户. 如此重复直到stages结束.
