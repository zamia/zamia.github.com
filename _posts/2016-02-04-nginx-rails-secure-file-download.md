---
layout: post
title: nginx+rails下进行文件的安全下载
date: 2016-02-04 18:30:00
disqus: y
---

## 问题

一般常见的文件下载有两类需求：

* 公开的文件下载，比如 rails 的 assets、或者用户上传的一些可以公开的文件，比如自己的头像；
* 较隐私的文件下载，使用场景也不少，比如某些企业后台上传的用户认证资料；

前一类需求rails中比较容易实现，一般直接把用户上传的文件存储目录直接放在 /public/some/dir 中，然后按照日期或者ID之类的做个目录结构也就够用了。

对于后一类场景，具体的需求有两点：

1. 必须经过用户认证和授权，检查认证和权限之后才能访问文件；这个就是典型的业务层需要处理的，也就是app server(rails层)需要实现的；
2. 保证速度，不能给 app server 过多负担。这个是web server擅长做的，比如nginx可以使用系统的sendfile功能可以直接从文件系统发送至网络层，可以少2次的内存copy；web server也可以做一些缓存等等；

根据上面的需求，文件的安全下载实现起来稍微麻烦一点，不过根据『这么通用的需求一定有现成的解决方案』的原则，其实配置和使用起来还是比较简单的，这篇小文就总结下使用nginx和rails配合、利用X-Accel（一般也称作X-Sendfile）来实现隐私文件的安全下载。

## 底层原理和实现
这是典型的web server和application server互相配合的过程，内部的访问过程 [这篇文章](http://thedataasylum.com/articles/how-rails-nginx-x-accel-redirect-work-together.html) 也讲的比较清楚，下面的总结更详细一些，补充了一些源码和日志，能帮助大家彻底搞清楚整个过程。

1. 浏览器访问一个地址，比如 /download/files/123 ，这个地址对应一个需要验证权限的文件；

```
GET /download/files/123 HTTP/1.1
```

1. nginx收到这个请求之后根据路由配置，把请求转发到rails。
	
	nginx转发的时候根据配置，添加上两个参数，转发给rails。后端rails服务器会根据这两个参数来对response body进行修改，后面会提到。

```
GET /download/files/123 HTTP/1.0
X-Forwarded-For: 127.0.0.1
X-Sendfile-Type: X-Accel-Redirect
X-Accel-Mapping: /var/www/fishtrip/private=/private
```

	上面的参数 X-Sendfile-Type 告知后端nginx支持什么样的参数（像apache、lighttpd支持的参数名称不同）；参数 X-Accel-Mapping 告知后端应该怎么样做文件名称的mapping。

1. rails收到这个请求之后，正常流进某个controller#action，经过业务代码的判断之后，找到这个url对应的真正的文件名，然后使用sendfile发送文件。

```ruby
def show
  pic = File.find params[:id]
  send_file pic.path, type: "image/jpeg", disposition: 'inline'
end
```
	
	其实在整个过程中，rails的背后是 [Rack::Sendfile](http://www.rubydoc.info/github/rack/rack/Rack/Sendfile) 这个middleware在工作。看看它的源码中的call函数的实现：
	
```ruby
# File rack/lib/rack/sendfile.rb
case type = variation(env)
when 'X-Accel-Redirect'
   path = F.expand_path(body.to_path)
   if url = map_accel_path(env, path)
     headers['Content-Length'] = '0'
     headers[type] = url
     body = []
   else
     env['rack.errors'].puts "X-Accel-Mapping header missing"
   end
 when ...
 end
 # some code here
```
	
	可以看到这个middleware吧content-length置为0，把body置空，返回给前端一个计算过mapping的url。
	
	其中的函数 map_accel_path 是private函数，长这样：
	
```ruby
def map_accel_path(env, file)
    if mapping = env['HTTP_X_ACCEL_MAPPING']
      internal, external = mapping.split('=', 2).map{ |p| p.strip }
      file.sub(/^#{internal}/i, external)
    end
  end
```
	
	其实就是根据nginx传入的 X-Accel-Mapping 参数把实际的地址替换成一个mapping地址。
	
	所以，这样也就不难猜测我们自己写的action里面的sendfile的实现了，sendfile只需要实现一个支持 to_path 调用的对象即可。去Rails中看看它的实现：
	
```ruby
# File actionpack/lib/action_controller/metal/data_streaming.rb
def send_file(path, options = {}) #:doc:
   # 省略一些代码
   self.status = options[:status] || 200
   self.content_type = options[:content_type] if options.key?(:content_type)
   self.response_body = FileBody.new(path)
 end
```
	
	里面的 FileBody 类就支持 to_path 调用；
	
	所以，经过 Rails 和 Rack::Sendfile 的配合，rails返回给nginx的就是一个没有 body ，只有 headers 的 response ，长下面这个样子（来源于 nginx 的 debug 日志，略去了部分内容）：
	
```
http proxy header: "Content-Disposition: inline; filename="abc.jpg""
http proxy header: "Content-Transfer-Encoding: binary"
http proxy header: "Content-Type: image/jpeg"
http proxy header: "Content-Length: 0"
http proxy header: "X-Accel-Redirect: /private/files/abc.jpg"
```
	
1. nginx 收到 rails 返回的数据之后，会检查 X-Accel-Redirect 参数的值，然后内部再根据 location 的配置进行内部跳转，找到真正的文件地址（需要配置，见下文），并且调用操作系统的 sendfile 接口，直接返回给用户。

这个就是整个文件安全下载的过程了，这个过程涉及到 nginx - rails - Rack::Sendfile - nginx 的这么一个过程，看起来有点复杂。实际上我们使用的时候配置起来还是比较简单的。


## 实现和配置
配置主要是两部分，nginx 和 rails 的部分，如果使用 capistrano 部署线上服务，因为涉及到软链的问题，所以配置略有不同。

### nginx

根据上面的原理部分的描述，nginx 的配置也分为两个部分：

1. 跳转后端时参数配置

```ini
set $app_root /var/some/dir;

location /download {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Sendfile-Type X-Accel-Redirect;
        proxy_set_header X-Accel-Mapping "$app_root/private=/private";

        proxy_set_header Host $http_host;
        proxy_redirect off;
        expires off;

        proxy_pass http://backend;
}
```
	
	这个配置的作用是 nginx 在把请求转发给rails 后端的时候添加 X-Senfile-Type 和 X-Accel-Mapping 参数；

2. 收到后端回复后内部地址的配置 

```ini
location /private {
        internal;
        alias $app_root/private;
}
```
	
	这个配置的作用是 nginx 收到 rails 后端返回的值时可以正确找到文件的实际地址；

### rails

rails 的配置就更简单了，添加一句话即可：

```ruby
config.action_dispatch.x_sendfile_header = 'X-Accel-Redirect'
```

### capistrano

capistrano 的部署使用了软链的方法，所以上面nginx 配置的地方，需要添加一个正则即可。这样后端 rails 就可以正常的去 mapping 了（也就是 Rack::Sendfile 里面的 map_accel_path 是支持正则的）：

```ini
proxy_set_header X-Accel-Mapping "$app_root/releases/\d{14}/private=/private";
```

如果实际部署情况跟这个不一致，只要走类似的方法就行了，你懂的~

## 总结
虽然文件的安全下载是一个小功能，而且现在文件的云存储很多（大鱼也迁移到了云存储上...），但是通过这里例子也可以看看 web server 和 application server 是如何配合工作的，反向代理的很多功能也都是类似机制完成的。

通过这个例子也可以了解一下 rack 的工作机制，可以看到如果通过简单的代码来实现一个相对复杂的功能.

以上~

## 参考文章
1. <http://thedataasylum.com/articles/how-rails-nginx-x-accel-redirect-work-together.html>
2. <https://gist.github.com/Djo/11374407>
3. <https://www.nginx.com/resources/wiki/start/topics/examples/x-accel/>
4. <http://airbladesoftware.com/notes/rails-nginx-x-accel-mapping/>
5. <http://www.rubydoc.info/github/rack/rack/Rack/Sendfile>
