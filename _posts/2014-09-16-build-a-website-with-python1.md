---
layout: post
title: "用python写网站（一）"
description: ""
category: "python"
tags: [python, website]
---

头一次用python写正规web（超过4，5个文件），虽然继续烂尾，但踩了不少坑，也抄了不少的代码，总结下。。

我主要参考了[这个项目](https://github.com/hustlzp/xichuangzhu)。

#### 数据库部分

首先关于数据库部分，xichuangzhu用了 falsk-sqlalchemy 也就是 flask 封装的 SQLAlchemy。我一开始没明白为什么要依赖这个东西。

于是参考[这篇文章](http://www.keakon.net/2012/12/03/SQLAlchemy%E4%BD%BF%E7%94%A8%E7%BB%8F%E9%AA%8C)直接裸用了SQLAlchemy。

一开始，我看 xichuangzhu 的项目对所有对数据库的操作都没有封装，全都是取session，execute xx， session.commit， session.close 这样的流程，实在是不能忍，于是学了上篇的文章对一些公用的操作进行写封装，就类似 ibatis 自动能生成 crud 模版一样。然后代码实际开始使用后发现了我封装的是操作都只适用于单线程，一多线程就2b了。


原来的封装如下

``` python
DB_CONNECT_STRING = 'mysql+mysqldb://root:111111@localhost/test?charset=utf8'
engine = create_engine(DB_CONNECT_STRING, echo=True, pool_size=10, max_overflow=0)
DB_Session = sessionmaker(bind=engine)
# 这是个带池的session pool
session = scoped_session(DB_Session)

# 公用model，附带公用方法
class BaseModel(BaseModel):
	__abstract__ = True

	@classmethod
	def set_attrs(cls, id, attrs):
	    if hasattr(cls, 'id'):
	        session.query(cls).filter(cls.id == id).update(attrs)
	        session.commit()

```

看起来很和谐，以为多线程下第一个线程做 query 的同时，第二个线程可以用同一个 session 继续query，其实不然，这个 session 在未 close 之前无法做别的查询，于是就2了。

于是再去看 flask-sqlalchemy 的源码知道这个东西存在的意义了。。。

``` python

class SQLAlchemy(object):

	def __init__(self, app=None,
                 use_native_unicode=True,
                 session_options=None):
        self.use_native_unicode = use_native_unicode

        if session_options is None:
            session_options = {}

        session_options.setdefault(
            'scopefunc', connection_stack.__ident_func__
        )

        self.session = self.create_scoped_session(session_options)

    def create_scoped_session(self, options=None):
        """Helper factory method that creates a scoped session."""
        if options is None:
            options = {}
        scopefunc=options.pop('scopefunc', None)
        return orm.scoped_session(
            partial(_SignallingSession, self, **options), scopefunc=scopefunc
        )

    def init_app(self, app):
    	...
    	teardown = app.teardown_appcontext
    	...

    @teardown
    def shutdown_session(response_or_exc):
        if app.config['SQLALCHEMY_COMMIT_ON_TEARDOWN']:
            if response_or_exc is None:
                self.session.commit()
        self.session.remove()
        return response_or_exc
```

它利用了flask的框架，为他的每个 request 都增加了 shutdown 的 hook 用于在一个请求完成后最后帮助 close 掉session。

可是也有个缺点，不是很通用，比较针对web，就是如果你的在一个请求中如果要请求多次数据库，那么这个过程中的 session 的 close 操作还是得自己 close，（因为你只有一个session） 这尼玛还是好蛋疼。

而且我业务代码都写的差不多了，该动起来也很烦，最最关键的是我用到多线程的地方只是个后台多线程处理xxx，和 flask 没有半毛钱关系，想用也用不了，于是只能自己写一个。

多线程操作db的原理就是每个线程拥有[属于自己的 session](http://docs.sqlalchemy.org/en/latest/orm/session.html#contextual-thread-local-sessions)，并且多线程的话必定是搭配session池来使用的，所以 session 在用完后不能 close，而是 remove 即将线程放回池。

session 的定义是 scoped_session， 然后在用完后将session close掉即可。

于是在网上找到个2种方式可以用来做一层代理。代理的作用是

1. create scoped_session

2. 拿 session 做xxx业务

3. session.remove

关于实现方法网上找到2种。 一个是利用[yield](http://pyzh.readthedocs.org/en/latest/the-python-yield-keyword-explained.html)，[stackoverflow 的具体例子](http://stackoverflow.com/questions/5544774/whats-the-recommended-scoped-session-usage-pattern-in-a-multithreaded-sqlalchem) 还有个是利用 [with](http://python.42qu.com/11155501)

由于不想以来其他的第三方包，于是用 with 实现了stackoverflow的例子

```python

class SessionProxy:
    def __enter__(self):
        self.session = scoped_session(DB_Session)
        return self.session

    def __exit__(self, type, value, trace):
        self.session.remove()
        logger.error('session error %s', trace) if trace is not None else trace

def get_session():
    return SessionProxy()

# 然后将公用 model 参数多加一个session（这样就可以是公用方法不被 session 绑死）
class BaseModel(BaseModel):
	__abstract__ = True

    @classmethod
    def set_attr(cls, session, id, attr, value):
        if hasattr(cls, 'id'):
            session.query(cls).filter(cls.id == id).update({
                attr: value
            })
            session.commit()

# 最后修改业务代码，在需要用到 session 的地方需要用 with 包起来，以实现proxy效果
# 当然可以将线程内部串型的方法写在一起。

def xxx():
	with get_session() as session:
		Video.set_attrs(session, v.id, {Video.status: 3, Video.result_status: 3})
		upload_video(v)
		Video.set_attrs(session, v.id, {Video.status: 4, Video.result_status: 4})

```

虽然这样也有点蛋疼，我的业务代码中所有操作db的代码需要用with包起来。但好处是session用完后可以不需要close。而且完美支持多线程，并发最多支持到同时get_session() `池的大小`个（有点拗口。。。）



