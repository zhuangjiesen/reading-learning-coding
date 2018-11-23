# redis 集群工具 redis-trib.rb 使用；

### 运行命令：
```

ruby redis-trib.rb

```

### 刚开始使用是报错的，因为是ruby 语言写的，需要安装 ruby 2.0以上的环境，安装完控制台中
```

查看当前安装成功否，并查看ruby安装版本号
ruby -v 

```


### 接着还需要安装 ruby 的 redis 模块，

```
查找redis 模块是否在 gems 目录中，说明安装 ruby 的 redis 模块成功
# find / -name "redis"
/Library/Ruby/Gems/2.3.1/gems/redis-3.0.6/lib/redis

```

### gems 安装 ruby 的 redis 模块

```
查看 gem 版本号
gem -v 

//因为gems官网的镜像在国外，所以国内网络经常断连。你需要链接一个国内镜像，我用的是https://ruby.taobao.org

查看镜像
gem sources -l

替换镜像
gem sources --add https://ruby.taobao.org/ --remove https://rubygems.org/

安装可能会失败，由于permision denied 权限问题

找到gems 的安装目录
sudo find / -name "Gems"
/Library/Ruby/Gems
cd /Library/Ruby/Gems
进入目录，将目录下的 2.0.0 （就是ruby 安装目录设置权限）
sudo chmod 777 2.0.0

接着就可以直接安装redis 模块了
sudo gem install redis




```

### 接着运行
```
ruby redis-trib.rb
就不报错了

$ ruby redis-trib.rb 
Usage: redis-trib <command> <options> <arguments ...>

  create          host1:port1 ... hostN:portN
                  --replicas <arg>
  check           host:port
  info            host:port
  fix             host:port
                  --timeout <arg>
  reshard         host:port
                  --from <arg>
                  --to <arg>
                  --slots <arg>
                  --yes
                  --timeout <arg>
                  --pipeline <arg>
  rebalance       host:port
                  --weight <arg>
                  --auto-weights
                  --use-empty-masters
                  --timeout <arg>
                  --simulate
                  --pipeline <arg>
                  --threshold <arg>
  add-node        new_host:new_port existing_host:existing_port
                  --slave
                  --master-id <arg>
  del-node        host:port node_id
  set-timeout     host:port milliseconds
  call            host:port command arg arg .. arg
  import          host:port
                  --from <arg>
                  --copy
                  --replace
  help            (show this help)

For check, fix, reshard, del-node, set-timeout you can specify the host and port of any working node in the cluster.

```


### 使用

```


redis 目录：
/Users/zhuangjiesen/develop/redis/redis-3.2.3/src

cd /Users/zhuangjiesen/develop/redis/redis-3.2.3/src


./redis-trib.rb create --replicas 1 127.0.0.1:9001 127.0.0.1:9002 127.0.0.1:9003  127.0.0.1:9004  127.0.0.1:9005  127.0.0.1:9006

```


