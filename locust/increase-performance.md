## 使用更快的HTTP客户端提高性能

Locust默认的HTTP客户端是[python-requests](http://www.python-requests.org/). 
它提高了一个很友好的API, 为众多python程序员所熟悉, 并被维护地良好.
但是如果你想进行更高吞吐量的测试, 而且又有有限的硬件资源时, 它就显得不够高效了.

因此, Locust提供了类***FastHttpUser <locust.contrib.fasthttp.FastHttpUser>***
该类使用的是[geventhttpclient](https://github.com/gwik/geventhttpclient/),
它提供了一个相似的API并且显著地使用更少的CPU时间, 有时能够在给定的硬件
条件下提高每秒最大请求数5到6倍.

虽然无法指出你自己的硬件准确的处理能力,
在最佳场景下,相对于使用HttpUser每核每秒大约850请求,
使用FastHttpUser可以达到每核每秒接近5000请求(在一个2018 MacBook Pro i7 2.6GHz上测试).
在现实世界中你的结果可能非常不同,
你可能只会看到很小的提升, 如果你加载的测试还会做其它CPU密集计算的事情.

注意:

    只要你的CPU没有过载, FastHttpUser的响应时间应该几乎与HttpUser完全一致.
    它不会在这个方面上加速, 当然, 它无法加速你在测试的系统.

    

## 怎么使用FastHttpUser

只需继承FastHttpUser代替HttpUser即可:

```
    from locust import task, FastHttpUser
    
    class MyUser(FastHttpUser):
        @task
        def index(self):
            response = self.client.get("/")
```

## 并发

一个独立的FastHttpUser/geventhttpclient会话能够运行并发请求, 你只需要为每个请求运行greenlets即可:

```
    @task
    def t(self):
        def concurrent_request(url):
            self.client.get(url)

        pool = gevent.pool.Pool()
        urls = ["/url1", "/url2", "/url3"]
        for url in urls:
            pool.spawn(concurrent_request, url)
        pool.join()

```

注意:

    FastHttpUser/geventhttpclient和HttpUser/python-requests很像, 
    但是有时有一些不易察觉的差别. 特别是当你使用客户端库的内部模块(例如手动管理cookie)的时候.

## API


### FastHttpUser class

类: locust.contrib.fasthttp.FastHttpUser

    成员属性: network_timeout, connection_timeout, max_redirects, max_retries, insecure, concurrency, client_pool


### FastHttpSession class

类: locust.contrib.fasthttp.FastHttpSession

    成员属性: request, get, post, delete, put, head, options, patch

类: locust.contrib.fasthttp.FastResponse

    成员属性: content, text, json, headers
