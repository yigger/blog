---
title: Rails 的 ActiveSupport::Concern
date: 2019-07-18
tags:
---
相信使用过 rails 的朋友们都经常会看到或者会使用到 ActiveSupport::Concern 这个模块，但是有没有想过为什么要使用这个模块呢？

因为以下内容会涉及上篇文章的内容，如果还不了解 ruby 的 include 和 extend 关键字的话，可先看看 [关于 Ruby 的 include 和 extend](https://yigger.cn/2019/07/17/ruby%e7%9a%84include-extend/ )

在上一篇文章的末尾我们说到可以通过 include 的钩子方法 included 来进行引入类方法，在看了 ActiveSupport::Concern 之后，我相信你会有新的领悟。

源码地址: https://github.com/rails/rails/blob/master/activesupport/lib/active_support/concern.rb

总的来说，Concern 模块所做的方法就是封装了 include 的钩子，之后以一种更友好的方式来引入模块，此话怎讲？看看示例吧

通常来说，按照上一篇文章，这段代码还这么写..
```ruby
# 示例代码01
module Skill
  def self.included(base)
    base.extend(ClassMethod)
    base.class_eval do
      def fly
        puts 'I can fly'
      end
    end
  end
  
  module ClassMethod
    def run
      puts  'Everybody can run'
    end
  end
end

class User
  include Skill
end

user = User.new
user.fly # 输出 I can fly
```

**使用了 ActiveSupport::Concern 改进后**

```ruby
# 示例代码02
require 'active_support/concern'
module Skill
  extend ActiveSupport::Concern

  included do
    def fly
      puts 'I can fly'
    end
  end
  
  class_methods do
    def run
      puts  'Everybody can run'
    end
  end
end

class User
  include Skill
end

user = User.new
user.fly # 输出 I can fly
```

仔细对比一下会发现，示例代码01 和 02 的区别在于原来的 `ClassMethod` 使用 `class_methods` 代码块来代替，而原来的 include 回调方法也使用了 `included` 代码块来代替

对应的查看一下源码你就会发现，其实 concern 模块只是帮我们再次封装了一层而已，我们分两点来查看

### class_methods 方法做了什么处理？
```ruby
# 源代码01
# Define class methods from given block.
# You can define private class methods as well.
# ....
def class_methods(&class_methods_module_definition)
  mod = const_defined?(:ClassMethods, false) ?
    const_get(:ClassMethods) :
    const_set(:ClassMethods, Module.new)

  mod.module_eval(&class_methods_module_definition)
end
```

通过 class_methods 的源码和注释可以知道，class_methods 所做的处理无非就是把代码块的内容整合到名为 ClassMethods 的代码块中

### included 方法做了什么处理？
```ruby
# 源码02
# Evaluate given block in context of base class,
# so that you can write class macros here.
# When you define more than one +included+ block, it raises an exception.
def included(base = nil, &block)
  if base.nil?
    if instance_variable_defined?(:@_included_block)
      if @_included_block.source_location != block.source_location
        raise MultipleIncludedBlocks
      end
    else
      @_included_block = block
    end
  else
    super
  end
end
```

咋看好像这个方法也没干啥啊，只是简单的变量赋值，好像也没有说像 `base.extend` 这样的代码？其实源文件也就几个方法，再仔细看看，发现一个“可疑的家伙” -  `append_features` 方法，方法倒是有这么句 `base.extend const_get(:ClassMethods) if const_defined?(:ClassMethods)` 代码，这个操作有点像之前在 `def self.included(base)` 里面所做的操作，而且也没有显式的进行调用，所以我推断这个方法必有蹊跷

### append_features 钩子
果然，一顿网络冲浪后，发现这个方法并不简单，原来 `append_features` 才是 "真正干活的回调方法"，还是看看代码比较好理解

```ruby
module A
  def self.included(target)
    v = target.instance_methods.include?(:method_name)
    puts "in included: #{v}"
  end

  def self.append_features(target)
    v = target.instance_methods.include?(:method_name)
    puts "in append features before: #{v}"
    super
    v = target.instance_methods.include?(:method_name)
    puts "in append features after: #{v}"
  end

  def method_name
  end
end

class X
 include A
end

# 以上代码的输出为
# in append features before: false
# in append features after: true
# in included: true
```

没错，append_features 的方法是先于 included 执行的回调方法，**可以在引入模块的前后**进行基本的变量设置等操作

OK，了解了 `append_features` 方法之后，再回到源码上看下面这段代码，你会发现，“卧槽，还是看不懂啊，他在干什么”
```ruby
def append_features(base) #:nodoc:
  if base.instance_variable_defined?(:@_dependencies)
    base.instance_variable_get(:@_dependencies) << self
    false
  else
    return false if base < self
    @_dependencies.each { |dep| base.include(dep) }
    super
    base.extend const_get(:ClassMethods) if const_defined?(:ClassMethods)
    base.class_eval(&@_included_block) if instance_variable_defined?(:@_included_block)
  end
end
```

其实这段代码在处理一个依赖的问题，考虑下面的情景
```ruby
module Foo
  def self.included(base)
    base.class_eval do
      def self.method_injected_by_foo
        ...
      end
    end
  end
end

module Bar
  def self.included(base)
    base.method_injected_by_foo
  end
end

class Host
  # 我实际上只想引用 Bar 模块
  # 引入 Foo 模块是因为 Bar 模块的某方法依赖于 Foo，所以我不得不在 Bar 之前引用一下 Foo
  include Foo 
  include Bar
end
```

以上情景就导致了依赖的产生，我只是想使用 Bar 模块却把 Foo 模块也引进来了，这不符合 Ruby 的优雅哲学，所以是否可以这样做呢？

```ruby
module Foo
  def self.included(base)
    base.class_eval do
      def self.method_injected_by_foo
        ...
      end
    end
  end
end

module Bar
  include Foo
  def self.included(base)
    base.method_injected_by_foo
  end
end

class Host
  include Bar
end
```

很遗憾，这种方式是会报错的，为什么？

因为我在终端试过是会报错的，报错信息
```ruby
undefined method `method_injected_by_foo' for Host:Class (NoMethodError)
```

原因在于：在 Bar 模块引用 Foo 的时候，Foo 模块的 included 的回调参数 base 的值不再是 Host，而是 Bar，也就是说 `method_injected_by_foo` 这个方法是属于 Bar 模块的，而不是 Host 类

换用 concern 模块来实现的话，是这样的
```ruby
require 'active_support/concern'

module Foo
  extend ActiveSupport::Concern
  included do
    def self.method_injected_by_foo
      ...
    end
  end
end

module Bar
  extend ActiveSupport::Concern
  include Foo

  included do
    self.method_injected_by_foo
  end
end

class Host
  include Bar # It works, now Bar takes care of its dependencies
end
```

为什么使用 concern 后变得可行？这就得益于 `append_features` 方法，在方法内部处理了模块引用之间的依赖的关系，现在回过头去看 `append_features` 方法，大概也就明白了为什么需要这么处理了吧。（什么？？还不明白？多看几遍


完。
