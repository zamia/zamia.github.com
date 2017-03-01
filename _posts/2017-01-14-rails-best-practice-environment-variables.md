---
layout: post
title: Rails最佳实践之配置管理
date: 2017-01-14 11:20:00
disqus: y
---

大家日常开发Rails项目的过程中一定会遇到些配置项（Configuration)，随着配置越来越多，总归需要管理起来，那么如何管理这些配置呢？本文期望梳理一下，找到一个较好的解决方案。

（其实是我们自己项目中的配置文件非常多且乱，所以力图找到一个好的方法来管理，从最终效果来看，这种方法应该是比较合理的，拿来跟大家分享。）

先放结论：

* 应用配置统一使用 settings.yml 来管理；
* 每一个资源配置（数据库、redis、第三方API）使用独立的配置文件管理；
* 资源配置中的敏感信息要使用环境变量管理；

如果不太理解的话可以往下看，如果项目中已经这么做了，不用浪费时间~

## 配置的简单分类

先把如何管理放在一边，看看配置都包括哪几个类别：

* Rails基础配置 (Framework Configuration)
  比如Rails需要配置 session store的地址、asset host、action mailer等等，这些配置都集中在 /config/application, 不同的环境放在 config/environments下面。

* 应用配置（Application Configuration)
  比如分页时每页的条数，开发环境数据不多，每页10条，线上环境需要50条。那这个分页条目就是简单的一项配置了。还有一些比如开关类的配置，临时加一个开关，到了固定的时候开启某些功能，都属于典型的应用配置。
  
* 资源配置（Resource Configuration）
  Rails项目本身应该是无状态的（Stateless），所有的资源都应该独立通过配置的形式体现。比如数据库就是一种资源（Resource），像Redis、ElasticSearch都可以认为是一种资源。另外，我们把第三方服务也称为资源，比如项目中需要访问第三方API，那么这个API我们也认为是一种资源。
  
Rails的基础配置无需多说，大家应该都很熟悉了，基本上Rails都已经做好了样例，按照样例配置就可以了。比如 config/application.rb 可能会有这些东西：

```
config.autoload_paths += Dir["#{config.root}/app/utilities/"]
config.i18n.default_locale = "zh-CN"
config.active_job.queue_adapter = :sidekiq
config.time_zone = 'Beijing'
```

所以下面直接说说应用配置和资源配置。

## 应用配置

### 简单版本
最常用的方法其实是在 config 目录下面写一个 settings.yml 的文件，比如：

```yml
# config/settings.yml
default: &default
	page_size: 10
	enable_some_feature: false
	
development:
	<<: *default
	
production:
	<<: *default
	page_size: 100
	enable_some_feature: true
```

然后添加一个 initializer 来加载它:

```ruby
# config/initializer/load_settings.rb
$settings = YAML.load_file("#{Rails.root}/config/settings.yml")[Rails.env].symbolize_keys

# use config in somewhere
logger.debug $settings[:page_size)
```

### config_for 版本
Rails 4.2 以后也提供了简单的方法来加载配置：

```ruby
# config/application.rb
$settings = config_for(:settings)
```

使用 config_for 可以自动加载对应RAILS_ENV的配置，还可以加载ERB内容，所以尽量使用 config_for 来加载配置。

但是上面两种方法本质差不多，也可以这样用，不过稍微也有一些不方便的地方：

* 使用不太方便，都是字符串做key，如果使用 symblolized_keys的话又不支持级联；
* 手动加载时不能使用ERB，比如加载环境变量等。

### 使用gem - railsconfig/config 

这个gem稍微升级了一下，做了方便使用的改动，比如自动支持多环境、支持ERB、支持『.』调用，还可以多级调用。使用也比较简单，一看文档即知。

```ruby
# Gemfile
gem 'config'

# 执行下面的命令，会生成一些模板
# rails g config:install

# 调用时
logger.debug Settings.some_config.other_config
```

这个Gem使用起来很方便，所以我们推荐应用配置均使用 setting.yml，不同的环境的文件放到 settings目录，这些内容并不包含敏感信息，因此可以放心的 checkin 到 git 库中。

```
config/settings.yml
config/settings/production.yml
config/settings/development.yml
config/settings/test.yml
```

## 资源配置

资源配置会稍微麻烦一些，先说下什么是资源。以下内容来源于 [12factor](https://12factor.net/zh_cn/backing-services):

* 把后端服务(backing services)当作附加资源。
* 后端服务是指程序运行所需要的通过网络调用的各种服务，如数据库，消息/队列系统，SMTP 邮件发送服务，以及缓存系统。
* 类似数据库的后端服务，通常由部署应用程序的系统管理员一起管理。除了本地服务之外，应用程序有可能使用了第三方发布和管理的服务。
* 12-Factor 应用不会区别对待本地或第三方服务。 对应用程序而言，两种都是附加资源，通过一个 url 或是其他存储在 配置 中的服务定位/服务证书来获取数据。

因为后端服务通常由系统管理员（运维同学）来统一管理，运维架构对于开发工程师来讲可能是透明的，比如一个数据库地址，可能是一组集群，管理员只会提供一个入口的vip而已。

明白了什么是资源之后，我们就把资源相关的配置也分为两类：

* 非敏感信息，一般也是开发工程师可以感知的信息。比如一个API的地址；
* 敏感信息，一般不对开发工程师开放。比如一个API的认证信息；

### resource.yml.example 的方式管理
对于资源类的配置，Rails默认的做法是采用 resource.yml，但是在 git 库中使用 resource.yml.example 的形式。

比如rails项目产生开始就会产生一个 database.yml.example，一般我们通过修改这个文件，然后copy/move一份 database.yml出来使用。

这种方式有几个问题，[12factor](https://12factor.net/zh_cn/config) 也有详细描述；

1. 随着资源越来越多，这类的yml越来越多，管理起来比较麻烦；
2. 每次添加一个资源都添加yml的方式很容易漏掉，导致敏感文件加入了git库；
3. 跟语言绑定，无法跨语言使用等；

### 使用 yml + 环境变量来管理资源类配置

虽然 12factor 中推荐应用配置存储在环境变量中，但是我们更近一步，只把资源配置中的敏感信息存储在环境变量中，而非敏感信息仍旧存储在yml中。

这样的话配置的目录大概是这样：

```ruby
# 配置相关目录结构
rails-app
  config
    mongo.yml
    redis.yml
    rabbitmq.yml
    mail_api.yml
.env
```

一个资源的配置文件大概是这样：

```yaml
# mongo.yml
default: &default
  host: ENV["MONGO_HOST"]
  port: ENV["MONGO_PORT"]
development:
  <<: *default
  database: mongo_development
production:
  <<: *default
  database: mongo_production
```

.env 文件大概是这样子的：

```bash
# .env 文件
MONGO_HOST=127.0.0.1
MONGO_PORT=27017
```

通过结合 yaml文件和env文件，资源的配置被分割为敏感信息和非敏感信息，所有的 yml 都可以安全的 checkin 到git库中，而 .env 文件是在部署的时候由运维工程师通过使用一些自动化工具来部署到线上环境。

因为 .env 文件在 rails 环境中无法做到自动加载，因为我们还需要一个 gem 来辅助：

```ruby
# Gemfile
gem 'dotenv-rails', require: 'dotenv/rails-now'
```

通过使用 dotenv 这个 Gem，可以做到很方便的自动加载 .env 文件，它甚至可以自动根据 Rails Env 来加载 .env.production 这样的配置，不过我们不需要这个功能，只需要一个 .env 就可以了。

然后就可以很方面的加载资源类的配置了：

```ruby
# config/application.rb 中添加
config.redis = config_for(:redis)

# 在 config/initializer/load_redis.rb 中就可以使用了:
$redis = Redis.new(Rails.configuration.redis)
```

另外，如果资源类文件较多，也可以把所有的资源类的配置统一放在一个目录下，这样就更清晰了。

```ruby
# config/application.rb

# 所有的资源配置放在 config/resources 下面
config.redis = config_for("resources/redis") 
```

这样就基本完成了，这样使用 yml + env 的方式的好处包括：

1. 所有的yml都可以安全的checkin到git库中了，里面不再包含敏感信息；
2. 一些非敏感的字段在 yml 中给开发工程师来集中管理，这样 env 变量就比较少了；
3. 部署的时候线上只需要一个 .env 就可以搞定所有的资源类的配置了；比如 capistrano 只需要 link 这一个文件即可。

## 总结

虽然配置管理是一个小事情，但是随着项目越来越复杂，调用的第三方服务越来越多，如果不好好进行规划，配置比较混乱，同时也容易出现安全事故（安全无小事啊！）

总结一下：

1. 应用配置，我们通过 config 这个gem，统一存储在 settings.yml 中，通过 Settings 对象来调用。
2. 资源配置，基本的资源配置统一存储在 config/resource.yml 中，通过 config_for 来加载；
3. 资源配置，敏感类信息通过存储在 .env 文件中，在部署时由运维进行管理，dotenv 来加载到 ENV 变量中；

以上~



