---
layout: post
title: "关于http session"
description: ""
category: "web"
tags: [java,clojure,compojure,ring,session]
---

最近在玩clojure，准备用[compojure](https://github.com/weavejester/compojure)和[ring](https://github.com/ring-clojure/ring)写一个网站。写登录注册模块的时候纠结在session上了。说来惭愧，上班都快一年了连session都没玩过。

然后先网上搜一下看到了[这个](http://www.zhihu.com/question/19786827)，然后再看ring的源码的时候，发现各种问题。结论，zhihu真的不靠谱。

#### 什么是http
简单来说http，干的事情就是服务端和客户端相互请求和发送字符串。

ring把它抽象成一段hash

```html
{:ssl-client-cert nil, :remote-addr "0:0:0:0:0:0:0:1", :scheme :http, :query-params {}, :session {}, :form-params {}, :multipart-params {}, :request-method :get, :query-string nil, :route-params {:x "1123"}, :content-type nil, :cookies {"ring-session" {:value "ZiCQ0rUvKHWdiS42Z6MidZci8tGKzehBcwtyJJjMBRJhDRLnEKWtpYH2OgLdA8rC--Ko2pbNbOTEUdNP3TWcULabbybHjc74hAVHX4Ddr+G9U="}}, :uri "/sess/1123", :session/key nil, :server-name "localhost", :params {:x "1123"}, :headers {"accept-encoding" "gzip, deflate", "user-agent" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_3) AppleWebKit/537.76.4 (KHTML, like Gecko) Version/7.0.4 Safari/537.76.4", "connection" "keep-alive", "accept-language" "zh-cn", "accept" "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8", "host" "localhost:3000", "cookie" "ring-session=ZiCQ0rUvKHWdiS42Z6MidZci8tGKzehBcwtyJJjMBRJhDRLnEKWtpYH2OgLdA8rC--Ko2pbNbOTEUdNP3TWcULabbybHjc74hAVHX4Ddr%2BG9U%3D"}, :content-length nil, :server-port 3000, :character-encoding nil, :body # <httpinput org.eclipse.jetty.server.httpinput@4decb8c1="">, :flash nil}
```

当server传给clinet这个字串后，client页面显示的部分仅仅是httpinput 这部分。

#### 关于cookie
网上关于[cookie](http://zh.wikipedia.org/wiki/Cookie)的解释很多。t它的特点是可以dump到client端。

#### session
session是附在request和response种的一段信息。只不过他是会话级别的，也就是说仅存于request和response里。

举例可见，裸用ring的session

```clojure
(GET "/name" [x :as {session :session} :as r]
  (let [count (:count session 0)]
    (do
      (println (:count session))
      {:status  (or nil 200)
       :session (assoc session :count (inc count))
       :body    (str r)
       }
      )))
```
从request里面取出key位session的内容，然后经过xxx操作后，再写回去。

当然直接用ring的session太不方便了。直接把[lib-noir-session](https://github.com/noir-clojure/lib-noir/blob/master/src/noir/session.clj)的代码搞到自己那里，注意namespace。当然也可以加一个lib－noir的依赖，不过最新的lib－noir依赖的ring不是最新的，会比较恶心。



session可以存很多地方，ring提供了sessionStore的接口`ring.middleware.session.store`，自带2中实现分别是 memeryStore（遇到爬虫岂不完蛋！）和cookieStore，第三方的[redis-session](https://github.com/wuzhe/clj-redis-session)

session存储原理是他会存一个sessionid 到cookie里面，然后根据此id 从自己定义的cookie的地方取出session，然后xxx。

```clojure
(defn- bare-session-request
  [request & [{:keys [store cookie-name]}]]
  (let [req-key  (get-in request [:cookies cookie-name :value])
        session  (store/read-session store req-key)
        session-key (if session req-key)]
    (merge request {:session (or session {})
                    :session/key session-key})))

(defn session-request
  "Reads current HTTP session map and adds it to :session key of the request.
  See: wrap-session."
  {:arglists '([request] [request options])
   :added "1.2"}
  [request & [options]]
  (-> request
      cookies/cookies-request
      (bare-session-request options)))
```

session一般存于server端，不过并不是说不能存在clinet，可以存到cookie里面。对于自己写的小东西，还是存cookie吧，

否则如果遭遇无良crawler，每次请求每次记录一个垃圾session再server端。即使有ttl，对于洁癖者来说，岂不是很蛋疼。。。

