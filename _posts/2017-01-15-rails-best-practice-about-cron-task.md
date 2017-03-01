---
layout: post
title: Rails最佳实践-定时任务
date: 2017-01-15 12:20:00
disqus: y
---

开发Rails项目中难免遇到一些需要做定时任务的情况，比如每天晚上去跑一些简单的统计，定时更新一下缓存等等情况，虽然是一个简单的事情，可是随着时间的推动，一个项目中的定时任务可能会很多，比如目前我们有一个项目，定时任务有上百条，已经非常的难以管理和容易出问题了。因此，本文简单总结了一些关注点帮助大家理解和规范定时任务的写法。

先简单介绍一下定时任务，定时任务是指那些周期性执行或者某些时刻固定执行的任务，一般来讲大家都熟悉Linux系统的crontab配置文件，这里面就是常见的定时任务了。Rails环境下，最简单的定时任务直接利用rake或者rails runner执行一个命令，然后把命令放到crontab配置文件中即可。

对于那些一次性执行的任务，大家都知道应该使用Job系统来完成，但是有时候有那种延期执行的任务，这个我个人觉得也不是定时任务应该负责的范畴，一般的Job系统，比如delayed job、sidekiq等也都提供延时执行的功能，直接使用这些就可以了。所以本文主要指那种周期性执行的任务。

## 定时任务的职责
就跟一个Class、一个method一样，先确定职责是比较重要的一件事。那么定时任务系统的职责是什么呢？

最初级的做法是直接在 rake task里面写一通逻辑，直接把任务的主逻辑放在这里。这样的话有一个最明显的问题是难以测试，大家可以google一下相关的方案，可以解决，但是较繁琐。

稍好一点的做法是把业务逻辑封装在一个 Service 里面，然后在 task 里面去调用这个 Service。至少这样的话可以解决掉测试的问题。但是这样做还是有一个问题，就是错误处理的问题。一般 task 中的业务逻辑还相对比较复杂，当这个 task 出错了怎么办呢？比如使用 crontab 来执行某个任务，任务出错时一般可能就是记录日志，再根据错误日志做个报警而已。可是很多时候任务的错误处理需要更及时、同时也属于业务逻辑的一部分，由报警系统处理再反馈给业务系统显然很不合理。

所以，目前社区内比较推荐的做法是把定时任务作为一个调度器（scheduler）而存在，定时任务只是一个调度器，真正的业务逻辑都是封装在Job中。比如下面这种方式：

```ruby
# lib/tasks/some_task.rake
task some_task: :environment do
	SomeJob.perform_async # 调用 Sidekiq 的 Job 来异步执行
end
```

这样的话整个定时任务的代码非常简单，只是充当一个调度器（scheduler）的功能，同时通过使用异步任务的形式还获得了下面的好处：

1. 任务有重试机制。一般的Job都可以很简单的配置重试机制，保证了最终Job一定会被成功执行。即使有bug，修复上线之后无需维护，下次重试的时候就可以正常完成任务，非常方便。
2. 任务有更实时的错误处理机制。比如大鱼内部有一种Job，在多次重试失败之后需要更改某个 ActiveRecord 的状态为 fail。这种情况通过 Job 的形式就非常方便了。

   比如 Job 大概长这样（拿 Sidekiq 举例）：

```ruby
# app/jobs/some_job.rb

class SomeJob
  include Sidekiq::Workder
	
  sidekiq_retries_exhausted do |job, e|
    process_failure_job(job, e)
  end
  
  def perform
    # 正常业务逻辑
  end
  
  def process_failure(job, error)
    # 多次重试失败后的处理逻辑
  end
end
```

总之，定时任务书写的时候应该是轻量级的，最好是与 Job 系统联合使用，定时任务只是作为一个调度器，Job 系统来真正执行业务逻辑。

## 定时任务的部署

rails社区中大家使用最多的部署方案就是 whenever + linux crontab 方案了，这种方案简单来说也没太问题，简单易用。但是随着项目的复杂度增加，我不太推荐这种部署方案，主要基于以下理由：

1. crontab 方案每一条任务都是独立的，都需要完整加载整个Rails运行环境，而Rails运行环境相对是比较耗资源的。想象一下每次几十个、上百个定时任务不停的启动停止，系统的负载可想而知。
2. crontab 本身是系统的组件，这种方案需要依赖于系统的组件，在很多 production 环境下可能没有 crontab 组件，比如运行在 heroku 上，比如运行在 docker 上。

一个web应用也期望都以[一个或多个无状态的进程来运行](https://12factor.net/zh_cn/processes)的形式来运行，从这个角度上来讲，也应该以一个独立进程来运行 cron 任务，而不是依赖于系统的cron组件。

那么有没有好的方案呢，其实方案有挺多的，都可以解决问题，比如：

* [rufus-scheduler](https://github.com/jmettraux/rufus-scheduler)
* [crono](https://github.com/plashchynski/crono)

还有一类是 Job 系统的插件形式，但是也可以以独立进程来运行，我没有实践过，应该也差不多：

* [sidekiq-scheduler](https://github.com/moove-it/sidekiq-scheduler)
* [resque-scheduler](https://github.com/resque/resque-scheduler)

下面以 crono 来例简单示例一下定时任务的写法，也非常简单。

```ruby
# Gemfile
gem 'crono', '~> 1.1', '>= 1.1.2'

# config/cronotab.rb
class HourlyPerformSomething
	def perform
		SomeModel.pending.each {|s| SomeJob.perform_async(s.id) }
	end
end

Crono.perform(HourlyPerformSomething).every 1.hour
```

执行时：

```bash
bundle exec crono RAILS_ENV=production
```

## 总结

简单总结下就是下面的2条吧：

1. 优先把定时任务当做调度器使用，主要业务逻辑通过 Job 系统来完成。特别是不要在rake任务中写执行大量的业务逻辑，会造成难以测试、无法重试和错误处理的问题。
2. 优先使用独立的进程来管理定时任务，而不是依赖于 Linux 系统的 crontab。这样的好处是更具有移植性、更易运维。

欢迎讨论~
  
  

