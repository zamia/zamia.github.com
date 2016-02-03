---
layout: post
title: Rails路由-解决多子域名问题
date: 2015-11-20 18:00:00
disqus: y
---

## rails多子域名问题

Rails项目发展到较大规模的时候、或者为了其他各种原因，一定会遇到多子域名的问题。目前网上的很多资料只是简单的介绍了利用constraints进行操作的方法，并没有系统的解决多子域名实操的时候会遇到的各种问题。比如：

1. 多个子域名下，路由和控制器的设计？
2. 多个routes文件如何拆分？
3. url helper使用的时候的注意事项？

这篇文章算是对上述问题进行的一个较深入的总结和实操，请阅读之前需要Rails路由先有个大概的了解，期望大家读完之后对Rails路由理解的更加深入。

### 多子域名问题解决
### constraint是基础
Rails提供了constraints方法来对一组路由进行限制，比如官方文档（Rails 3.2）中提供的例子：

```ruby
constraints(:id => /\d+\.\d+/) do
  resources :orders
end
```

上面的例子中只有 /posts/123.456 这样的id是允许的，而 "/posts/123" 就是无效的。这也是 constraint（限制）的意思。

那么，根据这个思路，我们可以限制一组路由只在某个子域名下生效，也就达到了多子域名的目的：

```ruby
constraints :subdomain => "m" do
  resources :orders # mobile下
end
constraints :subdomain => "www" do
  resources :orders # pc下
end
```

另外，不要在routes文件里面hard code一些配置，因为一般不同的环境的子域名可能不一样，比如你的测试环境可能需要 alpha.m.example.com 这样的域名，而正式环境才是 m.example.com 这样的域名。我们稍微修改的好一点（示例代码也要写好啊，否则新手容易跟着画瓢）：


config/environments/development.rb中config段添加：

```ruby
config.mobile_subdomain = "m"
config.main_subdomain = "www"
```

config/routes.rb中添加：

```ruby
constraints :subdomain => Rails.configuration.mobile_subdomain do
  resources :orders
end
constraints :subdomain => Rails.configuration.main_subdomain do
  resources :orders
  # 下面是其他路由，比如
  resources :users
end
```

但是这样显然是不work的（手册跟实际工作是有差距的），因为constraints只是做了『限制』，路由指向的controller并不会改变，也就是说上面的mobile子域名中的orders也是同样指向了 ::OrdersController 这个控制器，实际工作中，我们一般是期望 m.example.com/orders 路由指向 ::Mobile::USersController 这个控制器的。

那应该如何解决呢？这个时候就需要使用scope了。

### 先提下namespace
scope日常不太常使用，因为大家一般都是使用namespace就够了，比如最常见的写法：

```ruby
namespace :admin do
  resources :orders
end
```

这样会把 /admin/orders 路由指向 ::Admin::PostsController，很方便吧？

### scope来解决

但是在子域名环境下，这样是达不到目的的。因为我们期望 "m.example.com/orders" 能指向 ::Mobile::OrdersController，那么该如何搞呢，这个时候就需要scope了。scope提供了比较namepace更细粒度的控制参数，完全可以满足我们的需求。

下面的代码来自于官方文档，稍微翻译了下：

```ruby
### 把 url "/posts" (不包含/admin前缀) 指向Admin::PostsController
scope :module => "admin" do
  resources :posts
end 

### 把 posts相关路由添加 "/admin/" 前缀
scope :path => "/admin" do
  resources :posts
end

### 修改 url helper，用 +sekret_posts_path+ 替代 +posts_path+
scope :as => "sekret" do
  resources :posts
end
```

所以，这三个参数是可以独立使用的，那么上面提到的子域名的问题的解决方案也就有了：

```ruby
constraints :subdomain => Rails.configuration.mobile_subdomain do
  scope module: 'mobile' do 
    resources :orders
  end
end
constraints :subdomain => Rails.configuration.main_subdomain do
  resources :orders
end
```

这样的话，mobile子域名下的/orders会路由到 "::Mobile::OrdersController"，目标达成！

### 优化一下
这样好像还不够好，这样写有一个小小的问题，就是你在mobile下面引用一个url helper的时候，比如：

```ruby
### app/mobile/orders/show.html.erb
<%= link_to order_path(@order), order.id %>
```

读代码的人比较难以直观的知道这个 order_path 是指的哪个域名下的url，而且如果多个子域名下有url路径重复的话，一旦写错，rails不会提示错误，只有访问的时候才会报错。所以，最好给scope加一个as参数，把mobile下的url helper独立出来：

```ruby
### 添加as参数会修改url helper
constraints :subdomain => Rails.configuration.mobile_subdomain do
  scope module: 'mobile', as: 'mobile' do 
    resources :orders
  end
end

### 调用的时候
<%= link_to mobile_order_path(@order), order.id %>
```

### namespace 和 scope 其实是一个东西
其实事实上，看rails源码可以看到，namespace只不过是包装了一层，底层完全是用scope来实现的：

```ruby
### File actionpack/lib/action_dispatch/routing/mapper.rb, line 679
def namespace(path, options = {})
  path = path.to_s
  options = { :path => path, :as => path, :module => path,
              :shallow_path => path, :shallow_prefix => path }.merge!(options)
  scope(options) { yield }
end
```

子域名的问题到这里应该已经基本说清楚了，下面讲讲其他容易遇到的问题。

## rails routes文件拆分
一旦你有了多个子域名，可能你的routes文件开始变大很难维护了，这个时候把routes文件拆分是一个好主意。

routes文件拆分有两个办法，一种是rails内建的，但是Rails4已经移除了，一种是不受rails版本影响的方法。

[这篇英文的文章](http://blog.arkency.com/2015/02/how-to-split-routes-dot-rb-into-smaller-parts/)写得挺清楚了，下面简单说一下。


### 修改rails路径配置参数

通过修改 "config/routes" 配置来解决：

```ruby
config.paths["config/routes"] += Dir[Rails.root.join('config', 'routes','*.rb')]
```

如果对加载顺序有依赖（最好别依赖），可以一个文件一个文件的加载:

```ruby
config.paths["config/routes"] = %w(
      config/routes/admin.rb
      config/routes/api.rb
      config/routes.rb
    ).map { |p| Rails.root.join(p) }
```

### 利用instance_eval

利用instance_eval来加载其他路由文件即可：

```ruby
Example::Application.routes.draw do
  def draw(routes_name)
    instance_eval(File.read(Rails.root.join("config", "routes", "#{routes_name}.rb")))
  end

  # subdomain
  draw :mobile
  draw :api
  
  ### 下面正常写其他路由就可以了
  resources :users
end
```

这样把其他routes文件放在 config/routes/ 目录下即可。


## 多个子域名下的_path和_url的使用
主要提一个 \*\_path 和 \*\_url 的区别，虽然一个是相对地址、一个是绝对地址，但是单个域名下，其实区别不大，所以很多人都是随手用。

但是一旦有了多个子域名，如果还是随手混用就会导致很多问题。所以，多个子域名下应该注意下面的几个约定：

1. 除非必要，只用 \*\_path

  多个子域名下，大部分的内链还是在子域名内部的，所以尽量使用\*\_path来引用url。如果不是的话，请考虑产品设计的是否合理、是否本子域名下也需要一个独立的url来满足需求。
  
2. 如果使用 \*\_url，那么一定要加上子域名的参数
  如果在跨域名访问的情况下（或者是mailer中），使用 \*\_url 的时候一定要加上 subdomain 的参数：
  
  ```ruby
  <%= link_to mobile_users_path(subdomain: Rails.configuration.mobile_subdomain), users.nickname %>
  ```
  
3. 使用url helper，而不是字符串来代表地址。
  这一点，初级的rails工程师很容易犯，觉得写一个字符串非常简单，干嘛还要搞一个url helper？可是一旦使用字符串表达url，一旦需要重构代码、升级产品的时候基本上代码是不可维护的，这时候只能默默流泪了。

4. javascript代码中引用url
  这种情况也不少，可以使用data-url的形式，如：
  
  ```ruby
  ### 这样
  <div data-url="<%= mobile_users_path %>"></div>
  
  ### 或者这样
  <%= content_tag :div, :'data-url' => mobile_users_path do %>
    some content
  <% end %>
  ```

## 差不多就是这样
好，差不多就是这样，希望在多子域名的问题上对大家有帮助，欢迎大家提意见~

广告时间：[大鱼自助游](http://www.fishtrip.cn)还在招ror工程师哦，初高级均可，欢迎[直接抛简历](mail:medal@fishtrip.cn)给我。


