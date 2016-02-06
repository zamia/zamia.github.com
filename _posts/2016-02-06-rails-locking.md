---
layout: post
title: Rails 中乐观锁与悲观锁的使用
date: 2016-02-03 18:00:00
disqus: y
---

## 乐观锁和悲观锁的简介

只有有资源的争用就少不了使用各种锁，包括关系数据库中使用的悲观锁和乐观锁，分布式系统中的分布式锁（比如使用 zoo keeper 或者 redis 等实现），MRI ruby 中也存在 GIL(global intepreter lock)，mongodb 中也存在全局锁、database-level 锁和 collection-level 锁等等。

本文主要讲我们日常开发中很大可能会用到的两种锁：

* 悲观锁。悲观锁采用相对保守的策略，在资源争用比较严重的时候比较合适。悲观锁在事务开始之前就去尝试获得写权限，事务结束后释放锁；也就是说对于同一行记录，只有一个写事务可以并行；
* 乐观锁。乐观锁是在提交事务之前，大家可以各自修改数据，但是在提交事务的时候，如果发现在这个过程中，数据发生了改变，那么直接拒绝此次事务提交。乐观锁适合在资源争用不激烈的时候使用。

Rails 提供了很好用的 API 来帮助开发者分别去使用这两种锁，写起来很简单（写完之后我都怀疑写这篇文章是否有必要了，大家随意看看），不过对于一些新手同学可能有帮助。

## 悲观锁
### 常用场景
一般对于资源的争用都可以使用悲观锁，比如电商系统中涉及到订单的部分，比如用户支付完成后可能会同时有多条支付成功的通知（做过支付的都知道一般有同步通知和异步通知），比如订单改价的同时可能用户正在支付等等，对于这种会对订单状态发生改变的操作，我们内部一般对这种操作都做加锁处理。

### 使用

rails 的 [API 文档](http://api.rubyonrails.org/classes/ActiveRecord/Locking/Pessimistic.html)中有详细的说明：

```ruby
# select * from accounts where id=1 for update
Account.lock.find(1)
# 注意，这种最终会导致一个行锁

# select * from accounts where name = 'shugo' limit 1 for update
Account.where("name = 'shugo'").lock(true).first
# 注意，这里可不是行锁，这里会是一个表锁
```

注意上面的区别，mysql innodb 里面，对于 "select * from where xxx for update" 的情况，是会锁住整张表的，所以最好不要这样来用。Rails 也提供了一个很方便的方法 with_lock 来锁住单个记录，并且内嵌在事务之中。下面代码中的两段是等价的：

```ruby

account = Account.find(1)
Account.transaction do
    account.lock!
    account.balance -= 100
    account.save! 
end

# 和下面是等价的

account.with_lock do
    account.balance -= 100
    account.save!
end
```

## 乐观锁
### 常用场景
悲观锁出错概率小，因为一旦获得锁，其他进程会堵塞，但是也导致速度会受影响，系统开销比较大，不利于并发。乐观锁适用于资源竞争不是那么多的地方，这样系统的开销较小，速度也比较快。

乐观锁本质上算是一个利用多版本管理来控制并发的技术，如果事务提交之后，数据库发现写入进程传入的版本号与目前数据库中的版本号不一致，说明有其他人已经修改过数据，不再允许本事务的提交。所以，使用乐观锁之前需要给数据库增加一列 :lock_version，Rails 会自动识别这一列，像数据库提交数据的时候自动带上。另外，乐观锁是默认打开的，如果要关闭，需要配置一下。

在大鱼系统中，库存管理是使用乐观锁的，我们的流量没那么大，不太可能多个用户同时预订同一个住宿的同一个间夜，概率比较小，所以目前是使用乐观锁来实现的。如果抛异常，那么还可以进行重试。

### 使用
记得使用前添加 lock_version 的字段给相应的表，其他的就是自动的了，如果事务提交失败，那么 Rails 会抛一个 ActiveRecord::StaleObjectError 的异常。

比如，下面这段代码会进行重试：

```ruby
retry_times = 3

begin
    @order.with_lock do
        @order.set_paid!
    end
rescue ActiveRecord::StaleObjectError => e
    retry_times -= 1
    if retry_times > 0
        retry
    else
        raise e
    end
rescue => e
    raise e
end
```

## 需要注意的地方

1. 一般，使用锁的时候和事务同时使用，所以 with_lock 是用的比较多的，而且尽量使用行锁而不是表锁。
2. 另外，也注意异常的处理，需要使用那些会抛异常的方法；
3. 对于乐观锁，还需要注意如果是前端操作频繁，那么还需要把 lock_version 写入到 form 表单中，否则起不到锁的作用，这里讲的很详细了：<https://blog.engineyard.com/2011/a-guide-to-optimistic-locking>


以上~ （发现这篇没什么内容，不过我记得最初大家写的时候也经常犯错，算是一个总结吧）

## 参考
1. <http://railscasts.com/episodes/59-optimistic-locking-revised>
2. <https://blog.engineyard.com/2011/a-guide-to-optimistic-locking>
3. <http://api.rubyonrails.org/classes/ActiveRecord/Locking/Optimistic.html>



