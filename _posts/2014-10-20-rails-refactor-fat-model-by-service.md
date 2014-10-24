---
layout: post
title: Rails最佳实践: 利用Service优化Fat Model
date: 2014-10-20 18:00:00
disqus: y
---


# Rails最佳实践: 利用Service优化Fat Model


本文主要说明什么时候需要重构fat model，以及通过一个简单的案例来讲解如何一步步重构复杂的model，把它变成扩展性良好的service。

## 源起

在这篇文章 [7 Patterns to Refactor Fat ActiveRecord Models](http://blog.codeclimate.com/blog/2012/10/17/7-ways-to-decompose-fat-activerecord-models/) 中，其中提到了一点，利用Service重构 “Fat” Model。

原文中提到了几点需要重构成service的场景：

1. 功能逻辑比较复杂
2. 功能涉及到了多个model的时候
3. 此功能会和外部系统交互
4. 此功能并非底层model的核心责任，关联不大
5. 此功能可能会有多种实现方式

上面的总结很好，但是也很模糊，比如到底什么样的功能算是复杂？涉及到多个model就应该优化成service吗？怎么样才叫做关联不大？

每个人的判断可能不太一样，对于一个team来讲，需要一个相对比较明确的约定来定义什么时候，你的业务逻辑需要重构层一个service了。目前大鱼的开发团队是这么简单约定的：

* 当model class中出现跨model的‘写’行为的时候


为什么是这样的约定？

因为一般出现了跨model的写的时候，说明你的业务逻辑比较复杂了，可以考虑封装成一个service来完成这件相对“独立”的功能；特别是如果model的回调中，出现了跨model的写，这种情况更应该避免，因为将来逻辑复杂时，很有可能回调的条件已不再满足了。

所以service的封装的粒度应该是比较明确的，那就是对于复杂的功能逻辑，如果同时又比较独立，将来有一定的可能性会扩展成一个engine、甚至独立的组件（如http api），那么显然是应该封装层service的，那么目前的一个“较好”的标准，就是如果model内部出现了跨model的“写”，应当考虑把这部分功能封装层service，以便将来的扩展。


## 问题场景
案例实现的是电商中常见的“常用联系人”的功能，下面是一个简化的需求说明：

* 用户提交订单的时候，需要记录本订单使用的联系人姓名、电话、邮箱；每个订单需要存储一份；
* 订单提交后，系统需要把联系人信息记录在‘常用联系人’表中，供下次用户快捷填写；每个用户有多个’常用联系人‘
* 每个用户都有唯一的一份联系信息，每次订单提交后需要更新此信息；此信息会用在其他系统作为用户的标志；
* 常用联系人、订单联系人之间利用真实姓名进行弱关联，同一个名字认为是同一个用户

## 实现以及重构过程
### 基本的表结构
OrderContact 订单联系人表：跟订单是 1:1 的

```
order_id: integer
real_name: string
cellphone: string
email: string
***: 其他字段忽略
```

UserContact 常用旅客表：跟 User 是 N:1 的

````
user_id: integer
real_name: string
cellphone: string
email: string
***: 其他字段忽略
````

UserProfile 用户基本信息表， 跟User是 1:1 的

```
user_id: integer
real_name: string
cellphone: string
email: string
****: 其他字段忽略
```

如果是你来实现这个需求，你会怎么写？hoho，请继续看下去吧！

```
# 最基本的几个关联关系
class User
  has_many :user_contacts
  has_one :user_profile
end
class Order
  has_one :order_contact
end
class UserContact
  belongs_to :user
end
class OrderContact
  belongs_to :order
end
class UserProfile
  belongs_to :user
end
```

### 第一次，常见Rails的写法

常见的rails的写法是，在 order_contact model层添加after_save回调，分别去更新对应的user_contact和user_profile即可，写起来也会很快捷；

```
class OrderContact < ActiveRecord::Base
  belongs_to :order
  
  after_save :update_user_contact
  after_save :update_user_profile

  private
  def update_user_contact
    user_contact = order.user.user_contacts.by_real_name(real_name).first_or_initialize
    
    user_contact.email = self.email
    user_contact.cellphone = self.cellphone
    user_contact.real_name = self.real_name
    user_contact.save
  end
  def update_user_profile
    user_profile = order.user.user_profile
    
    user_profile.email = self.email
    user_profile.cellphone = self.cellphone
    user_profile.real_name = self.real_name
    user_profile.save
  end
end

class OrderController < ApplicationController
  def create
    # 创建订单的时候保存联系人信息
    @order = Order.create(params[:order])
    @order_contact = @order.create_order_contact(params[:order_contact])
  end
end
```

这样的写法有两个问题：

* 从当前的逻辑来看，所有的order_contact更新的时候都必须更新另外两个model，可是可能马上需求就要变化。这种利用callback的写法，当需求变化的时候再改动就会比较困难，这个时候负责新功能的工程师需要理清楚原有的思路，并且必须陷入到ActiveRecord类中去；
* 例子中的UserContact类和UserProfile类，可能很快也会变化，这个时候直接在 order_contact类 中调用它们的 attribute=() 方法就显得很不合适了；至少这些类需要提供一个写接口，这样才能应对变化；

好，接下来就把上面两个缺点给重构掉：


### 重构一下，去掉回调
基本的策略是把 after_save的回调，方法是在controller里面调用相关的方法了；然后我们要去掉直接在model里面去写另外一个model的逻辑，方法是让它提供相应的封装好的写接口；

提供封装好的写接口:

```
class OrderContact
  # 删除原有的 after_save 以及相关的方法
end
class UserContact
  def self.update_by_order_contact(user, order_contact)
    user_contact = user.user_contacts.by_real_name(real_name).first_or_initialize

    user_contact.real_name = order_contact.real_name
    user_contact.cellphone = order_contact.cellphone 
    user_contact.email = order_contact.email

    user_contact.save
  end
end

class UserProfile
  def self.update_by_order_contact(user, order_contact)
    user_profile = user.profile

    user_profile.real_name = order_contact.real_name
    user_profile.cellphone = order_contact.cellphone 
    user_profile.email = order_contact.email

    user_profile.save
  end
end

```

然后在controller里面直接调用
```
class OrderController < ApplicationController
  def create
    # 创建订单的时候保存联系人信息
    @order = Order.create(params[:order])
    @order_contact = @order.create_order_contact(params[:order_contact])

    # 调用写接口，更新user相关的信息
    UserProfile.update_by_order_contact(@order.user, @order_contact)
    UserContact.update_by_order_contact(@order.user, @order_contact)
  end
end
```

上面的代码利用类函数的方法把“写”代码移到了正确的类中，代码看起来清晰了一些，但是好像复杂性并没有降低，反而有点冗余了：

* 在controller中原来是自动调用，现在需要写独立的代码，以后将来又有了新的类，不只是UserProfile和UserContact呢？就得在很多地方多添加一行，比如新的类是 UserInfo，那在每个controller里面都必须都写一行；
* 静态函数里面其实隐含了一个需求，那就是更新常用联系人是根据 真实姓名 来更新的, 在这一行里面提现：

```
  user_contact = user.user_contacts.by_real_name(real_name).first_or_initialize
```

其实这个就是典型的业务逻辑了，显然也不应该放藏的这么深，将来也很难去维护。

那么，有没有更好的办法呢？

### Service出场，封装成service
利用service的方法，我们把所有的业务逻辑抽离出来，把数据逻辑继续留在model层中；并且是把“更新用户信息”当做一个独立的小模块来实现, 而现在这个service只提供一个接口，那就是根据 order_contact 来更新用户信息。

从这个角度看问题，我们创建UserInfoService, 并且它的职责以及范围就很清楚了，继续往下改进：

model层改成这个样子：

```
class UserContact < ActiveRecord::Base
  def update_by_order_contact(order_contact)
    self.real_name = order_contact.real_name
    self.cellphone = order_contact.cellphone 
    self.email = order_contact.email
    self.save
  end
end

class UserProfile < ActiveRecord::Base
  def update_by_order_contact(order_contact)
    self.real_name = order_contact.real_name
    self.cellphone = order_contact.cellphone 
    self.email = order_contact.email
    self.save
  end
end
```

新的UserInfoService只是一个简单的ruby类：

```
class UserInfoService
  def initialize(user)
    @user = user
  end

  def refresh_from_order_contact(order_contact)
    # 更新常用联系人
    user_contact = find_user_contact(order_contact.real_name)
    user_contact.update_by_order_contact(order_contact)

    # 更新用户个人信息
    @user.profile.update_by_order_contact(order_contact)
  end

  private
  def find_user_contact(real_name)
    @user.user_contacts.by_real_name(real_name).first_or_initialize
  end
end
```

新的控制器中代码的写法: 

```
class OrderController < ApplicationController
  def create
    # 创建订单的时候保存联系人信息
    @order = Order.create(params[:order])
    @order_contact = @order.create_order_contact(params[:order_contact])

    # 调用写接口，更新user相关的信息
    UserInfoService.new(@order.user).refresh_by_order_contact(@order_contact)
  end
end
```

经过上面的改动，有没有感觉代码更清晰呢？这些写有下面几个好处：

1. 把更新用户信息这个逻辑抽离，service本身是可复用的，可以用在任何地方，包括task；console的调试等；
1. service的接口很明确，就是根据order_contact更新用户的信息；如果以后有了新的需求，我们可以添加新的接口；如果原有的需求发生了变化，也可以修改目前的方法；都是很简单的。


## 总结
总结一下什么时候应该抽取 service:

1. 当发生跨model的写的时候。这不是必然，但是可以认为是一个信号，表示你的业务逻辑开始变的复杂了。同时，当跨model的“写”都遵守了这个规则时，rails的model层就会变成一个真正的 DAL（Data Access Layer），不再是混合了数据逻辑和业务逻辑的 “Fat Model”；
1. 一般来讲，callback 是要慎用的，特别是 callback 里面涉及到了调用其他 model 、修改其他 model 的情况，这个时候就可以考虑把相关的逻辑抽成 service 。

其他像文章最初提到的一些规则都比较模糊，需要经验丰富的工程师才能比较明确的判断，比如业务逻辑比较复杂、相对独立、将来可能会被升级成独立的模块的时候，这些需要一定的经验积累才比较容易判断。

service 的好处，基本上是抽象层独立的类之后的好处：

1. 复用性比较好。因为是 ruby plain object，所以复用性上很简单，可以用在各种地方；
2. 比较独立，可扩展性比较好。可以扩展 service ，给它添加新的方法、修改原有的行为均可；
3. 可测试性也会较好。

抽取service 的本质是要把数据逻辑层和业务逻辑区别对待，让 model 层稍微轻一些；Rails 里面有 view logic、data logic、domain logic，把它们区别对待是最基本的，这样才能写出更清晰、可维护的大型应用来。


大概就是这些了，以上。



