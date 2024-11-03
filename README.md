
## 前言


之前学习和实际生产环境的flask都是用`app.run()`的默认方式启动的，因为只是公司内部服务，请求量不高，一直也没出过什么性能问题。最近接管其它小组的服务时，发现他们的服务使用Gunicorn \+ Flask的方式运行的，本地开发用的gevent的WSGIServer。对于Gunicorn之前只是耳闻，没实际用过，正好捣鼓下看看到底能有多少性能提升。本文简单记录flask在各种配置参数和运行方式的性能，后面也会跟其他语言和框架做个对比。


* python版本：3\.11
* flask版本：3\.0\.3
* Gunicorn：23\.0\.0
* wrk作为性能测试工具
* 运行环境：vbox虚拟机，debian 12, 4C4G的硬件配置


## wrk测试脚本


wrk支持用lua脚本对请求的响应结果进行验证，以下脚本对响应码和响应内容进行校验



```
wrk.method = "GET"
wrk.host = "127.0.0.1:8080"
wrk.path = "/health"
wrk.timeout = 1.0

response = function(status, headers, body)
    if status ~= 200 then
        print("Error: expected 200 but got " .. status)
    end

    if not body:find("ok") then
        print("Error: response does not contain expected content.")
    end
end


```

## Flask框架的测试记录


1. 先测试默认运行方式，且没有sleep的情况下的并发性能。



```
from flask import Flask

app = Flask(__name__)

@app.get("/health")
def health():
    return "ok"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)

```

使用命令`nohup python demo.py > /dev/null`启动，以下为wrk测试结果，可以看到已经出现超时请求。



```
$ wrk -s bm.lua -t 4 -c2000 -d60s http://127.0.0.1:8080/health
Running 1m test @ http://127.0.0.1:8080/health
  4 threads and 2000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    65.47ms   81.07ms   1.99s    97.69%
    Req/Sec   575.99    107.20     1.08k    66.71%
  137538 requests in 1.00m, 22.82MB read
  Socket errors: connect 0, read 114, write 0, timeout 144
Requests/sec:   2288.49
Transfer/sec:    388.86KB

```

2. 还是默认启动方式，增加等待时间，模拟处理任务的时间消耗。后续测试都会增加等待时间。



```
from flask import Flask
from time import sleep

app = Flask(__name__)

@app.get("/health")
def health():
    sleep(0.1)
    return "ok"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)

```

wrk测试结果，不出所料性能会有所下降。



```
$ wrk -s bm.lua -t 4 -c2000 -d60s http://127.0.0.1:8080/health
Running 1m test @ http://127.0.0.1:8080/health
  4 threads and 2000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   201.90ms  239.99ms   2.00s    95.05%
    Req/Sec   479.79    185.87     1.82k    68.19%
  114440 requests in 1.00m, 18.99MB read
  Socket errors: connect 0, read 2, write 0, timeout 1833
Requests/sec:   1904.45
Transfer/sec:    323.62KB

```

3. flask 更新到版本2后支持使用异步函数（需要安装异步相关依赖`python -m pip install -U flask[async]`）



```
from flask import Flask
import asyncio

app = Flask(__name__)

@app.route('/health')
async def health():
    await asyncio.sleep(0.1)
    return "ok"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)

```

wrk测试结果，性能相较于同步函数甚至还下降了，QPS几乎砍半，看来异步版Flask还有待增强。



```
$ wrk -s bm.lua -t 4 -c2000 -d60s http://127.0.0.1:8080/health
Running 1m test @ http://127.0.0.1:8080/health
  4 threads and 2000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   275.49ms  190.22ms   2.00s    95.46%
    Req/Sec   242.86    104.25   720.00     65.91%
  57896 requests in 1.00m, 9.61MB read
  Socket errors: connect 0, read 48, write 0, timeout 611
Requests/sec:    964.31
Transfer/sec:    163.86KB

```

4. 接管的Flask应用在本地使用gevent的WSGIServer运行，所以也来试试。



```
from gevent.pywsgi import WSGIServer
from flask import Flask
from time import sleep

app = Flask(__name__)

@app.route('/health')
def health():
    sleep(0.1)
    return "ok"

if __name__ == "__main__":
    http_server = WSGIServer(('0.0.0.0', 8080), app)
    http_server.serve_forever()

```

wrk测试结果，惨不忍睹，像是单线程在挨个处理请求，每个请求都会阻塞住。



```
$ wrk -s bm.lua -t 4 -c2000 -d60s http://127.0.0.1:8080/health
Running 1m test @ http://127.0.0.1:8080/health
  4 threads and 2000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   185.36ms  221.17ms   1.93s    96.30%
    Req/Sec     5.54      3.39    10.00     46.55%
  592 requests in 1.00m, 67.64KB read
  Socket errors: connect 0, read 0, write 0, timeout 322
Requests/sec:      9.85
Transfer/sec:      1.13KB

```

5. 按网上搜的结果加上了monkey patch



```
from gevent.pywsgi import WSGIServer
from gevent import monkey
from flask import Flask
from time import sleep

monkey.patch_all()

app = Flask(__name__)

@app.route('/health')
def health():
    sleep(0.1)
    return "ok"

if __name__ == "__main__":
    http_server = WSGIServer(('0.0.0.0', 8080), app)
    http_server.serve_forever()

```

wrk测试结果，加上monkey patch后似乎也没什么作用。



```
$ wrk -s bm.lua -t 4 -c2000 -d60s http://127.0.0.1:8080/health
Running 1m test @ http://127.0.0.1:8080/health
  4 threads and 2000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   182.89ms  209.48ms   1.82s    96.07%
    Req/Sec     5.55      3.50    10.00     47.89%
  592 requests in 1.00m, 67.64KB read
  Socket errors: connect 0, read 0, write 0, timeout 312
Requests/sec:      9.85
Transfer/sec:      1.13KB

```

6. 正式上gunicorn，代码没有任何改动，也不需要引用gevent的WSGServer。



```
from flask import Flask
from time import sleep

app = Flask(__name__)

@app.get("/health")
def health():
    sleep(0.1)
    return "ok"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)

```

运行命令。指定`-k gevent`。`demo:app`中的`demo`是代码文件名。`--worker-connections`默认为1000



```
gunicorn demo:app -b 0.0.0.0:8080 -w 4 -k gevent --worker-connections 2000

```

wrk测试结果。性能相较于默认启动方式有了接近10倍的提升，请求响应时间也很稳定，最大响应时间也只有310\.48。



```
$ wrk -s bm.lua -t 4 -c2000 -d60s http://127.0.0.1:8080/health
Running 1m test @ http://127.0.0.1:8080/health
  4 threads and 2000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   126.18ms    9.52ms 310.48ms   84.68%
    Req/Sec     3.98k   165.70     4.53k    77.34%
  948506 requests in 1.00m, 143.83MB read
Requests/sec:  15799.31
Transfer/sec:      2.40MB

```

## 其它框架和语言


在t4c2000的wrk配置下，flask\+unicorn的每个进程基本都占用了85\+%的CPU，再提高就得加CPU核心数了，不过这样的性能已经能满足公司内部服务的需求了，而且实际业务中，短板更可能是网络IO。


这里也测测其它语言和框架，看看Flask在Gunicorn的加持下能否打出python的牌面。


### Golang


上来先试试最熟悉的Go， version: 1\.22\.4，使用标准库。（编译打包出来就能直接运行，不需要jvm这样的虚拟机，也不需要python这样的解释器，更不需要docker这样的容器运行时，特喜欢Go这一点）



```
package main

import (
	"fmt"
	"net/http"
	"time"
)

func MyHandler(w http.ResponseWriter, r *http.Request) {
	time.Sleep(time.Millisecond * 100)
	w.Write([]byte("ok"))
}

func main() {
	http.HandleFunc("/health", MyHandler)
	err := http.ListenAndServe("0.0.0.0:8080", nil)
	if err != nil {
		fmt.Println(err)
	}
}


```

wrk结果如下，请求量是目前测试以来第一个突破百万，而且也没有timeout的出现。使用top观察资源消耗，CPU只占用了约30%，而且还只有一个进程。



```
$ wrk -s bm.lua -t 4 -c2000 -d60s http://127.0.0.1:8080/health
Running 1m test @ http://127.0.0.1:8080/health
  4 threads and 2000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   101.56ms    1.48ms 121.62ms   77.58%
    Req/Sec     4.94k   191.07     5.05k    93.49%
  1180108 requests in 1.00m, 132.80MB read
Requests/sec:  19643.94
Transfer/sec:      2.21MB

```

不断加大连接，直到系统平均负载到达4（虚拟机CPU核心数为4）。连接数加了10倍，QPS差不多也是10倍于Flask \+ Gunicorn。这时候实际上wrk也占用了不少CPU资源，服务端的性能并没到瓶颈。



```
$ wrk -s bm.lua -t 4 -c20000 -d60s http://127.0.0.1:8080/health
Running 1m test @ http://127.0.0.1:8080/health
  4 threads and 20000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   167.56ms   43.45ms 397.95ms   60.37%
    Req/Sec    29.48k     8.84k   48.26k    63.05%
  6867733 requests in 1.00m, 772.85MB read
Requests/sec: 114258.70
Transfer/sec:     12.86MB

```

### FastAPI


Go的性能已经很不错了，就性能来说还不是Flask\+Gunicorn能媲美的。再来试试号称性能并肩Go的FastAPI（官网features里面写的）。FastAPI版本：0\.115\.4


纯uvicorn启动，用的是同步函数。



```
from fastapi import FastAPI
import uvicorn
from fastapi.responses import PlainTextResponse
from time import sleep

app = FastAPI()

@app.get("/health")
def index():
    sleep(0.1)
    return PlainTextResponse(status_code=200,content="ok")

if __name__ == '__main__':
    uvicorn.run(app, host="127.0.0.1", port=8080, access_log=False)

```

wrk测试结果，可以看到相当低下，甚至还不如flask的默认运行方式，超时请求数都过2w了。



```
$ wrk -s bm.lua -t 4 -c2000 -d60s http://127.0.0.1:8080/health
Running 1m test @ http://127.0.0.1:8080/health
  4 threads and 2000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.22s   438.35ms   1.95s    60.00%
    Req/Sec   115.03     57.31   410.00     86.69%
  23494 requests in 1.00m, 3.02MB read
  Socket errors: connect 0, read 0, write 0, timeout 22894
Requests/sec:    390.92
Transfer/sec:     51.54KB

```

改用异步函数再试试。



```
from fastapi import FastAPI
import uvicorn
from fastapi.responses import PlainTextResponse
from time import sleep
import asyncio

app = FastAPI()

@app.get("/health")
async def health():
    await asyncio.sleep(0.1)
    return PlainTextResponse(status_code=200,content="ok")

if __name__ == '__main__':
    uvicorn.run(app, host="127.0.0.1", port=8080, access_log=False)

```

wrk测试结果，可以看到性能好很多了，而且没有timeout。QPS是Flask默认启动方式的2倍，但实际性能应该不止2倍。



```
$ wrk -s bm.lua -t 4 -c2000 -d60s http://127.0.0.1:8080/health
Running 1m test @ http://127.0.0.1:8080/health
  4 threads and 2000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   484.53ms   63.18ms 654.48ms   61.26%
    Req/Sec     1.09k   698.29     3.13k    63.78%
  246744 requests in 1.00m, 31.77MB read
Requests/sec:   4106.80
Transfer/sec:    541.43KB

```

uvicorn支持指定worker数，这里设置为CPU核心数。



```
from fastapi import FastAPI
import uvicorn
from fastapi.responses import PlainTextResponse
from time import sleep
import asyncio

app = FastAPI()

@app.get("/health")
async def health():
    await asyncio.sleep(0.1)
    return PlainTextResponse(status_code=200,content="ok")

if __name__ == '__main__':
    uvicorn.run(app="demo2:app", host="127.0.0.1", port=8080, access_log=False, workers=4)

```

wrk测试结果，响应时间还是非常稳的，完全没有timeout的情况，延迟还更低。



```
$ wrk -s bm.lua -t 4 -c2000 -d60s http://127.0.0.1:8080/health
Running 1m test @ http://127.0.0.1:8080/health
  4 threads and 2000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   164.52ms   13.57ms 273.27ms   73.49%
    Req/Sec     3.05k   544.20     4.64k    68.67%
  727517 requests in 1.00m, 93.73MB read
Requests/sec:  12123.17
Transfer/sec:      1.56MB

```

gunicorn也支持uvicorn，看看fastapi在gunicorn的加持下会有怎样的性能表现。



```
gunicorn demo2:app -b 127.0.0.1:8080 -w 4 -k uvicorn.workers.UvicornWorker --worker-connections 2000

```

wrk测试结果，相较于unicorn运行方式，性能提升并不多。



```
$ wrk -s bm.lua -t 4 -c2000 -d60s http://127.0.0.1:8080/health
Running 1m test @ http://127.0.0.1:8080/health
  4 threads and 2000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   146.43ms   21.21ms 263.13ms   69.52%
    Req/Sec     3.43k   508.71     4.88k    71.40%
  818281 requests in 1.00m, 105.35MB read
Requests/sec:  13620.16
Transfer/sec:      1.75MB

```

### Sanic


之前用过一段时间Sanic，也是个python异步框架，版本：24\.6\.0



```
from sanic import Sanic
from sanic.response import text
import asyncio

app = Sanic("HelloWorld")

@app.get("/health")
async def hello_world(request):
    await asyncio.sleep(0.1)
    return text("ok")

if __name__ == "__main__":
    app.run(host="127.0.0.1", port=8080, fast=True, debug=False, access_log=False)

```

wrk测试结果。虽然QPS比FastAPI高，但是有timeout的情况，不是很稳定。



```
$ wrk -s bm.lua -t 4 -c2000 -d60s http://127.0.0.1:8080/health
Running 1m test @ http://127.0.0.1:8080/health
  4 threads and 2000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   104.05ms   45.30ms   1.82s    99.59%
    Req/Sec     4.84k   538.65     5.07k    95.98%
  1154792 requests in 1.00m, 115.64MB read
  Socket errors: connect 0, read 0, write 0, timeout 88
Requests/sec:  19218.08
Transfer/sec:      1.92MB

```

### Openresty


openresty基于nginx，通过集成lua，也可以用来写api。配置如下，只是增加了一个location，稍微调整下nginx的参数



```
worker_processes  auto;
worker_cpu_affinity auto;

events {
    worker_connections  65535;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    access_log off;
    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       8080 deferred;
        server_name  localhost;
        location /health {
            content_by_lua_block {
                ngx.sleep(0.1)
                ngx.print("ok")
            }
        }
        location / {
            root   html;
            index  index.html index.htm;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}

```

wrk测试结果，和Go语言相当。



```
$ wrk -s bm.lua -t 4 -c2000 -d60s http://127.0.0.1:8080/health
Running 1m test @ http://192.168.0.201:8080/health
  4 threads and 2000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   101.43ms    1.62ms 136.59ms   88.97%
    Req/Sec     4.94k   213.51     5.33k    92.65%
  1178854 requests in 1.00m, 211.36MB read
Requests/sec:  19619.86
Transfer/sec:      3.52MB

```

`top`观察openresty的cpu占用并不高，加大连接再试试。连接数达到25000后，系统平均负载已经基本满了，而且wrk也占用了不少CPU资源。和Go差不多，并没有到服务端的性能瓶颈，而是受到系统资源限制。



```
$ wrk -s bm.lua -t 4 -c25000 -d60s http://127.0.0.1:8080/health
Running 1m test @ http://127.0.0.1:8080/health
  4 threads and 25000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   149.17ms   36.61ms 659.40ms   71.22%
    Req/Sec    41.00k     6.64k   59.03k    65.95%
  9330399 requests in 1.00m, 1.63GB read
Requests/sec: 155277.58
Transfer/sec:     27.84MB

```

## 小结


整理下测试数据汇总成表格如下




| 项目 | 总请求量 | 每秒请求量 | 平均响应时间 | 最大响应时间 | 备注 |
| --- | --- | --- | --- | --- | --- |
| Flask\-no sleep | 137538 | 2288\.49 | 65\.47ms | 1\.99s | 有响应超时情况 |
| Flask\-同步方式 | 114440 | 1904\.45 | 201\.90ms | 2\.00s | 有响应超时情况 |
| Flask\-异步函数 | 57896 | 964\.31 | 275\.49ms | 2\.00s | 有响应超时情况 |
| Flask\+gevent | 592 | 9\.85 | 185\.36ms | 1\.93s | 有响应超时情况 |
| Flask\+gevent(monkeypatch) | 592 | 9\.85 | 182\.89ms | 1\.82s | 有响应超时情况 |
| Flask\+gevent\+unicorn | 948506 | 15799\.31 | 126\.18ms | 310\.48ms |  |
| Golang | 1180108 | 19643\.94 | 101\.56ms | 121\.62ms |  |
| Golang | 6867733 | 114258\.70 | 167\.56ms | 397\.95ms | wrk的配置为t4c20000 |
| FastAPI\-同步函数 | 23494 | 390\.92 | 1\.22s | 1\.95s | 有响应超时情况 |
| FastAPI\-异步函数 | 246744 | 4106\.80 | 484\.53ms | 654\.48ms |  |
| FastAPI\-多worker | 727517 | 12123\.17 | 164\.52ms | 273\.27ms |  |
| FastAPI\+Gunicorn | 818281 | 13620\.16 | 146\.43ms | 273\.27ms |  |
| Sanic | 1154792 | 19218\.08 | 104\.05ms | 1\.82s | 有响应超时情况，不是很稳定 |
| OpenResty | 1178854 | 19619\.86 | 101\.43ms | 136\.59ms |  |
| OpenResty | 9330399 | 155277\.58 | 149\.17ms | 659\.40ms | wrk的配置为t4c25000 |


根据测试结果，测试的三个Python Web框架中，Flask\+gevent\+unicorn综合最佳，不低的QPS，而且没有请求超时的情况，也不需要将代码修改成异步方式。Sanic的QPS虽高，但是有响应超时的情况，说明并不稳定，而且代码需要是异步的。FastAPI\+Gunicorn的表现也不差，在不使用Gunicorn的情况下也能提供不错的性能，但代码同样需要改成异步方式。对于Sanic和FastAPI，Gunicorn的加持并不必要，而Gunicorn对Flask的性能提升至少7倍，而且能避免请求超时的情况，生产环境下应该尽量使用Gunicorn来运行Flask。


Go比各个Python框架的性能都更好，资源占用也更低，运行方式还更简单，不需要依赖编程语言环境和其他组件，非要说缺点的话就是开发没有Python快。


OpenResty的性能在测试中是最高的，主要是nginx本身性能良好。缺点是开发更麻烦。虽然是用lua开发，但lua作为动态语言，既不如Python极其灵活，还有动态语言本身代码不够清晰的缺点。以前尝试过用openresty实现一个crud服务，后来连自己都懒得维护就放弃了，干脆只用来当网关。


鱼与熊掌不可兼得，开发速度跟运行速度往往相斥，除非代码以后都是AI来写。就公司目前这服务的使用情况来说，Flask\+Gunicorn的性能已经足够，还不需要改代码，实乃社畜良伴。而且现在啥都上k8s了，服务扩展也简单，性能不够就加实例嘛 😃


 本博客参考[milou加速器](https://jiechuangmoxing.com)。转载请注明出处！
