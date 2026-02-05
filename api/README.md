# Currency Rates Service

Application for fetching currency rates and viewing them via UI, optimized for high load.

---

## Experiment Goal

- **1500 RPS (unique users)**
- **~1000 ms HTTP response time (p90)**
- **1800–2000 concurrent users (VUs)**

Test environment: **MacBook Pro M1, 16GB RAM, Docker (14GB RAM / 8 CPU)**

---

### Backend Deploy (baseline, no Octane, no x-mutagen)

```bash
cd api
make build
make generate-env
make install
```

### Test User

```bash
test0@example.com
password
```

## Phase 0 — Baseline (Reference Point)

**Stack**
- Laravel
- PHP-FPM + Nginx
- No Octane
- Redis / APCu
- Opcache enabled

This phase exists only to establish a **starting ceiling**.

---

## Phase 1 — Classic Laravel Optimization Ceiling
*(PHP-FPM + Nginx, no Octane)*

### Hypothesis
Classic Laravel optimizations should scale RPS close to the target.

### Actions
- Redis / APCu repository cache
- `php artisan optimize`
- Optimized docker-compose & Nginx configs

### Results
- **Redis**: **838 RPS**, p90 **1.02s**, **800 VUs**
- **APCu**: **806 RPS**, p90 **1.07s**, **800 VUs**
- **0% errors**

### Conclusion
Stable system, but a **hard performance ceiling**.  
RPS stops scaling despite added VUs.

---

## Phase 2 — Sanctum & PHP-FPM Tuning

### Hypothesis
Authentication layer and PHP-FPM are hidden bottlenecks.

### Actions
- Sanctum tokens cached in Redis
- Async token updates
- APCu for sessions
- `pm.max_children = 200`

### Results
- **~780 RPS**, p90 **1.4s**, **1000 VUs**
- **0% errors**
- No RPS growth

### Conclusion
System is **CPU / IO bound**.  
Horizontal scaling on a local Docker setup is meaningless.

---

## Phase 3 — Laravel Octane Introduction (Swoole)

### Hypothesis
Octane will remove PHP-FPM overhead and increase throughput.

### Attempt 3.1 — APCu (Invalid)

- **874 RPS**, p90 **1.19s**, **1000 VUs**

⚠️ Result discarded: **APCu is not shared between Octane workers**.

---

### Attempt 3.2 — Shared Cache (Memcached)

- **929 RPS**, p90 **1.13s**, **1000 VUs**

### Scaling Experiments
- Up to **1500 VUs**
- Manual tuning: `--workers`, `--task-workers`
- Nginx balancer with multiple Octane instances

### Results
- **~940 RPS**, p90 **1.64s**, **1500 VUs**

### Conclusion
Octane helped, but **Nginx + Docker filesystem** became the dominant bottleneck.

---

## Phase 4 — Environment Bottleneck Removal

### Hypothesis
Infrastructure overhead limits performance more than code.

### Actions
- Disabled Nginx
- Direct Octane handling
- Introduced **x-mutagen**
- Removed all unnecessary proxy layers

### Breakthrough Result
🔥 **2076 RPS**, p90 **936 ms**, **1800 VUs**

✅ **Initial goal exceeded**

---

## Phase 5 — Beyond the Goal (Deep Refactoring)

### Hypothesis
After infra limits are removed, application logic becomes dominant.

### Actions
- `spatie/laravel-responsecache` (Redis)
- Sanctum logic refactored
- Async token updates
- More realistic load-test scenario
- Octane tuned to `--workers=16`

### Result
🔥 **2362 RPS**, avg **833 ms**, **1800 VUs**

---

## Phase 6 — Hidden Architecture Traps

### Discoveries
1. Sanctum SHA-256 hashing → replaced with **Blake2b**
2. Service container singleton access in Octane is **~10× slower** than static access
3. Model serialization caused corrupted state

### Fixes
- Explicit store/restore instead of serialization
- Memcached removed
- Redis split by responsibility
- Separate Redis for async token updates
- Sanctum extracted into `dayonyz/sanctum-bulwark`

### Final Result
🔥 **2747 RPS**  
🔥 **p90 = 645 ms**  
🔥 **1800 VUs**  
🔥 **0% errors**

---

## Final Outcome

- Target not only reached, but **significantly exceeded**
- Main limits were **environmental**, not framework-related
- Octane requires **architectural discipline**
- Authentication is a **major hidden hotspot**
- High-load optimization is **investigation, not a checklist**

🏆 **Experiment complete. Goal surpassed. Expectations broken.**
