---
title: Flask sqlalchemy调优总结
date: 2017-7-23 20:23:11
tags: 
 - flask
 - sqlalchemy
categories:
 - 后端
---

直接用tcpdump监听数据库端口（mysql为例）可以打印所有的sql语句。（注意修改网卡的名称）

``` bash
king@king:/tmp$ sudo tcpdump -i lo -s 0 -l -w - dst port 3306 | strings | perl -e '
while(<>) { chomp; next if /^[^ ]+[ ]*$/;
  if(/^(SELECT|UPDATE|DELETE|INSERT|SET|COMMIT|ROLLBACK|CREATE|DROP|ALTER)/i) { 
    if (defined $q) { print "$q\n"; }
    $q=$_;
  } else {
    $_ =~ s/^[ \t]+//; $q.=" $_";
  }
}'
```

由于stdout缓存，会有大概率最后几条出不来，（继续执行sql语句就会把这一次没出来的挤出来）

[sqlalchemy官网](http://docs.sqlalchemy.org/en/latest/faq/performance.html)可以设置一些时间，用于监听任务执行的时间。注意

1. before_cursor_execute
2. after_cursor_execute

``` python
from sqlalchemy import event
from sqlalchemy.engine import Engine
import time
import logging

logging.basicConfig()
logger = logging.getLogger("myapp.sqltime")
logger.setLevel(logging.DEBUG)

@event.listens_for(Engine, "before_cursor_execute")
def before_cursor_execute(conn, cursor, statement,
                        parameters, context, executemany):
    conn.info.setdefault('query_start_time', []).append(time.time())
    logger.debug("Start Query: %s", statement)

@event.listens_for(Engine, "after_cursor_execute")
def after_cursor_execute(conn, cursor, statement,
                        parameters, context, executemany):
    total = time.time() - conn.info['query_start_time'].pop(-1)
    logger.debug("Query Complete!")
    logger.debug("Total Time: %f", total)
```

完整的事件文档参考
1. <http://docs.sqlalchemy.org/en/latest/core/events.html>
2. <http://docs.sqlalchemy.org/en/latest/orm/events.html>
3. <http://docs.sqlalchemy.org/en/latest/core/event.html#event-reference>

使用工具[line_profiler](https://github.com/rkern/line_profiler)分析调用时间，使用非常简单，在需要分析的函数加上【@prifile】注解,

若使用ide，会提示注解找不到符号，不用管他

``` python
@profile
def run():
    a = [1]*100
    b = [x*x*x for x in a ]
    c = [x for x in b]
    d = c*2
run()
```

---

安装line_profiler会一并安装一个kernprof.py脚本，需要这个脚本运行我们的程序才会不出错

在程序退出后，-v参数会指示程序将结果输出到stdout，例如
``` bash
Total time: 8.55619 s
File: /home/king/bug/venv/lib/python3.5/site-packages/sqlalchemy/orm/query.py
Function: __iter__ at line 2733

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
2733                                            @profile
2734                                            def __iter__(self):
2735      2176      1140139    524.0     13.3       context = self._compile_context()
2736      2176         3682      1.7      0.0       context.statement.use_labels = True
2737      2176         2284      1.0      0.0       if self._autoflush and not self._populate_existing:
2738      2176        28075     12.9      0.3           self.session._autoflush()
2739      2176      7382008   3392.5     86.3       return self._execute_and_instances(context)
```


参考<https://www.oschina.net/translate/python-performance-analysis?print>

也可以用cProfile来分析，参考

只需要在运行时，加上【-m cProfile】python选项
``` bash
python -m cProfile /home/king/code/bug/main.py
```

然后在需要作分析测试的地方
``` python
cProfile.run('requests.get("http://tech.glowing.com")')  
```

一种更简单的办法
``` python
import cProfile
import io
import pstats
import contextlib

@contextlib.contextmanager
def profiled(limit=20):
    pr = cProfile.Profile()
    pr.enable()
    yield
    pr.disable()
    s = io.StringIO()
    ps = pstats.Stats(pr, stream=s).sort_stats('cumulative')
    ps.print_stats(limit)
    # uncomment this to see who's calling what
    # ps.print_callers()
    print(s.getvalue())
```
``` python
def Test(cars):
    with profiled(limit=30):
        for num in range(1, 20):
            for car in cars:
                car.test()
```

参考

1. <http://docs.sqlalchemy.org/en/latest/faq/performance.html>
2. <http://tech.glowing.com/cn/python-profiling/>
3. <http://ju.outofmemory.cn/entry/284150>
