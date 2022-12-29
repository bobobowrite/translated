URL 路由
========

当应用程序合并有多个控制器或者视图函数(无论如何你喜欢这么称呼它们)时, 我们就需要一个分发器了.
一个简单的方案是应用正则表达式测试***PATH_INFO***, 然后调用注册的回调函数并且返回值.

Werkzeug提高了一个类似于***Routes***的更强大的系统.
本页所提到的所有的对象须从模块***werzeug.routing***中导入, 而不是从模块***werkzeug***.

[Routes](https://routes.readthedocs.io/en/latest/)


快速入门
========

这里是一个简单的例子, 示例在博客应用程序中URL的定义:

    from werkzeug.routing import Map, Rule, NotFound, RequestRedirect

    url_map = Map([
        Rule('/', endpoint='blog/index'),
        Rule('/<int:year>/', endpoint='blog/archive'),
        Rule('/<int:year>/<int:month>/', endpoint='blog/archive'),
        Rule('/<int:year>/<int:month>/<int:day>/', endpoint='blog/archive'),
        Rule('/<int:year>/<int:month>/<int:day>/<slug>',
             endpoint='blog/show_post'),
        Rule('/about', endpoint='blog/about_me'),
        Rule('/feeds/', endpoint='blog/feeds'),
        Rule('/feeds/<feed_name>.rss', endpoint='blog/show_feed')
    ])

    def application(environ, start_response):
        urls = url_map.bind_to_environ(environ)
        try:
            endpoint, args = urls.match()
        except HTTPException, e:
            return e(environ, start_response)
        start_response('200 OK', [('Content-Type', 'text/plain')])
        return [f'Rule points to {endpoint!r} with arguments {args!r}'.encode()]

这些到底是什么呢? 首先我们创建了一个新的类***Map***, 它用来保存一批URL规则.
然后我们给它传递一个包含类***Rule***的对象的列表.

每个类***Rule***的对象由一个表示规则的字符串和一个将这些规则视作别名的端点.
多规则可以有共同的端点, 但是应该有不同的参数形成URL的构造.

URL规则的格式是简单易懂的, 下面会详细解释.

在WSGI应用程序内部, 我们绑定***url_map***到当前的请求, 这会返回一个新的
***MapAdapter***. 这个url_map适配器之后将会被用来对当前的请求进行匹配或者建造域.

方法***MapAdapter.match***将会返回一个***(endpoint, args)***形式的元组, 或者抛
出三种异常之一: ***werkzeug.exceptions.NotFound***, ***werkzeug.exceptions.MethodNotAllowed***,
***werkzeug.exceptions.RequestRedirect***.  
查看API文档***MapAdapter.match***方法获取更多关于这些异常的细节.


规则格式
========

规则字符串是带着变量占位符的URL路径, 其格式为: ***<converter(arguments):name>***. 
***converter***和***arguments***(带着括号)是可选的.
如果没有指定***converter***， 则使用默认的***converter***(***string***).
可用的***converter***将在下方介绍.

以***/***结尾的规则是分支, 不是以***/***结尾的是叶子.
如果***strict_slashes***是激活的(默认是激活), 访问一个不以***/***结尾的分支URL
将被重定向到添加上***/***的URL.

许多HTTP服务器收到请求后会把连续的斜线字符合并成一个.
如果***merge_slashes***是激活的(默认是激活), 在匹配和构建时, 
规则会合并非变量部分的斜线字符.
访问带有连续斜线的URL会重定向到合并斜线后的URL.
如果你希望为一个***Rule***或者***Map***关闭***merge_slashes***, 你也需要恰当地配置你的
Web服务器.


内置转换器
==========

Werkzeug已经内置了通常类型的URL变量转换器.
可以用***Map.converters***来自定义转换器.

* UnicodeConverter

* PathConverter

* AnyConverter

* IntegerConverter

* FloatConverter

* UUIDConverter


Maps, Rules and Adapters
========================

* Map

   * converters

      持有转换器的字典类型. 可以随时修改, 但是只能影响修改后加入的规则.
      如果规则是随着这个列表传递到Map建立的, 那么就得使用构造器参数构建转换器了.

* MapAdapter

* Rule

   * empty


Matchers
========

* StateMachineMatcher


规则工厂
========

* RuleFactory

   * get_rules

* Subdomain

* Submount

* EndpointPrefix


规则模板
========

* RuleTemplate


自定义转换器
============

你可以自定义转换器以增加内置转换器所没有的处理.
要自定义转换器, 只需继承类***BaseConverter***, 然后将该类设置为类***Map***的***converters***参数,
或者把它添加到***url_map.converters***参数.

转换器应该有一个用于匹配的正则表达式存放在***regex***属性.
如果转换器能够处理URL中的动态参数, 它应该在其***__init__***构造方法中接收
它们.整个正则表达式应该匹配一个分组用于转换.

如果自定义转换器可以匹配正斜杠***/***, 它应该有
属性***part_isolating***设置为False。这将确保
使用自定义转换器的规则正确匹配。

它可以实现一个***to_python***方法来将匹配的字符串转换为
其他一些对象.
这也可以做额外的验证: 如果和***regex***属性不匹配，则引发一个
***werkzeug.routing.ValidationError***.
抛出任何其他异常将导致 500 错误。

它可以实现一个***to_url***方法来将Python对象转换为构建URL时的字符串.
此处引发的任何错误都将转换为***werkzeug.routing.BuildError***并最终导致 500 错误.

这个例子实现了一个能够匹配字符串"yes", "no"和"maybe"的***BooleanConverter***,
对于"maybe"返回一个随机值:

```
    from random import randrange
    from werkzeug.routing import BaseConverter, ValidationError

    class BooleanConverter(BaseConverter):
        regex = r"(?:yes|no|maybe)"

        def __init__(self, url_map, maybe=False):
            super().__init__(url_map)
            self.maybe = maybe

        def to_python(self, value):
            if value == "maybe":
                if self.maybe:
                    return not randrange(2)
                raise ValidationError
            return value == 'yes'

        def to_url(self, value):
            return "yes" if value else "no"

    from werkzeug.routing import Map, Rule

    url_map = Map([
        Rule("/vote/<bool:werkzeug_rocks>", endpoint="vote"),
        Rule("/guess/<bool(maybe=True):foo>", endpoint="guess")
    ], converters={'bool': BooleanConverter})
```

如果你想改变默认的转换器, 可以对***default***键设置一个不同转换器来实现.

Host匹配
========

从Werkzeug 0.7开始,不只是能匹配子域外还能够匹配全部的host了.
你需要传参***host_matching=True***给Map类的构造函数, 并为所有路由提供***host***参数来
开启这个特性,

```
    url_map = Map([
        Rule('/', endpoint='www_index', host='www.example.com'),
        Rule('/', endpoint='help_index', host='help.example.com')
    ], host_matching=True)
```

在Host部分当然还是可以使用动态变量的:

```
    url_map = Map([
        Rule('/', endpoint='www_index', host='www.example.com'),
        Rule('/', endpoint='user_index', host='<user>.example.com')
    ], host_matching=True)

```

WebSockets
==========

如果一个Rule对象使用***websocket=True***创建,
它将只匹配绑定带有***url_scheme***, ***ws***或者***wss***请求的***Map***.


注意:

   Werkzeug除了路由外没有更进一步的WebSocket支持.
   这个功能绝大多数是使用ASGI项目.

```
    url_map = Map([
        Rule("/ws", endpoint="comm", websocket=True),
    ])
    adapter = map.bind("example.org", "/ws", url_scheme="ws")
    assert adapter.match() == ("comm", {})
```

如果只是匹配一个WebSocket规则,
而绑定是HTTP(或者只是匹配HTTP, 而绑定是WebSocket),
一个***WebsocketMismatch***(从***werkzeug.exceptions.BadRequest***获得)异常会被抛出.

WebSocket URLs的scheme不同, 所以匹配规则总是使用sheme和host一起构建, 
这里***force_external=True***是内部默认的.

```
    url = adapter.build("comm")
    assert url == "ws://example.org/ws"
```

状态机匹配
==========

默认的匹配算法使用一个在请求路径不同部分进行转换的状态机来匹配.
要理解它是怎么工作的, 参考这个规则:

    /resource/<id>

首先这个规则被分解成两个***RulePart***.
第一个是一个内容为***resource***的静态部分,
第二个是一个动态的和需要一个正则匹配***[^/]+***.

状态机之后创建一个初始状态来表示首先的规则***/***.
这个初始化状态有一个单一的静态的转换到下一个表示规则第二个***/***的下一个状态.
这个状态有一个单一的动态的装换到包含规则的最终状态.

路径匹配的过程就是这样工作: 匹配器启动, 初始化状态, 根据条件状态转换.

很清晰地, 一个待审查的路径***/resource/2***
有***""***, ***resource***和***2***三部分匹配转换, 因此一个规则将匹配.
然而***/other/2***将无法匹配, 因为没有从初始状态按照***other***可进行的转换.

从这个规则仅有的改变就是:
如果一个***RulePart***不是隔离的, 即它将匹配***/***. 在这种情况, ***RulePart***
被认为是最终的, 并且表示一个必须包含所有带审查路径随后的部分的转换.
