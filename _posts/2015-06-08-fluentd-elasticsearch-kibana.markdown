---
layout: post
title:  "日志管理工具的安装与配置"
date:   2015-06-08
categories: 日志管理
---
有很多[日志管理工具](http://www.importnew.com/12383.html)，除了这些还有 [InfluxDB + Grafana](https://ruby-china.org/topics/23470) 的组合。工具这么多，但还是有区别的，例如 ELK (logstash, elasticsearch, kibana) 侧重日志信息的收集、储存和搜索， InfluxDB + Grafana 侧重数据的监控。

下面主要写一下 Fluentd, Elasticsearch 和 Kibana 4 的安装和配置。

## Fluentd

fluentd 是一个日志收集系统，优点是可定制性强，扩展丰富，简单易用。 
[官方教程](http://docs.fluentd.org/v0.12/categories/installation) 
[参考资料1](https://www.digitalocean.com/community/tutorials/elasticsearch-fluentd-and-kibana-open-source-log-search-and-visualization#installing-fluentd-via-the-td-agent-package) 
[参考资料2](http://blog.vars.me/blog/2014/03/07/fluentdchu-tan-1-jian-jie-yu-an-zhuang/)

#### 安装

以 centos 举例：

```
curl -L https://td-toolbelt.herokuapp.com/sh/install-redhat-td-agent2.sh | sh
# 启动
sudo service td-agent start
```

配置文件 `/etc/td-agent/td-agent.conf` ，日志文件 `/var/log/td-agent/td-agent.log` 。

#### rails 的配置

rails 整合 fluentd 很简单，在 Gemfile 加上 [act-fluent-logger-rails](https://github.com/actindi/act-fluent-logger-rails) 这个 gem 。

环境配置文件 `config/environments/*.rb` ：

```ruby
config.log_level = :info
  config.logger = ActFluentLoggerRails::Logger.new(
    log_tags: {
      ip: :ip,
      ua: :user_agent,
      uid: ->(request) { request.session[:uid] }
  })
```

新建 `config/fluent-logger.yml` ：

```yaml
development:
  fluent_host:   'your.remote.host.ip'
  fluent_port:   24224
  tag:           'foo'
  messages_type: 'string'

test:
  fluent_host:   'your.remote.host.ip'
  fluent_port:   24224
  tag:           'foo'
  messages_type: 'string'

production:
  fluent_host:   'your.remote.host.ip'
  fluent_port:   24224
  tag:           'foo'
  messages_type: 'string'
```

#### 日志收集服务器上 fluentd 的配置

安装 elasticsearch 的插件

```
# 换 gem 源
sudo /opt/td-agent/embedded/bin/fluent-gem --remove https://rubygems.org/
sudo /opt/td-agent/embedded/bin/fluent-gem -a https://ruby.taobao.org/
sudo /opt/td-agent/embedded/bin/fluent-gem install fluent-plugin-elasticsearch
sudo /opt/td-agent/embedded/bin/fluent-gem install fluent-plugin-record-reformer
```

修改配置文件 `/etc/td-agent/td-agent.conf` ：

```
<source>
  type forward
  port 24224
</source>
<match foo>
  type elasticsearch
  logstash_format true
  flush_interval 5s # for debug
</match>
```

重启服务 `sudo service td-agent restart` 。

请注意，配置文件里 `source` 的 `type` 和 `port` 不能重复。

启动 rails 项目之后，日志文件就会自动收集到远程服务器上。更多配置请参考[官方文档](http://docs.fluentd.org/articles/config-file)。

## Elasticsearch

Elasticsearch 用于索引和搜索 log 的信息。 
[官方教程](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html) 
[参考资料](http://es.xiaoleilu.com/010_Intro/10_Installing_ES.html)

#### 环境要求

`Java 8 update 20 or later, or Java 7 update 55 or later`

建议安装最新版 jdk

```
java -version
yum remove java-1.6.0-openjdk  # 如果jdk低于要求，删除旧的jdk
yum search java | grep openjdk
yum install java-1.8.0-openjdk.x86_64

```

#### 安装

官方提供文件和库两种安装方式，我选择库安装的方法：

```
rpm --import https://packages.elastic.co/GPG-KEY-elasticsearch
sudo vim /etc/yum.repos.d/elasticsearch.repo
```

```
# /etc/yum.repos.d/elasticsearch.repo
[elasticsearch-1.6]
name=Elasticsearch repository for 1.6.x packages
baseurl=http://packages.elastic.co/elasticsearch/1.6/centos
gpgcheck=1
gpgkey=http://packages.elastic.co/GPG-KEY-elasticsearch
enabled=1.6
```

```
sudo yum install elasticsearch
sudo chkconfig --add elasticsearch
sudo service elasticsearch start
```

测试 `curl localhost:9200/_nodes/process?pretty`

```
{
  "status" : 200,
  "name" : "Namora",
  "cluster_name" : "elasticsearch",
  "version" : {
    "number" : "1.6.0",
    "build_hash" : "cdd3ac4dde4f69524ec0a14de3828cb95bbb86d0",
    "build_timestamp" : "2015-06-09T13:36:34Z",
    "build_snapshot" : false,
    "lucene_version" : "4.10.4"
  },
  "tagline" : "You Know, for Search"
}
```

我只是简单的试用，暂时不用修改默认配置，关于详细的配置可查看这两篇文章[集群配置](http://blog.csdn.net/changong28/article/details/38292305)，
[配置详解](http://rockelixir.iteye.com/blog/1883373)。

## Kibana 4

Kibana 是一个 Elasticsearch 分析和搜索仪表板。 [参考资料](http://kibana.logstash.es/content/kibana/index.html)

#### 安装

```
wget https://download.elastic.co/kibana/kibana/kibana-4.1.1-linux-x64.tar.gz
tar xvf kibana-4.1.1-linux-x64.tar.gz
kibana-4.1.1-linux-x64/bin/kibana
```

用浏览器访问 `http://yourhost.com:5601` ，参考[简单设置](http://kibana.logstash.es/content/kibana/v4/setup.html#%E8%AE%A9-kibana-%E8%BF%9E%E6%8E%A5%E5%88%B0-elasticsearch)完成初步配置。

启动我们之前设置好的 rails 项目，访问一两个网页，我们就可以在 kibana 上看到日志的信息了 ![kibana-example](http://r.loli.io/VV7JRv.png)