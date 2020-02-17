---
title: Sidekiq源码学习
date: 2019-07-31 23:00:03
tags: ruby rails
categories: 后端技术
---

在公司项目里面，我们是用 sidekiq 来做消息队列的，但是都仅仅是停留在用的阶段，前几天有空看了下源码才知道内部结构原来是这么回事

下面我从使用的过程一直走到源码内部来看看整个过程是怎样的，请坐好扶稳

模拟场景：异步发送邮件

```ruby
# 定义发送邮件的 worker
class EmailWorker
  include Sidekiq::Worker
  def perform(email, content)
    Email.send(email, content) # 发送邮件
  end
end

# 调用端
class EmailController
  def welcome
    EmailWorker.perform_async("xxx@xx.com", "welcome to my home")
  end
end
```

定义了 EmailWorker 以后，在 controller 层就可以直接调用 这个 worker，为什么不是 EmailWorker.perform 而是 perform_async？ 因为 perform 是实例方法，而 perform_async 是类方法，所以，真正工作的是 perform_async，OK，那下面来看看 perform_async 方法做了什么处理，需要注意的是，perform_async 所接收的参数是跟 perform 是一致的


### Sidekiq 是如何生产数据的？（生产者）

```ruby
为了排版以下内容只列出关键方法，均有删减
def perform_async(*args)
    # 1. 把类对象和参数组成 hash 后传递给 client_push
    client_push("class" => self, "args" => args)
end

def client_push(item)
    # 2. 实例化一个 client 把 item 参数传递给 push 方法
    Sidekiq::Client.new(pool).push(item)
end
def push(item)
  normed = normalize_item(item) # 格式化 hash
  payload = process_single(item["class"], normed) # 中间件方法，后面插曲有提到

  if payload
    raw_push([payload]) # 关键方法
    payload["jid"]
  end
end

def raw_push(payloads)
  @redis_pool.with do |conn|
    conn.multi do
      atomic_push(conn, payloads)
    end
  end
  true
end

# 最终处理的方法
def atomic_push(conn, payloads)
  # 是否定时任务，如果是定时任务，则按照定时任务的逻辑来处理字符串后放入队列
  if payloads.first["at"]
    conn.zadd("schedule", payloads.map { |hash|
      at = hash.delete("at").to_s
      [at, Sidekiq.dump_json(hash)]
    })
  else
    # 非定时任务，记录入队时间，把它放入 redis 的列表
    queue = payloads.first["queue"]
    now = Time.now.to_f
    to_push = payloads.map { |entry|
      entry["enqueued_at"] = now
      Sidekiq.dump_json(entry)
    }
    conn.sadd("queues", queue)
    conn.lpush("queue:#{queue}", to_push)
  end
end

完。
```

以上，就是 `EmailWorker.perform_async("xxx@xx.com", "welcome to my home")` 所做的处理，其实归根结底，生产者所做的事情都比较简单，**就是把类名和参数格式化一下，然后存储到 redis 的队列里面去**

### 插曲（中间件)
在客户端代码中，process_single 所做的事情就是调用已注册的中间件，具体可参考以下这个[文件](https://github.com/mperham/sidekiq/blob/master/lib/sidekiq/middleware/chain.rb)

基于此中间件，你可以定义在消息塞入队列前后的操作
```
# Sidekiq.configure_client do |config|
#   config.client_middleware do |chain|
#     chain.insert_after ActiveRecord, MyClientHook
#   end
# end

# class MyClientHook
#   def call(worker_class, msg, queue, redis_pool)
#     puts "Before push"
#     result = yield
#     puts "After push"
#     result
#   end
```

### 如何进行消费？（消费者）
消费者比较复杂，但是仔细看代码还是能看明白的，sidekiq的代码真的写得非常棒，基本上不需要怎么看注释，很自然而然的就让下读，读的很舒服。

当在命令行敲击 bundle exec sidekiq 的时候，实际上运行的就是以下 3 行代码
```
cli = Sidekiq::CLI.instance
cli.parse
cli.run 
```

当然了，这三行代码只是一个入口，我们可以慢慢往下看
```ruby
以下内容有删减
# 首先，打开了管道
self_read, self_write = IO.pipe 
# 注册信号量
sigs.each do |sig|
    trap sig do
      self_write.write("#{sig}\n")
    end
end

launcher.run # 主要方法

# 死循环一直读取管道，直到发送了信号量才进行 handle_signal，可参考后面的信号量说明
while (readable_io = IO.select([self_read]))
  signal = readable_io.first[0].gets.strip
  handle_signal(signal)
end
```

注意到第一步，只是简单的进行一个死循环，等待用户给信号量，比如 `kill -USR2 ` 等信息，主要方法在于 `launcher.run`，以下，是 `launcher.run` 方法，也仅仅是做了 3 件事，第一：注册心跳包，定时检测状态；第二：不知道，往下看；第三：也不知道，往下看；

```
def run
  @thread = safe_thread("heartbeat", &method(:start_heartbeat))
  @poller.start
  @manager.start
end
```

沿着 **@poller.start** 的调用栈一直往下走，最后可以发现实际上执行的是以下代码
```ruby
class Enq
  def enqueue_jobs(now = Time.now.to_f.to_s, sorted_sets = SETS)
    # A job's "score" in Redis is the time at which it should be processed.
    # Just check Redis for the set of jobs with a timestamp before now.
    Sidekiq.redis do |conn|
      sorted_sets.each do |sorted_set|
        # Get the next item in the queue if it's score (time to execute) is <= now.
        # We need to go through the list one at a time to reduce the risk of something
        # going wrong between the time jobs are popped from the scheduled queue and when
        # they are pushed onto a work queue and losing the jobs.
        while (job = conn.zrangebyscore(sorted_set, "-inf", now, limit: [0, 1]).first)

          # Pop item off the queue and add it to the work queue. If the job can't be popped from
          # the queue, it's because another process already popped it so we can move on to the
          # next one.
          if conn.zrem(sorted_set, job)
            Sidekiq::Client.push(Sidekiq.load_json(job))
            Sidekiq.logger.debug { "enqueued #{sorted_set}: #{job}" }
          end
        end
      end
    end
  end
end
```
其实也比较好理解，就是排序列表后，取出定时任务，然后放入到队列里面去执行，比如说你设置了今晚 10 点的发送邮件的任务，你在早上8点就已经调用了并且启动了 sidekiq，那么 sidekiq 会检测定时任务的列表，直到晚上 10 点，sidekiq 会把这个任务的相关信息取出来，然后发送到处理任务的队列里面去，也就是 `Sidekiq::Client.push(Sidekiq.load_json(job))`

OK，看完 poller.start 方法的处理逻辑以后，可以继续往下看 **@manager.start** 方法，是的，这个就是真正进行消费的代码逻辑
```ruby
def initialize(options = {})
  logger.debug { options.inspect }
  @options = options
  @count = options[:concurrency] || 10
  @done = false
  @workers = Set.new
  # 根据 count 的设置，来决定实例化多少个 `Processor` 对象，默认是 10 个
  @count.times do
    @workers << Processor.new(self)
  end
  @plock = Mutex.new
end

# Processor.new(self) 执行 options[:concurrency] 次 start
def start
  @workers.each do |x|
    x.start
  end
end

# 继续进入 Processor 类的 start 方法看

def start
  @thread ||= safe_thread("processor", &method(:run))
end

def run
  process_one until @done
  @mgr.processor_stopped(self)
rescue Sidekiq::Shutdown
  @mgr.processor_stopped(self)
rescue Exception => ex
  @mgr.processor_died(self, ex)
end
```

**看到 start 方法大概就已经明白了，其实就是起 N 个线程来跑 run 方法，而继续往下看 run 方法实际上就是不断的从队列里面阻塞的取数据，下面就是取数据的关键方法**

```
def retrieve_work
  work = Sidekiq.redis { |conn| conn.brpop(*queues_cmd) }
  UnitOfWork.new(*work) if work
end
```

### 消费者的大概流程

![image](/images/image.png)

### 存在的问题
1. sidekiq是能保证顺序的，但是因为从队列里面取数据的时候是阻塞的取的，所以造成了尽管有 20 个线程，某个线程从 redis 取数据的时候，其它 19 个线程是处于等待的状态的，并不能实现完美的并发消费。
2. 基于 redis 的队列使得结构较为单一，意思就是队列只有一个，但是线程太多了，无法同时进行处理。不过，sidekiq 里面可以设置不同的队列名称，使得可以并发的执行不同的队列名
3. 如何保证消息的可靠性？因为线程从队列里面拿出来以后，这条消息就相当于被消费了，那么如果线程拿出来后就死掉的话，这条消息是不是就丢了呢？

### 信号量（插曲）
在看到处理服务端的的时候，有一段代码是关于处理信号量的
```
sigs = %w[INT TERM TTIN TSTP]
... 
sigs.each do |sig|
trap sig do
  self_write.write("#{sig}\n")
end
rescue ArgumentError
  puts "Signal #{sig} not supported"
end
```

这段代码是为了发出信号的时候，通知管道的另一端处理，那么信号量是怎么一回事？ 看看下面代码

```
puts "I have PID #{Process.pid}"

Signal.trap("USR1") {puts "prodded me"}

loop do
  sleep 5
  puts "doing stuff"
end

```

首先，可以先 `ruby example.rb` ，然后呢，怎么触发信号量？ 通过 `ps -ef | grep ` 找到对应的进程，再次执行 `kill -USR1 xxx`，就会触发 `puts prodded me`

[解释信号的一篇文章](https://ddl1st.iteye.com/blog/1772049)


### 小 Demo 测试队列的示例代码
```ruby
require 'sidekiq'

Sidekiq.configure_client do |config|
  config.redis = { db: 1 }
end

Sidekiq.configure_server do |config|
  config.redis = { db: 1 }
end

Sidekiq.configure_client do |config|
  class MyClientHook
    def call(worker_class, msg, queue, redis_pool)
      puts "Before push"
      result = yield
      puts "After push"
      puts result
      result
    end
  end
  config.client_middleware do |chain|
    chain.add MyClientHook
  end
end

class OurWorker
  include Sidekiq::Worker

  def perform(msg)
    puts "hello #{msg}"
  end
end
# 启动服务端
# bundle exec sidekiq -r ./woker.rb
# 启动客户端
# bundle exec irb -r ./woker.rb
# OurWorker.perform_async("abc") # 生产一条数据
# 输出： {"class"=>"OurWorker", "args"=>["abc"], "retry"=>true, "queue"=>"default", "jid"=>"966a57511c1d2b8d314b1318", "created_at"=>1563630849.305185}
```
