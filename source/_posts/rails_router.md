---
title: Rails 关于路由的一些事
date: 2020-07-18
categories: rails
---

### Rails 路由
> 本文的路由分析是基于 rails 4.0.13

### 前言
在创建完 rails 应用以后，默认的会生成 `config/routes.rb` 文件，平时我们所用到的路由入口都可以定义此处，而使用方法也很简单，只需要在块内简单定义一下路由的路径和响应的 Controller 即可，如下所示，可以简单的定义一个 http GET /welcome 的请求，然后到 HomeController 找到名为 welcome 的 action 并执行熟悉的 MVC 过程（ps: 至于代码如何响应并发送，不过这个暂时不在本节讨论范围，本节主要是看看路由是怎么被定义的

```ruby
Rails.application.routes.draw do
	get '/welcome' => 'home#welcome'
end
```

### 路由是如何被定义的？
下面开始从第一行代码开始分析，`Rails.application.routes.draw` 做了什么？

```ruby
# file: railties-4.0.13/lib/rails/engine.rb
# Rails.application.routes
# routes 是类方法，它初始化了 RouteSet 对象
def routes
  @routes ||= ActionDispatch::Routing::RouteSet.new
  @routes.append(&Proc.new) if block_given?
  @routes
end

# file: actionpack-4.0.13/lib/action_dispatch/routing/route_set.rb
# Rails.application.routes.draw
# draw 方法接受一个代码块，就是 do .. end 包裹的一系列代码块区域
def draw(&block)
  clear! unless @disable_clear_and_finalize
  eval_block(block)
  finalize! unless @disable_clear_and_finalize
  nil
end

# eval_block(block) 代码继续往下走
def eval_block(block)
	# 省略部分代码
  mapper = Mapper.new(self)
  # 这个方法是处理我们定义的内容，通过此方法可以把 block 传到 mapper
  # eg:
  # Gitlab::Application.routes.draw do
  #		get '/welcome' => 'home#welcome'
  # end
  # 
  # instance_exec 可以看下面的官方说明
  mapper.instance_exec(&block) 
end

# PS:
# instance_exec 是顶级类 Object 的方法，它接收一个块，块内可以使用示例对象的上下文
# 官方示例：
class KlassWithSecret
  def initialize
    @secret = 99
  end
end
# 实例化对象
k = KlassWithSecret.new
# instance_exec 接收一个块，块内可以正常使用实例化对象 k 的 @secret 对象（可能有点拗口，理解一下）
# 原文: https://apidock.com/ruby/Object/instance_exec
k.instance_exec do
	@secret+4
end
```

那么，路由的定义可以简单的理解为
```ruby
Rails.application.routes.draw do
	get '/welcome' => 'home#welcome'
end

=>

m = ActionDispatch::Routing::Mapper.new
m.instance_exec do
	get '/welcome' => 'home#welcome'
end
```

我们平时使用的 `get`、`post`、`resources` 等等方法都属于这个类 `ActionDispatch::Routing::Mapper` 的实例方法

下面继续看看 `get` 方法做了什么事情
```ruby
 module HttpHelpers
   def get(*args, &block)
   	map_method(:get, args, &block)
   end
   def post(*args, &block)
   	map_method(:post, args, &block)
   end
   def put(*args, &block)
   	map_method(:put, args, &block)
   end
   # delete, patch 方法也是类似的定义
   
   private
   def map_method(method, args, &block)
     # 过滤，筛选并返回最后一个参数
     # a = [{a: 1}, {b: 2}]
		 # a.extract_options! => {:b=>2}
     options = args.extract_options!
     options[:via] = method
     match(*args, options, &block)
     self
   end
 end 
```

无论是 get、post、put 或是其它，最终的处理逻辑都是跑到 `match` 方法，而 match 方法的使用方式有很多种，可以看到官方的注释和方法的定义

```ruby
# match 'path' => 'controller#action'
# match 'path', to: 'controller#action'
# match 'path', 'otherpath', on: :member, via: :get
def match(path, *rest)
	# 此方法主要处理我们在 route.rb 定义的所有内容
	# 把 *rest 转化为 options，最终传入下一个方法 decomposed_match
end

def decomposed_match(path, options)
	...
	# 终于到了主菜部分，前面把数据组装好以后，通过这个方法把添加路由，具体怎么添加呢？
	add_route(path, options)
	...
end

def add_route(action, options)
	# 此处省略 options 和 action 的处理，具体可以参考源代码
	...
  mapping = Mapping.new(@set, @scope, URI.parser.escape(path), options)
  app, conditions, requirements, defaults, as, anchor = mapping.to_route
  @set.add_route(app, conditions, requirements, defaults, as, anchor)
end
```

 `add_route` 之后又 `add_route`，在阅读过程中出现了贼多的 `add_route`，感觉就是一层层的套娃。

But，此处就是最最最核心的方法，涉及 route 的存放位置，其实核心代码主要就是两行，咱们可以一行行进行解读， to_route 到底做了什么，这里就不展开说明了，简单来说 to_route 只是把我们的参数进行了处理，使之更加*面向对象*

`app, conditions, requirements, defaults, as, anchor = mapping.to_route`

| 变量名称     | 作用                                                         | 备注 |
| ------------ | ------------------------------------------------------------ | ---- |
| app          | Constraints 类实例对象，内部映射着对应的 controller 类       |      |
| conditions   | 上层传过来的 options                                         |      |
| requirements | 可配置此路由的正则，或者 format；主要用在之后的路由匹配过程，不符合正则或 format 的情况是不会进入到对应路由的响应 |      |
| default      | 用处未知                                                     |      |
| as           | 一般用作路由的别名，eg: get 'repositories/latest/created' => 'project_tags#latest', as: :latest_created_projects |      |
| anchor       | 用处未知                                                     |      |

由以上可知，`to_route`方法的作用是对用户在 router.rb 定义的 dsl 做一个更细化的处理，处理完成后向 mapper 抛出对应的参数，比如**路由的名称**，**路由的规则**，**路由的响应**，**以及其它可配置选项**。在此方法的最后，把这些参数都传到 `@set` 实例变量进行存储起来，而 `@set` 实例变量在开头 `Rails.application.routes` 就已经初始化了

```ruby
# Rails.application.routes.draw 的 Rails.application.routes 就是此方法
def routes
  @routes ||= ActionDispatch::Routing::RouteSet.new
  @routes.append(&Proc.new) if block_given?
  @routes
end

# actionpack-4.0.13/lib/action_dispatch/routing/route_set.rb
class ActionDispatch::Routing::RouteSet
  def initialize(request_class = ActionDispatch::Request)
    ...
    @set    = Journey::Routes.new
    ...
  end
end
```

至此，我们已经解决了第一个问题，路由是怎么被定义以及怎么被存储的：路由在定义之后，在启动应用的过程(rails s)，会加载 router.rb 文件，然后初始化类 `@set = Journey::Routes.new`，之后将我们定义的诸如`get '/welcome' => 'home#welcome'`经过加工处理后，存储到 `@set` 的 routers 变量

在阅读的途中，我画了一下调用栈来帮助理解
![调用栈](/images/action_pack/1.png)

