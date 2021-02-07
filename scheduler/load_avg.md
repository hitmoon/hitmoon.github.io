
### update_load_avg
```
/* Update task and its cfs_rq load average */
update_load_avg(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
```
如果 flags 不包含 SKIP_AGE_LOAD
调用 __update_load_avg_se(now, cfs_rq, se);

然后调用 update_cfs_rq_load_avg

```
/**
 * update_cfs_rq_load_avg - update the cfs_rq's load/util averages
 * @now: current time, as per cfs_rq_clock_pelt()
 * @cfs_rq: cfs_rq to update
 *
 * The cfs_rq avg is the direct sum of all its entities (blocked and runnable)
 * avg. The immediate corollary is that all (fair) tasks must be attached, see
 * post_init_entity_util_avg().
 *
 * cfs_rq->avg is used for task_h_load() and update_cfs_share() for example.
 *
 * Returns true if the load decayed or we removed load.
 *
 * Since both these conditions indicate a changed cfs_rq->avg.load we should
 * call update_tg_load_avg() when this function returns true.
 */
 
static inline int update_cfs_rq_load_avg(u64 now, struct cfs_rq *cfs_rq)
```

__update_laod_avg_cfs_rq(u64 now, struct cfs_rq *cfs_rq) 主要调用一下两个辅助函数:

___update_load_sum 和 ___update_load_avg

#### ___update_load_sum
```
/*
 * We can represent the historical contribution to runnable average as the
 * coefficients of a geometric series.  To do this we sub-divide our runnable
 * history into segments of approximately 1ms (1024us); label the segment that
 * occurred N-ms ago p_N, with p_0 corresponding to the current period, e.g.
 *
 * [<- 1024us ->|<- 1024us ->|<- 1024us ->| ...
 *      p0            p1           p2
 *     (now)       (~1ms ago)  (~2ms ago)
 *
 * Let u_i denote the fraction of p_i that the entity was runnable.
 *
 * We then designate the fractions u_i as our co-efficients, yielding the
 * following representation of historical load:
 *   u_0 + u_1*y + u_2*y^2 + u_3*y^3 + ...
 *
 * We choose y based on the with of a reasonably scheduling period, fixing:
 *   y^32 = 0.5
 *
 * This means that the contribution to load ~32ms ago (u_32) will be weighted
 * approximately half as much as the contribution to load within the last ms
 * (u_0).
 *
 * When a period "rolls over" and we have new u_0`, multiplying the previous
 * sum again by y is sufficient to update:
 *   load_avg = u_0` + y*(u_0 + u_1*y + u_2*y^2 + ... )
 *            = u_0 + u_1*y + u_2*y^2 + ... [re-labeling u_i --> u_{i+1}]
 */
___update_load_sum(u64 now, struct sched_avg *sa, unsigned long load, unsigned long runnable, int running)

```
最后调用 accumulate_sum
```
/*
 * Accumulate the three separate parts of the sum; d1 the remainder
 * of the last (incomplete) period, d2 the span of full periods and d3
 * the remainder of the (incomplete) current period.
 *
 *           d1          d2           d3
 *           ^           ^            ^
 *           |           |            |
 *         |<->|<----------------->|<--->|
 * ... |---x---|------| ... |------|-----x (now)
 *
 *                           p-1
 * u' = (u + d1) y^p + 1024 \Sum y^n + d3 y^0
 *                           n=1
 *
 *    = u y^p +					(Step 1)
 *
 *                     p-1
 *      d1 y^p + 1024 \Sum y^n + d3 y^0		(Step 2)
 *                     n=1
 */
```

load sum 统计优化

```
commit a481db34b9beb7a9647c23f2320dd38a2b1d681f
Author: Yuyang Du <yuyang.du@intel.com>
Date:   Mon Feb 13 05:44:23 2017 +0800

    sched/fair: Optimize ___update_sched_avg()

    The main PELT function ___update_load_avg(), which implements the
    accumulation and progression of the geometric average series, is
    implemented along the following lines for the scenario where the time
    delta spans all 3 possible sections (see figure below):

      1. add the remainder of the last incomplete period
      2. decay old sum
      3. accumulate new sum in full periods since last_update_time
      4. accumulate the current incomplete period
      5. update averages

    Or:

                d1          d2           d3
                ^           ^            ^
                |           |            |
              |<->|<----------------->|<--->|
      ... |---x---|------| ... |------|-----x (now)

      load_sum' = (load_sum + weight * scale * d1) * y^(p+1) +      (1,2)

                                            p
                  weight * scale * 1024 * \Sum y^n +                (3)
                                           n=1

                  weight * scale * d3 * y^0                         (4)

      load_avg' = load_sum' / LOAD_AVG_MAX                          (5)

    Where:

     d1 - is the delta part completing the remainder of the last
          incomplete period,
     d2 - is the delta part spannind complete periods, and
     d3 - is the delta part starting the current incomplete period.

    We can simplify the code in two steps; the first step is to separate
    the first term into new and old parts like:

      (load_sum + weight * scale * d1) * y^(p+1) = load_sum * y^(p+1) +
                                                   weight * scale * d1 * y^(p+1)

    Once we've done that, its easy to see that all new terms carry the
    common factors:

      weight * scale

    If we factor those out, we arrive at the form:

      load_sum' = load_sum * y^(p+1) +

                  weight * scale * (d1 * y^(p+1) +

                                             p
                                    1024 * \Sum y^n +
                                            n=1

                                    d3 * y^0)

    Which results in a simpler, smaller and faster implementation.

```