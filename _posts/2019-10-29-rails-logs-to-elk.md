---
layout:     post
title:      "Rails应用将日志送到ELK中"
subtitle:   "Rails ELK integration"
date:       2019-10-29
author:     "leasy"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Elastic
    - Logstash
    - Kibana
    - Rails
    - Docker
---

> "Rails日志发送到ELK平台"

最近在研究Rails如何将日志送到ELK中，这里记录下。我会创建

使用工具
- [Lograge](https://github.com/roidrage/lograge)
- [Logstash-logger](https://github.com/dwbutler/logstash-logger)
- [Elastic search](https://www.elastic.co/guide/en/elastic-stack-get-started/current/get-started-docker.html)
- [Logstash](https://www.elastic.co/guide/en/logstash/current/docker.html)
- [Kibana](https://www.elastic.co/guide/en/kibana/current/docker.html)
- [Docker](https://www.docker.com/)

### 准备工作
- 配置ELK，我是用Docker来配置和运行ELK工具的，如何配置参考[TBD](https://github.com/LeasyZhang/rails-elk-docker)
- 创建一个Rails项目，运行
```bash
$ rails new app
```
创建一个结构完整的Rails 应用
- 安装依赖库，打开Gemfile,添加
```ruby
gem 'lograge'
gem 'logstash-event'
gem 'logstash-logger'
```
三个依赖，运行
```bash
$ bundle install
```
命令安装依赖
- 

### 格式化Rails日志
首先在config/initializers添加一个配置文件logstash.rb,添加下面的配置
```ruby
    Rails.application.configure do
        config.lograge.enabled = true
        config.lograge.formatter = Lograge::Formatters::Logstash.new
    end
```
这个时候运行程序，访问localhost:3000输出的结果应该是
```json
{
    "method": "GET",
    "path": "/",
    "format": "json",
    "controller": "index",
    "action": "/",
    "status": 200,
    "duration": 27.2,
    "view": 0.22,
    "db": 11.32,
    "@timestamp": "2019-10-30T10:19:14.766Z",
    "@version": "1",
    "message": "[200] GET / "
}
```
如果我们需要在日志中添加更多的字段，那么可以在application_controller.rb中添加方法
```ruby
    def append_info_to_payload(payload)
        super
        payload[:uuid] =  request.uuid
        payload[:remote_ip] = request.remote_ip
    end
```
然后在logstash.rb中继续添加配置文件
```ruby
config.lograge.custom_options = lambda do |event|
    {
        :uuid => event.payload[:uuid],
        :remote_ip => event.payload[:remote_ip]
    }
end
```
这样子配置完,输出的日志格式就变成了
```json
{
    "method": "GET",
    "path": "/",
    "format": "json",
    "controller": "index",
    "action": "/",
    "status": 200,
    "duration": 27.2,
    "view": 0.22,
    "db": 11.32,
    "@timestamp": "2019-10-30T10:19:14.766Z",
    "@version": "1",
    "message": "[200] GET / ",
    "uuid": "0x908fjasq09qjfsoidapfu",
    "remote_ip": "127.0.0.1"
}
```
多了两个自定义的字段。
### 日志送到logstash
logstash的配置和使用在另外一篇文章介绍，这里就找了一个简单的配置文件说明要送到logstash需要做的配置
```bash
input {
    stdin {}
  file {
    type=> 'application'
    path=>'~/out.log'
    codec=>'json'
  }
}

output {
  stdout {}
  elasticsearch {
    cluster=> 'logstash'
    protocol => http
  }
}
```

然后替换lograge的logger
```ruby
config.lograge.logger = LogStashLogger.new(type: :tcp, host: localhost, port: 5288)
```
这样子就可以把日志送到logstash，因为ELK已经配置好，所以可以在kibana上面看到日志。默认的日志是"logstash-日期",在kibana中新建一个"logstash-*"的index pattern就可以搜索到输出的日志。

### 替换Rails.logger
在Rails中打印日志通常是用
```ruby
Rails.logger.info("some logs")
```
来打印日志。
尝试了一下在application.rb中用
```ruby
config.log = LogStashLogger.new(type: :tcp, host: localhost, port: 5288)
```
发现会报错，所以就创建了一个module，里面持有一个logstash-logger对象
```ruby
module LogHelper
    Logger = LogStashLogger.new(type: :tcp, host: localhost, port: 5288)
end
```
用LogHelper::Logger的方式来使用logstash logger,比如
```ruby
LogHelper::Logger.info 'some logger'
```
但是这样做使用的是logstash-logger默认的输出格式，不会受到上面lograge配置的影响，所以需要配置logstash-logger
```ruby
module AapLoggerHelper
    LogStashLogger.configure do |config|
        config.customize_event do |event|
          event["log_format"] = "json"
          event["log_level"] = "info"
        end
    end
    Logger = LogStashLogger.new(type: :tcp, host: localhost, port: 5288)
end
```
这样子就可以定义自己想要输出的格式了。

### 结论
这篇文章简单介绍如何在Rails中把日志通过logstash送到elastic中，然后在Kibana上进行搜索和分析，首先是引入相关依赖，然后是配置lograge和logstash，这个时候已经可以把访问日志通过logstash送到elastic中，然后介绍了如何替换默认的log，将日志输出的logstash中。
不过还有很多地方没有介绍，如何处理异常，如何记录db的日志，而且感觉创建log的方式不够优雅，还有很多需要改进。