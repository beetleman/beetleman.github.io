---
title: Clojure vs Python web benchamrk
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

Im web developer and I want to know which tool I shoud use in given situation for backend. I have 3 languages in my belt: Clojure, JavaScript and Python so this post will be about python and clojure because i dont like JS:D.  I will check performance of my favorite python and clojure microframeworks:

- Python
  - [flask]
  - [aiohttp]
- Clojure
  - [luminus]
  - [catacumba]
  - [yada]

I saw a lot of benchmarks and most of them dont check how app scale if we depends on slow service/db. For me its rly important because in microservice word You will olways depend on some kind of service.

# How

# Players

# Results

{% asset_img requests_per_sec.png Request per second %}

{% asset_img transfer.png Transfer %}

{% asset_img latency.png Latency %}

{% asset_img timeouts.png Timeouts %}

{% asset_img all_errors.png All errors %}

# Source Code

You can generate/experiment with data by You own using code from [benchmark] repo.


[benchmark]: https://github.com/beetleman/clojure-vs-python-web-benchmark
[flask]: http://flask.pocoo.org/
[aiohttp]: https://aiohttp.readthedocs.io/en/stable/index.html
[luminus]: http://www.luminusweb.net/
[yada]: https://juxt.pro/yada/index.html
[catacumba]: https://funcool.github.io/catacumba/latest/
