---
title: Clojure vs Python web benchmark
tags:
    - python
    - clojure
    - yada
    - luminus
    - flask
    - aiohttp
    - catacumba
---

# Why?

I'm a web developer and I want to know which tool I should use in a given situation in the back-end. I have 3 languages in my arsenal: [Clojure], JavaScript and [Python] so this post will be about [Python] and Clojure because i don't like JS :D.  I will check the performance of my favorite [python] and clojure micro-frameworks:

- [Python]
  - [flask]
  - [aiohttp]
- [Clojure]
  - [luminus]
  - [catacumba]
  - [yada]

I saw a lot of benchmarks and most of them don't check how app scale if we depends on slow service/db. For me it's really important because in micro-services world one will always depend on othe services.

# How?

For this benchmark I implemented simple hello-word application in each micro-framework but with one twist. Every app accept GET param `delay` and this value is used to simulate delay. For implementing delay I used best method for each platform but I thing async [Python] have advantage in this example because `await asyncio.sleep(delay)` is a lot quicker than [Clojure] `(Thread/sleep delay)` and it works more like synchronous `time.sleep(delay)` from synchronous [Python].

# Implementations

I skip details and provide only router/response part of each app.

## Flask

[Flask] is synchronous [Python] micro-framework and implementing hello-world as easy as:

```python
app = Flask(__name__)


@app.route("/")
def hello():
    delay = request.args.get('delay')
    time.sleep(toInt(delay)/1000)
    return "hello world!"
```
## Aiohttp

[Aiohttp] is asynchronous [Python] micro-framework and it's somehow similar to [Flask]:

```python
async def index(request):
    delay = request.GET.get('delay')
    await asyncio.sleep(toInt(delay)/1000)
    return web.Response(text="hello world!")


app = web.Application()
app.router.add_get('/', index)
```

## Luminus

[Luminus] is more project template than micro-framework in my opinion and generate a lot of unnecessary files but it's extremely simple to start:

```bash
lein new luminus <your project name> +service
```
and we have service with swagger and docker file. As default web server [Luminus] use [Immutant] with is based on [Undertow]. Implementation:

```clojure
(defn hello-handler [delay]
  (Thread/sleep delay)
  "hello world!")

(defapi service-routes
  {:swagger {:ui "/swagger-ui"
             :spec "/swagger.json"
             :data {:info {:version "1.0.0"
                           :title "Sample API"
                           :description "Sample Services"}}}}

  (GET "/" []
       :query-params [{delay :- Long 0}]
       :return String
       (ok (hello-handler delay))))
```

it's more LOC than [Python] but we have field validation and swagger.

## Catacumba

[Catacumba] is a new player in [Clojure] world. It's based on [Ratpack] and It's asynchronous. Implementation:

```clojure
(defn hello-handler [ctx]
  (let [delay (-> ctx :query-params :delay str->int)
        result (d/future
                 (Thread/sleep delay)
                 "hello world!")]
    result))

(defn routes [base]
  (ct/routes [
              [:get "" hello-handler]]))
```
It has less lines than [Luminus] implementation but saves functionality as [Python] implementations(no validation, no swagger).

## Yada

[Yada] yada yada... I really like idea behind yada and its some how similar to [Aiohttp] and [Flask]. But its asynchronous by design and have goal to be compliance with HTTP standards. Implementation:

```clojure
(defn hello-response [ctx]
  (let [delay  (get-in ctx [:parameters :query :delay])
        result (d/future
                 (Thread/sleep delay)
                 "hello world!")]
    result))

(def hello-resource
  (yada/resource
   {:methods {:get
              {:parameters {:query {:delay s/Int}}
               :produces   {:media-type "text/plain"}
               :response   hello-response}}}))

(defn routes [base]
  [base
   [
    ["" hello-resource]
    [true (yada/as-resource nil)]]])
```
Again more code then [Python] but code is more declarative in my opinion and we have query validation build-in.

# Benchmark

For benchmark I used [wrk] with this set op parameters:

```bash
wrk -t12 -c1000 -d60s -s report.lua http://localhost:8080/?delay=$delay
```

Where `report.lua` is little script generating csv from test and it's available in [benchmark] repo. Before each test [wrk] is used without lua script for warm up [JVM] (its not needed for [Python]).

[Python] implementations were run on [gunicorn] using `number_of_cpu_cores*2` workers. This should give [Python] equal chance vs [JVM].

# Results

For the first run I got surprising results because [Yada] was faster than [Flask] only:

## Aiohttp

```
Running 1m test @ http://localhost:8080/?delay=0
  12 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    77.29ms   43.31ms 294.88ms   65.91%
    Req/Sec     1.10k   369.46     3.74k    71.47%
  786155 requests in 1.00m, 122.96MB read
Requests/sec:  13084.33
Transfer/sec:      2.05MB
```

## Catacumba

```
Running 1m test @ http://localhost:8080/?delay=0
  12 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    34.96ms   30.03ms 965.51ms   91.57%
    Req/Sec     2.58k   370.05     5.92k    88.60%
  1846723 requests in 1.00m, 158.51MB read
  Non-2xx or 3xx responses: 1
Requests/sec:  30736.96
Transfer/sec:      2.64MB
```

## Flask

```
flask-app
Running 1m test @ http://localhost:8080/?delay=0
  12 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   111.64ms  132.68ms   1.99s    96.28%
    Req/Sec    97.81    131.25     1.13k    93.16%
  41315 requests in 1.00m, 6.78MB read
  Socket errors: connect 0, read 12, write 0, timeout 594
Requests/sec:    687.52
Transfer/sec:    115.48KB
```

## Luminus (Immutant)

```
Running 1m test @ http://localhost:8080/?delay=0
  12 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    94.16ms  133.16ms   1.58s    97.87%
    Req/Sec     1.09k   159.41     2.13k    86.47%
  732819 requests in 1.00m, 248.80MB read
  Socket errors: connect 0, read 0, write 0, timeout 996
Requests/sec:  12193.31
Transfer/sec:      4.14MB
```
## Yada

```
Running 1m test @ http://localhost:8080/?delay=0
  12 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   151.23ms  235.77ms   1.99s    88.18%
    Req/Sec   750.00    288.08     2.09k    71.45%
  532697 requests in 1.00m, 128.68MB read
  Socket errors: connect 0, read 0, write 0, timeout 658
  Non-2xx or 3xx responses: 243
Requests/sec:   8867.34
Transfer/sec:      2.14MB
```

In my opinion all timeouts are because [Yada] has many interceptors (similar to middlewares in other framework) by default so It does more than others in this example. I did more tests with delays from 0 to 400ms. I present only charts because they are more readable but all data are available in [repo]

{% asset_img requests_per_sec.png Request per second %}

During tests with bigger delay interceptors overload are less visible and around 400ms are not visible at all. But still [catacumba] is the beast if we care only about performance and [Aiohttp] with delays is as good as [catacumba].. big surprise!!

{% asset_img latency.png Latency %}

If delay grows only [Catacumba], [Yada] and [Aiohttp] have acceptable latency.

{% asset_img all_errors.png All errors %}

level of HTTP errors for the big 3 are on a similar level if delays grows.

# Conclusion

If I must choose I will pick between [Yada], [Catacumba] and [Aiohttp]. [Catacumba] is extremely fast but new and offer much less than [Yada] in my opinion so for me It is more about [Yada] vs [Aiohttp]. For simple things I think [Aiohttp] is the best choice possible but for bigger maybe [Yada] is better. I will try to implement more real live examples in both and check if [Aiohttp] is still the winner in terms of performance and code simplicity.

# Source Code

You can generate/experiment with data on your own using code from [benchmark] repo.


[benchmark]: https://github.com/beetleman/clojure-vs-Python-web-benchmark
[repo]: https://github.com/beetleman/clojure-vs-Python-web-benchmark
[flask]: http://flask.pocoo.org/
[aiohttp]: https://aiohttp.readthedocs.io/en/stable/index.html
[luminus]: http://www.luminusweb.net/
[yada]: https://juxt.pro/yada/index.html
[catacumba]: https://funcool.github.io/catacumba/latest/
[ratpack]: https://ratpack.io/
[immutant]: http://immutant.org/
[undertow]: http://undertow.io/
[Clojure]: https://clojure.org
[Python]: https://www.python.org
[aleph]: http://aleph.io/
[wrk]: https://github.com/wg/wrk
[jvm]: http://openjdk.java.net
[gunicorn]: http://gunicorn.org
