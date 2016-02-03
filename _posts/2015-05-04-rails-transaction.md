---
layout: post
title: Rails中的事务处理
date: 2015-05-04 18:00:00
disqus: y
---

<!--
create time: 2016-02-03 10:03:54
Author: amoblin

This file is created by Marboo<http://marboo.io> template file $MARBOO_HOME/.media/starts/default.md
本文件由 Marboo<http://marboo.io> 模板文件 $MARBOO_HOME/.media/starts/default.md 创建
-->
翻译自：http://markdaggett.com/blog/2011/12/01/transactions-in-rails/

发现大鱼团队中不少同学对Rails 中事务的使用不当，发现这篇文章不错，花一点时间翻译了一下，希望对各位有用~

## 使用事务的原因
事务用来确保多条SQL语句要么全部执行成功、要么不执行。事务可以帮助开发者保证应用中的数据一致性。常见的使用事务的场景是银行转账，钱从一个账户转移到另外一个账户。如果中间的某一步出错，那么整个过程应该重置。这个例子的伪码如下：

```ruby
ActiveRecord::Base.transaction do
  david.withdrawal(100)
  mary.deposit(100)
end
```

Rails中，通过ActiveRecord对象的类方法或者实例方法即可实现事务：

```ruby
Client.transaction do
  @client.users.create!
  @user.clients(true).first.destroy!
  Product.first.destroy!
end

@client.transaction do
  @client.users.create!
  @user.clients(true).first.destroy!
  Product.first.destroy!
end
```

可以看到上面的例子中，每个事务中均含有多个不同的 model 。在同一个事务中调用多个 model 对象是常见的行为，因为事务是和一个数据库连接绑定在一起的，而不是某个 model 对象；而同时，也只有在对多个纪录进行操作，并且希望这些操作作为一个整体的时候，事务才是必要的。

另外，Rails 已经把类似 #save 和 #destroy 的方法包含在一个事务中了，因此，对于单条数据库记录来说，不需要再使用显式的调用了。

## 触发事务回滚
事务通过 rollback 过程把记录的状态进行重置。在 Rails 中，rollback 只会被一个 exception 触发。这是非常关键的一点，很多事务块中的代码不会触发异常，因此即使出错，事务也不会回滚。比如下面的写法：

```ruby
ActiveRecord::Base.transaction do
  david.update_attribute(:amount, david.amount -100)
  mary.update_attribute(:amount, 100)
end
```

因为 Rails 中，#update_attribute 方法在调用失败的时候也不会触发 exception，它只是简单的返回 false ，因此必须确保 transaction 块中的函数在失败时会抛异常。正确的写法是这样的：

```ruby
ActiveRecord::Base.transaction do
  david.update_attributes!(:amount => -100)
  mary.update_attributes!(:amount => 100)
end
```

注意，Rails 中约定，带有叹号的函数一般会在失败时抛异常。


同时，我也看到一些代码中，在事务块中使用了 #find_by 方法，实际上，find_by 等魔术方法当找不到记录的时候会返回 nil，而 #find_ 方法在找不到记录的时候会抛出一个 ActiveRecord::RecordNotFound 异常。比如下面的例子：

```ruby
ActiveRecord::Base.transaction do
  david = User.find_by_name("david")
  if(david.id != john.id)
    john.update_attributes!(:amount => -100)
    mary.update_attributes!(:amount => 100)
  end
end
```

发现上面的逻辑错误了吗？nil 对象也有一个 id 方法，导致记录没有被找到的错误被隐藏了。同时，由于 find_by 也不会抛出异常，因此下面的代码被错误的执行了。这就意味着，有的时候在某些场景下，我们需要人工抛异常。因此这段代码因此改成下面的形式:

```ruby
ActiveRecord::Base.transaction do
  david = User.find_by_name("david")
  raise ActiveRecord::RecordNotFound if david.nil?
  if(david.id != john.id)
    john.update_attributes!(:amount => -100)
    mary.update_attributes!(:amount => 100)
  end
end
```

当错误出现时，事务本身会回滚，同时异常也会在外层抛出。因此，你的调用方必须考虑 catch 这个异常，并进行相应的处理。

有一个特殊的异常，ActiveRecord::Rollback，当它被抛出时，事务本身会回滚，但是它并不会被重新抛出，因此你也不需要在外部进行 catch 和处理。

## 何时使用嵌套事务？
错误使用或者过多使用嵌套异常是比较常见的错误。当你把一个 transaction 嵌套在另外一个事务之中时，就会存在父事务和子事务，这种写法有时候会导致奇怪的结果。比如下面来自于 Rails API 文档的例子：

```ruby
User.transaction do
  User.create(:username => 'Kotori')
  User.transaction do
    User.create(:username => 'Nemu')
    raise ActiveRecord::Rollback
  end
end
```

上面提到，ActiveRecord::Rollback 不会传播到上层的方法中去，因此这个例子中，父事务并不会收到子事务抛出的异常。因为子事务块中的内容也被合并到了父事务中去，因此这个例子中，两条 User 记录都会被创建！

可以把嵌套事务这样理解，子事务中的内容被归并到了父事务中，这样子事务变空。

为了保证一个子事务的 rollback 被父事务知晓，必须手动在子事务中添加 :require_new => true 选项。比如下面的写法：

```ruby
User.transaction do
  User.create(:username => 'Kotori')
  User.transaction(:requires_new => true) do
    User.create(:username => 'Nemu')
    raise ActiveRecord::Rollback
  end
end
```

事务是跟当前的数据库连接绑定的，因此，如果你的应用同时向多个数据库进行写操作，那么必须把代码包裹在一个嵌套事务中去。比如：

```ruby
Client.transaction do
  Product.transaction do
    product.buy(@quantity)
    client.update_attributes!(:sales_count => @sales_count + 1)
  end
end
```

## 事务相关的回调
上面提到 #save 和 #destroy 方法被自动包裹在一个事务中，因此相关的回调，比如 #after_save 仍然属于事务的一部分，因此回调代码也有可能被回滚。

因此，如果你希望代码在事务外部执行的话，那么可以使用 #after_commit 或者 # after_rollback 这样的回调函数。

## 事务陷阱
不要在事务内部去捕捉 ActiveRecord::RecordInvalid 异常。因为某些数据库下，这个异常会导致事务失效，比如 Postgres。一旦事务失效，要想让代码正确工作，就必须从头重新执行事务。

另外，测试回滚或者事务回滚相关的回调时，最好关掉 transactional_fixtures 选项，一般的测试框架中，这个选项是打开的。

## 常见的事务反模式
1. 单条记录操作时使用事务
2. 不必要的使用嵌套式事务
3. 事务中的代码不会导致回滚
4. 在 controller 中使用事务
