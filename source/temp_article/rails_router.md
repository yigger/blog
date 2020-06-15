### Rails 路由
> 本文的路由分析是基于 rails 4.0.13

### 前言
在创建完 rails 应用以后，默认的会生成 `config/routes.rb` 文件，平时我们所用到的路由入口都可以定义此处，而使用方法也很简单，只需要在块内简单定义一下路由的路径和响应的 Controller 即可，如下所示，可以简单的定义一个 http GET /welcome 的请求，然后到 HomeController 找到名为 welcome 的 action

```ruby
Rails.application.routes.draw do
	get '/welcome' => 'home#welcome'
	# do something
end
```

可以看到整个过程非常简单，上手即用，没有过多的逻辑进行参杂，下面主要分两部分来讲解路由体系，1. 路由的定义 2. 路由是如何响应的

### 路由是如何被定义的？
下面开始从第一行代码开始分析，`Rails.application.routes.draw` 做了什么？
```ruby
# routes 是类方法，它初始化了 RouteSet 对象
def routes
  @routes ||= ActionDispatch::Routing::RouteSet.new
  @routes.append(&Proc.new) if block_given?
  @routes
end

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
  mapper.instance_exec(&block) # 注意这个方法，下面有解释 instance_exec 方法
end


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

@routes
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
	# 此方法主要处理我们在 route.rb 定义的参数
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

跟着源码绕着绕着其实也有点乱了，很多时候，如果看源码觉得乱，那此时建议画张图理一理思路，此时我也是如此..





