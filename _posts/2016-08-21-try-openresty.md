---
layout: post
title: "openresty lua 小记"
description: ""
category: "play"
tags: [lua, openresty, lelouchcrgallery]
published: true
---

最近1周，陆陆续续的花了[10个小时](https://wakatime.com/@282d837d-0987-4551-8598-40b5577dd621/projects/rrgdfphytn?start=2016-08-15&end=2016-08-21)，将我的 [私人项目](http://lelouchcrgallery.tk/) 从 jvm 迁移到 openresty 上。

至于为什么要迁移, 主要是最近发现 vps 的内存不够用了，看一下下面的数据

迁移之前

```
%CPU %MEM ...
0.0  55.9 349:51.88 java -Xmx299m -Xms299m -server -jar xxx.jar
0.0  10.2 133:00.58 ./src/redis-server 127.0.0.1:6379
```

benchmark

```
root@vultr:~/wrk-4.0.2# ./wrk -t2 -c 20 -d30s 'http://localhost:9000/p/p/1'
Running 30s test @ http://localhost:9000/p/p/1
  2 threads and 20 connections
^C  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   321.11ms  370.56ms   1.84s    87.80%
    Req/Sec    12.13      7.15    30.00     58.89%
  134 requests in 11.22s, 238.29KB read
```

迁移之后

```
%CPU %MEM ...
0.0  0.5   0:10.83 nginx: worker process
0.3  0.5   0:10.88 nginx: worker process
0.0  0.4   0:00.00 nginx: master process ./
0.0  10.2 133:00.58 ./src/redis-server 127.0.0.1:6379
```

benchmark

```
root@vultr:~/wrk-4.0.2# ./wrk -t2 -c 20 -d30s 'http://lelouchcrgallery.tk/p/1'
Running 30s test @ http://lelouchcrgallery.tk/p/1
  2 threads and 20 connections
^C  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    78.30ms  106.42ms 355.13ms   79.17%
    Req/Sec     2.94k     1.94k    5.62k    47.16%
  57459 requests in 10.71s, 177.11MB read
```

结果就是 资源耗费下降了，性能上去了，而且上去了不是一点点 (不过这里不是在黑 java，毕竟 以前为了图方便且没有什么性能要求没用nio, 用netty 的话 性能肯定会提上去)

把原本在java层做的 业务全利用 redis 的script 放到 redis-server 里面做了。


我是根据 [Openresty 最佳实践](https://moonbingbing.gitbooks.io/openresty-best-practices)这本书边写边看的, 但还是遇到了不少坑，下面记录一下


## module 

由于lua 的module 能cache 所以尽量将 一些 immutable 的变量放到module 里面，避免重复申请

```lua
local _M = {}

local ran_arr = { 0.5, 0.6, 0.625, 0.7, 0.8, 0.85, 0.9, 0.95, 1, 1.05, 1.1, 1.15, 1.2, 1.25, 1.3, 1.35,1.4, 1.5, 1.75, 2, 2.5, 3 }

local ran_arr_len = #ran_arr
math.randomseed(os.time())

-- 获得随机的 比率
function _M.ran_ratio(self, ...)
    local args = {...}
    return ran_arr[math.random(ran_arr_len)]
end

return _M
```

这样，ran_arr 数组 一个nginx 进程只会有一份


## 局部变量

最佳实践 里面很多篇幅都提到 没有加local 的变量就是全局变量，容易坑，但是有的时候不得不使用全局变量

在openresty 里面可以用[三种全局变量](https://moonbingbing.gitbooks.io/openresty-best-practices/content/ngx_lua/lua-variable-scope.html)

比如有一个非常大的script， 我不想每次都eval， 我想script load 后 evalsha，减少请求大小。

所以我不得不找一个全局变量来存 script_sha 

由于 进程间 共享变量要用 `lua_shared_dict` 实现，所以建议放在 `init_by_lua` 中实现 [参考](http://www.mrhaoting.com/?p=157)

```
lua_shared_dict dogs 1m;
init_by_lua '
    local dogs = ngx.shared.dogs;
    dogs:set("Tom", 50)
'
server {
    location = /api {
        content_by_lua '
            local dogs = ngx.shared.dogs;
            ngx.say(dogs:get("Tom"))
        '
    }
}
```

不过由于我的vps 只有一个核，所以就一个 进程，所以 我直接用了 进程内的变量，也就是lua vm 的全局变量

```lua
api_script_sha = nil

function _M.api_script_sha(...)
    local arg = {...}
    local red = arg[1]
    if not api_script_sha then
        ngx.log(ngx.WARN, "start api_script_sha : ")
        api_script_sha, err = red:script("load", api_script)
        ngx.log(ngx.WARN, "end api_script_sha : ", api_script_sha)
    end
    return api_script_sha
end
```

## lua_code_cache

在开发时， 可以关闭 `lua_code_cache` 选项，方便调试。

不过由于开启了这个选项, 所以，相当于每次请求都会初始化 lua vm，并且 reload 所有 script。

所以，当你需要调试 lua vm 级别的 global variable 的时候，还是要把它开启

## 调试

lua table 不能直接 print log，可以用 `cjson.safe` print 

```lua
print("--> res type:" .. type(res) .. ' -- ' .. cjson_safe.encode(res))
```

## random

lua 的random 比较坑爹，即使用 math.randomseed (os.time()) 也是没秒变化一次

但是更坑爹的事 redis 的 lua script 并不包括 os [see ](http://redis.io/commands/eval#available-libraries) 并且不抛错

所以根据文档，在 redis里面 用redis 的time 来做seed (能精确到ms 级别)

```lua
local _time = redis.call('time')  -- {"1471783692","979591"}
math.randomseed(tonumber(_time[2]))
```

## 开发环境

开发环境 建议直接用类似虚拟机的环境比如docker，以便节省不必要的时间花在不同环境下.


# refernece

https://moonbingbing.gitbooks.io/openresty-best-practices

http://redis.io/commands/eval#available-libraries


