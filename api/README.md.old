# Currency Rates Service -  application for fetching currency rates and viewing them via UI, optimized for high load.
## Goal 1500 RPS (from unique user), HTTP request duration ~ 1000ms, 1800-2000 VUs load

## Backend Deploy (simple, no Laravel octane, no x-mutagen)

```
cd api
make build
make generate-env
make install
```

### There are test credentials in .env, and maybe they are currently invalid.
- Feel free to register at [Open Exchange Rates](https://openexchangerates.org) to get own OER_KEY_ID=

## Test User credentials
```
test0@example.com
password
```

## Optimization

- Redis/APC caches used for repository
- Opcache enabled
- Optimize docker-compose.yml and nginx settings
- Laravel caching: php artisan optimize

## Install before K6 utility on your environment and run for stress-test
```
cd api
k6 run load_test.js
```

### Macbook Pro M1 16Gb RAM 10 CPU environment, Docker - 14 Gb RAM, 8 CPU, Swap 512 Mb
### The final results with current configuration that could be achieved with A/B tests was: 
- 838 RPS for Redis cache, HTTP request duration - 1.02s, 800 VUs (virtual users per second) load 
```
   █ THRESHOLDS 

    http_req_duration
    ✓ 'p(90)<3000' p(90)=1.02s


  █ TOTAL RESULTS 

    checks_total.......: 352084  838.285372/s
    checks_succeeded...: 100.00% 352084 out of 352084
    checks_failed......: 0.00%   0 out of 352084

    ✓ status is 200

    HTTP
    http_req_duration..............: avg=817.56ms min=4.79ms  med=912.23ms max=2.21s p(90)=1.02s p(95)=1.07s
      { expected_response:true }...: avg=817.56ms min=4.79ms  med=912.23ms max=2.21s p(90)=1.02s p(95)=1.07s
    http_req_failed................: 0.00%  0 out of 352084
    http_reqs......................: 352084 838.285372/s

    EXECUTION
    iteration_duration.............: avg=3.28s    min=34.68ms med=3.72s    max=5.89s p(90)=3.98s p(95)=4.13s
    iterations.....................: 88021  209.571343/s
    vus............................: 1      min=1           max=800
    vus_max........................: 800    min=800         max=800

    NETWORK
    data_received..................: 1.5 GB 3.6 MB/s
    data_sent......................: 71 MB  168 kB/s




running (7m00.0s), 000/800 VUs, 88021 complete and 0 interrupted iterations
default ✓ [======================================] 000/800 VUs  7m0s

```
- 806 RPS for Apc cache, HTTP request duration - 1.07s, 800 VUs load
```
   █ THRESHOLDS 

    http_req_duration
    ✓ 'p(90)<3000' p(90)=1.07s


  █ TOTAL RESULTS 

    checks_total.......: 338672  806.284131/s
    checks_succeeded...: 100.00% 338672 out of 338672
    checks_failed......: 0.00%   0 out of 338672

    ✓ status is 200

    HTTP
    http_req_duration..............: avg=850.21ms min=5.26ms  med=932.92ms max=2.66s p(90)=1.07s p(95)=1.13s
      { expected_response:true }...: avg=850.21ms min=5.26ms  med=932.92ms max=2.66s p(90)=1.07s p(95)=1.13s
    http_req_failed................: 0.00%  0 out of 338672
    http_reqs......................: 338672 806.284131/s

    EXECUTION
    iteration_duration.............: avg=3.41s    min=35.77ms med=3.83s    max=6.21s p(90)=4.15s p(95)=4.31s
    iterations.....................: 84668  201.571033/s
    vus............................: 1      min=1           max=800
    vus_max........................: 800    min=800         max=800

    NETWORK
    data_received..................: 1.4 GB 3.4 MB/s
    data_sent......................: 68 MB  162 kB/s




running (7m00.0s), 000/800 VUs, 84668 complete and 0 interrupted iterations
default ✓ [======================================] 000/800 VUs  7m0s
```

### Next optimization level is minimize Sql queries from Sanctum removing its tokens to a cache and make horizontal scaling

### Conducted the next level optimization: 
 - Sanctum: Redis cache, asynchronous tokens updates, race elimination 
 - APCu: Main repository cache, session cache 
 - PHP-FPM: Max value pm.max_children = 200.

#### Now the result:

```
█ THRESHOLDS 

    http_req_duration
    ✓ 'p(90)<3000' p(90)=1.4s


  █ TOTAL RESULTS 

    checks_total.......: 328596  782.344888/s
    checks_succeeded...: 100.00% 328596 out of 328596
    checks_failed......: 0.00%   0 out of 328596

    ✓ status is 200

    HTTP
    http_req_duration..............: avg=1.09s min=4.96ms  med=1.2s  max=2.71s p(90)=1.4s  p(95)=1.48s
      { expected_response:true }...: avg=1.09s min=4.96ms  med=1.2s  max=2.71s p(90)=1.4s  p(95)=1.48s
    http_req_failed................: 0.00%  0 out of 328596
    http_reqs......................: 328596 782.344888/s

    EXECUTION
    iteration_duration.............: avg=4.39s min=32.34ms med=4.94s max=6.78s p(90)=5.41s p(95)=5.6s 
    iterations.....................: 82149  195.586222/s
    vus............................: 2      min=2           max=1000
    vus_max........................: 1000   min=1000        max=1000

    NETWORK
    data_received..................: 1.4 GB 3.4 MB/s
    data_sent......................: 66 MB  157 kB/s




running (7m00.0s), 0000/1000 VUs, 82149 complete and 0 interrupted iterations
default ✓ [======================================] 0000/1000 VUs  7m0s
```

- RPS ≈ 780 is stable at 1000 VU and at the same time no fails (0% http_req_failed)
- Latency p90 = 1.4s - means the system can handle the load, but has a limitation.
- 1000 VUs load

### Conclusions

The bottleneck has been found – RPS has hit the performance limit of the current stack.
That is, the system handles the load without errors, but the increase in the number of users does not increase RPS - this means the CPU-bound or IO-bound limit has already been reached.

RPS does not grow, but is stable, so the system is scalable, but does not accelerate further on one or more instances.

There are no reason to make horizontal scale with Nginx balancer on one instance (Macbook M1) - because we reached max 8 CPU limit with 1000 VUs, pm.max_children = 200 for Php-fpm and macOS uses the virtualization but not the containerization

### Next optimization level is switch to Laravel Octane (swoole server)

- And next record 874 RPS with APCu cache with Laravel Octane (swoole server), but it's wrong way after detail review of cache state because APCu in not shared cache between Octane workers instances
- HTTP request duration - 1.19s, 1000 VUs load
```
█ THRESHOLDS

    http_req_duration
    ✓ 'p(90)<3000' p(90)=1.19s


█ TOTAL RESULTS

    checks_total.......: 367316  874.552963/s
    checks_succeeded...: 100.00% 367316 out of 367316
    checks_failed......: 0.00%   0 out of 367316

    ✓ status is 200

    HTTP
    http_req_duration..............: avg=980.83ms min=3.37ms  med=1.06s max=1.78s p(90)=1.19s p(95)=1.52s
      { expected_response:true }...: avg=980.83ms min=3.37ms  med=1.06s max=1.78s p(90)=1.19s p(95)=1.52s
    http_req_failed................: 0.00%  0 out of 367316
    http_reqs......................: 367316 874.552963/s

    EXECUTION
    iteration_duration.............: avg=3.93s    min=25.78ms med=4.33s max=6.17s p(90)=4.86s p(95)=4.96s
    iterations.....................: 91829  218.638241/s
    vus............................: 2      min=2           max=1000
    vus_max........................: 1000   min=1000        max=1000

    NETWORK
    data_received..................: 1.5 GB 3.6 MB/s
    data_sent......................: 74 MB  175 kB/s




running (7m00.0s), 0000/1000 VUs, 91829 complete and 0 interrupted iterations
default ✓ [======================================] 0000/1000 VUs  7m0s
```

So APCu replaced by Memcached shared cache (Laravel Octane swoole), and we have next record: 
929 RPS, HTTP request duration - 1.13s, 1000 VUs load

```
█ THRESHOLDS

    http_req_duration
    ✓ 'p(90)<3000' p(90)=1.13s


█ TOTAL RESULTS

    checks_total.......: 390220  929.050838/s
    checks_succeeded...: 100.00% 390220 out of 390220
    checks_failed......: 0.00%   0 out of 390220

    ✓ status is 200

    HTTP
    http_req_duration..............: avg=922.91ms min=2.88ms  med=1.01s max=1.66s p(90)=1.13s p(95)=1.42s
      { expected_response:true }...: avg=922.91ms min=2.88ms  med=1.01s max=1.66s p(90)=1.13s p(95)=1.42s
    http_req_failed................: 0.00%  0 out of 390220
    http_reqs......................: 390220 929.050838/s

    EXECUTION
    iteration_duration.............: avg=3.7s     min=23.57ms med=4.12s max=5.8s  p(90)=4.59s p(95)=4.67s
    iterations.....................: 97555  232.26271/s
    vus............................: 1      min=1           max=1000
    vus_max........................: 1000   min=1000        max=1000

    NETWORK
    data_received..................: 1.6 GB 3.8 MB/s
    data_sent......................: 78 MB  186 kB/s




running (7m00.0s), 0000/1000 VUs, 97555 complete and 0 interrupted iterations
default ✓ [======================================] 0000/1000 VUs  7m0s
```

- Up to 1500VUs with same configuration - Conclusions - only 4.5 from 8 CPU used, nginx balancer on 2-3-4 instances had no effect, even worse than expected
- Raised octane workers manually to command: php artisan octane:start --server=swoole --host=0.0.0.0 --port=${DOCKER_OCTANE_PORT} --workers=12 --task-workers=4 --max-requests=2000
- Next record - is 941 RPS but avg HTTP request duration - 1.64s, 1500 VUs load
- Any other tunes had no effect, and then I decided to turn off nginx on the next load test
```
█ THRESHOLDS

    http_req_duration
    ✓ 'p(90)<3000' p(90)=1.64s


█ TOTAL RESULTS

    checks_total.......: 395488  941.626007/s
    checks_succeeded...: 100.00% 395488 out of 395488
    checks_failed......: 0.00%   0 out of 395488

    ✓ status is 200

    HTTP
    http_req_duration..............: avg=1.36s min=2.86ms  med=1.53s max=2.28s p(90)=1.64s p(95)=1.89s
      { expected_response:true }...: avg=1.36s min=2.86ms  med=1.53s max=2.28s p(90)=1.64s p(95)=1.89s
    http_req_failed................: 0.00%  0 out of 395488
    http_reqs......................: 395488 941.626007/s

    EXECUTION
    iteration_duration.............: avg=5.48s min=22.57ms med=6.2s  max=7.73s p(90)=6.66s p(95)=6.75s
    iterations.....................: 98872  235.406502/s
    vus............................: 3      min=3           max=1500
    vus_max........................: 1500   min=1500        max=1500

    NETWORK
    data_received..................: 1.6 GB 3.9 MB/s
    data_sent......................: 79 MB  189 kB/s




running (7m00.0s), 0000/1500 VUs, 98872 complete and 0 interrupted iterations
default ✓ [======================================] 0000/1500 VUs  7m0s
```

```
*******@MacBook-Pro-**** currency-rate % docker stats --no-stream
CONTAINER ID   NAME                                   CPU %     MEM USAGE / LIMIT     MEM %     NET I/O           BLOCK I/O     PIDS
6d2f71f9a383   currency-rates-fetching-nginx-octane   45.66%    251.1MiB / 13.65GiB   1.80%     1.5GB / 1.55GB    0B / 36.5MB   9
17b614dc1e86   currency-rates-fetching-queue          69.05%    32.59MiB / 13.65GiB   0.23%     618MB / 499MB     0B / 0B       1
2ee9e9f86613   api-php-worker-1                       314.01%   336.5MiB / 13.65GiB   2.41%     1.91GB / 2.12GB   0B / 4.1kB    25
13f9921ec37b   currency-rates-fetching-redis          36.56%    149.2MiB / 13.65GiB   1.07%     1.08GB / 1.65GB   0B / 0B       7
1eb8cc77a760   currency-rates-fetching-mysql          15.44%    381.4MiB / 13.65GiB   2.73%     53.1MB / 389MB    0B / 483kB    47
db21e55401a9   currency-rates-fetching-memcached      13.09%    9.137MiB / 13.65GiB   0.07%     171MB / 603MB     0B / 0B       10
```

Results without nginx with one Laravel Octane worker - next achievement: 
1116 RPS, HTTP request duration - 1.39s, 1500 VUs load -  not bad, but we can better

```
█ THRESHOLDS

    http_req_duration
    ✓ 'p(90)<3000' p(90)=1.39s


█ TOTAL RESULTS

    checks_total.......: 468760  1116.063279/s
    checks_succeeded...: 100.00% 468760 out of 468760
    checks_failed......: 0.00%   0 out of 468760

    ✓ status is 200

    HTTP
    http_req_duration..............: avg=1.15s min=2.96ms  med=1.28s max=2.13s p(90)=1.39s p(95)=1.74s
      { expected_response:true }...: avg=1.15s min=2.96ms  med=1.28s max=2.13s p(90)=1.39s p(95)=1.74s
    http_req_failed................: 0.00%  0 out of 468760
    http_reqs......................: 468760 1116.063279/s

    EXECUTION
    iteration_duration.............: avg=4.62s min=23.39ms med=5.2s  max=6.18s p(90)=5.73s p(95)=5.79s
    iterations.....................: 117190 279.01582/s
    vus............................: 3      min=3           max=1500
    vus_max........................: 1500   min=1500        max=1500

    NETWORK
    data_received..................: 1.9 GB 4.6 MB/s
    data_sent......................: 94 MB  224 kB/s




running (7m00.0s), 0000/1500 VUs, 117190 complete and 0 interrupted iterations
default ✓ [======================================] 0000/1500 VUs  7m0s
```

After several sleepless nights and attempts to scale to several octane instances on lightweight HAProxy, I hit the ceiling - 1000 RPS, 1.6 s http response on 1500 VUs load and no more. I realized that I forgot something important. And yes, I did. This is a MacBook M1 and this is a local machine, Docker and not balancing between several servers. So my answer was... x-mutagen and disabling any proxies and balancers. I hope you read this far :)

**🔥 And here is the long-awaited result and our goal: 2076 RPS, HTTP request duration - 936ms, 1800 VUs load 🔥**

```
 █ THRESHOLDS 

    http_req_duration
    ✓ 'p(90)<3000' p(90)=936.92ms


  █ TOTAL RESULTS 

    checks_total.......: 872232  2076.398017/s
    checks_succeeded...: 100.00% 872232 out of 872232
    checks_failed......: 0.00%   0 out of 872232

    ✓ status is 200

    HTTP
    http_req_duration..............: avg=742.26ms min=2.35ms  med=826.55ms max=1.23s p(90)=936.92ms p(95)=971.12ms
      { expected_response:true }...: avg=742.26ms min=2.35ms  med=826.55ms max=1.23s p(90)=936.92ms p(95)=971.12ms
    http_req_failed................: 0.00%  0 out of 872232
    http_reqs......................: 872232 2076.398017/s

    EXECUTION
    iteration_duration.............: avg=2.97s    min=20.99ms med=3.37s    max=4.24s p(90)=3.62s    p(95)=3.7s    
    iterations.....................: 218058 519.099504/s
    vus............................: 5      min=5           max=1800
    vus_max........................: 1800   min=1800        max=1800

    NETWORK
    data_received..................: 3.6 GB 8.6 MB/s
    data_sent......................: 175 MB 417 kB/s




running (7m00.1s), 0000/1800 VUs, 218058 complete and 0 interrupted iterations
default ✓ [======================================] 0000/1800 VUs  7m0s
```

#### What is x-mutagen - you will find in the documentation, but that is a completely different story. 

Happy end, my friends 🎉🎉🎉  🏆🏆🏆

P.S.

And here is my last experiment: 
- enabled spatie/laravel-responsecache package using memcached driver
- refactored Sanctum cached tokens update
- refactored load test users scenario
- octane tunes up to --workers=16 --task-workers=4 --max-requests=5400
- octane swoole enabled logging for debug (commented for production)

**🔥 2362 RPS, avg HTTP request duration - 833ms, 1800 VUs load 🔥**

```
 █ THRESHOLDS 

    checks
    ✓ 'rate>0.95' rate=100.00%

    http_req_duration
    ✓ 'p(90)<3000' p(90)=833.39ms


  █ TOTAL RESULTS 

    checks_total.......: 992556  2362.454002/s
    checks_succeeded...: 100.00% 992556 out of 992556
    checks_failed......: 0.00%   0 out of 992556

    ✓ User Journey - Login: GET current rate status is 200
    ✓ User Journey - Login: GET rates first paginator, page 1 status is 200
    ✓ User Journey - Random paginator: GET current rate status is 200
    ✓ User Journey - Random paginator: GET rates random paginator, page 1 status is 200
    ✓ User Journey - Random paginator: GET rates random paginator, random page status is 200

    HTTP
    http_req_duration..............: avg=619.93ms min=1.98ms   med=666.58ms max=1.35s p(90)=833.39ms p(95)=885.34ms
      { expected_response:true }...: avg=619.93ms min=1.98ms   med=666.58ms max=1.35s p(90)=833.39ms p(95)=885.34ms
    http_req_failed................: 0.00%  0 out of 992556
    http_reqs......................: 992556 2362.454002/s

    EXECUTION
    iteration_duration.............: avg=3.93s    min=225.12ms med=4.23s    max=6.85s p(90)=5.05s    p(95)=5.26s   
    iterations.....................: 165426 393.742334/s
    vus............................: 7      min=7           max=1800
    vus_max........................: 1800   min=1800        max=1800

    NETWORK
    data_received..................: 3.3 GB 8.0 MB/s
    data_sent......................: 199 MB 474 kB/s




running (7m00.1s), 0000/1800 VUs, 165426 complete and 0 interrupted iterations
contacts ✓ [======================================] 0000/1800 VUs  7m0s
```

Impossible mission - completed 😎

P.P.S

I was very unhappy with my decision to serialize Sanctum and its provider tokens. This overly simplistic solution resulted in corrupted restored Eloquent Models, which made my load tests unrealistic. So, I fixed this issue, even improving the service responsiveness. While solving this problem, I came across several interesting discoveries:
1) Laravel's Sanctum uses the outdated and slower SHA256 token encryption algorithm, which I replaced with Blake2b with a shorter hash to speed up access checks under high load.
2) When working with Octane, I discovered that accessing a frequently used singleton from service container was much slower than if I had placed its instance in a separate helper. The result was a 10x speedup for service retrieving.
3) Model serialization and deserialization proved unjustified, so I found a more reliable and faster way to save the model in the cache and restore it from there. It's even better than creating a copy of the model using new Model()->forceFill().

So final changes are:
- PersonalAccessToken hash algorithm changed from SHA256 to Blake2b
- Sanctum caching moved to separate package "dayonyz/sanctum-bulwark"
- Sanctum tokens serialize-unserialize replaced via store-restore methods SanctumBulwark\TokenRepository
- SanctumBulwark\TokenRepository retrieved once in SanctumBulwark\StaticContainer
- Memcached is totally removed
- All caches placed into different Redis DBs on the same instance
- A separate Redis instance is created for async tokens updates from queues
- Increased breaks between user actions in the stress-test scenario

After all these changes and refactoring, I got more realistic and even better results, and I'm happy with them. Now that's a happy end!!!

**🔥 2747 RPS, avg HTTP request duration - 645.46ms, 1800 VUs load 🔥**

```
  █ THRESHOLDS 

    checks
    ✓ 'rate>0.95' rate=100.00%

    http_req_duration
    ✓ 'p(90)<3000' p(90)=645.46ms


  █ TOTAL RESULTS 

    checks_total.......: 1155186 2747.986164/s
    checks_succeeded...: 100.00% 1155186 out of 1155186
    checks_failed......: 0.00%   0 out of 1155186

    ✓ User Journey - Login: GET current rate status is 200
    ✓ User Journey - Login: GET rates first paginator, page 1 status is 200
    ✓ User Journey - Random paginator: GET current rate status is 200
    ✓ User Journey - Random paginator: GET rates random paginator, page 1 status is 200
    ✓ User Journey - Random paginator: GET rates random paginator, random page status is 200

    HTTP
    http_req_duration..............: avg=492.49ms min=1.79ms   med=530.32ms max=1.12s p(90)=645.46ms p(95)=690.51ms
      { expected_response:true }...: avg=492.49ms min=1.79ms   med=530.32ms max=1.12s p(90)=645.46ms p(95)=690.51ms
    http_req_failed................: 0.00%   0 out of 1155186
    http_reqs......................: 1155186 2747.986164/s

    EXECUTION
    iteration_duration.............: avg=3.37s    min=433.73ms med=3.68s    max=4.86s p(90)=4.05s    p(95)=4.14s   
    iterations.....................: 192531  457.997694/s
    vus............................: 12      min=12           max=1800
    vus_max........................: 1800    min=1800         max=1800

    NETWORK
    data_received..................: 3.8 GB  9.1 MB/s
    data_sent......................: 232 MB  551 kB/s




running (7m00.4s), 0000/1800 VUs, 192531 complete and 0 interrupted iterations
contacts ✓ [======================================] 0000/1800 VUs  7m0s
```