
## Batch sched

### BT 抢占
```
/*
 * Preempt the current task with a newly woken task if needed:
 */
static void check_preempt_wakeup_bt(struct rq *rq, struct task_struct *p, int wake_flags)
{
        struct task_struct *curr = rq->curr;
        struct sched_entity *se = &curr->bt, *pse = &p->bt;
        struct bt_rq *bt_rq = task_bt_rq(curr);
        int scale = bt_rq->nr_running >= sched_nr_latency;
        int next_buddy_marked = 0;

        if (unlikely(se == pse))
                return;

        if (sched_feat(NEXT_BUDDY) && scale && !(wake_flags & WF_FORK)) {
                set_next_buddy_bt(pse);
                next_buddy_marked = 1;
        }

        /*
         * We can come here with TIF_NEED_RESCHED already set from new task
         * wake up path.
         *
         * Note: this also catches the edge-case of curr being in a throttled
         * group (e.g. via set_curr_task), since update_curr() (in the
         * enqueue of curr) will have resulted in resched being set.  This
         * prevents us from potentially nominating it as a false LAST_BUDDY
         * below.
         */
        if (test_tsk_need_resched(curr))
                return;

        /* BT tasks are by definition preempted by non-bt tasks. */
        if (likely(p->policy < SCHED_BT))           // 被CFS 等高优先级调度抢占
                goto preempt;

        if (!sched_feat(WAKEUP_PREEMPTION))
                return;

        update_curr_bt(bt_rq_of(se));
        BUG_ON(!pse);
        if (wakeup_preempt_bt_entity(se, pse) == 1) {
                /*
                 * Bias pick_next to pick the sched entity that is
                 * triggering this preemption.
                 */
                if (!next_buddy_marked)
                        set_next_buddy_bt(pse);
                goto preempt;
        }

        return;

preempt:
        resched_curr(rq);
        /*
         * Only set the backward buddy when the current task is still
         * on the rq. This can happen when a wakeup gets interleaved
         * with schedule on the ->pre_schedule() or idle_balance()
         * point, either of which can * drop the rq lock.
         *
         * Also, during early boot the idle thread is in the fair class,
         * for obvious reasons its a bad idea to schedule back to it.
         */
        if (unlikely(!se->on_rq || curr == rq->idle))
                return;

        if (sched_feat(LAST_BUDDY) && scale && bt_entity_is_task(se))
                set_last_buddy_bt(se);
}

```

### cfs 抢占修改

#### 修改
```
	/* Idle tasks are by definition preempted by non-idle tasks. */
#ifdef	CONFIG_BT_SCHED
	if (unlikely(curr->policy == SCHED_IDLE) &&
	    likely(p->policy != SCHED_IDLE && p->policy != SCHED_BT))   // 目标进程不是IDLE， 也不是BT ?
#else
	if (unlikely(curr->policy == SCHED_IDLE) &&
	    likely(p->policy != SCHED_IDLE))
#endif
		goto preempt;
```

---
## CPU 选择
#### CFS pick_next_task 修改

```
@@ -6279,11 +6272,17 @@ pick_next_task_fair(struct rq *rq, struct task_struct *prev, struct rq_flags *rf
        struct cfs_rq *cfs_rq = &rq->cfs;
        struct sched_entity *se;
        struct task_struct *p;
+#ifndef        CONFIG_BT_SCHED
        int new_tasks;

 again:
+#endif
        if (!cfs_rq->nr_running)
+#ifdef CONFIG_BT_SCHED
+               return NULL;                   // cfs没有进程，返回空， 给BT进程调度机会
+#else
                goto idle;                     //  否则，idle_balance
+#endif


```
#### core pick_next_task修改
```
static inline struct task_struct *
pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
	const struct sched_class *class;
	struct task_struct *p;

	/*
	 * Optimization: we know that if all tasks are in the fair class we can
	 * call that function directly, but only if the @prev task wasn't of a
	 * higher scheduling class, because otherwise those loose the
	 * opportunity to pull in more work from other CPUs.
	 */
#ifdef CONFIG_BT_SCHED
	if (likely((prev->sched_class == &idle_sched_class ||
		    prev->sched_class == &bt_sched_class ||
		    prev->sched_class == &fair_sched_class) &&
		   (rq->nr_running - rq->bt_nr_running) == rq->cfs.h_nr_running)) {   // 仅剩下BT和CFS的情况下，优先选择CFS进程
#else
	if (likely((prev->sched_class == &idle_sched_class ||
		    prev->sched_class == &fair_sched_class) &&
		   rq->nr_running == rq->cfs.h_nr_running)) {
#endif

		p = fair_sched_class.pick_next_task(rq, prev, rf);
		if (unlikely(p == RETRY_TASK))
			goto again;

#ifdef CONFIG_BT_SCHED
		if (!p)                                             // 没有CFS 进程运行的话，选择BT进程
			p = bt_sched_class.pick_next_task(rq, prev, rf);
#endif

		if (unlikely(!p))
			p = idle_sched_class.pick_next_task(rq, prev, rf);

		return p;
	}

again:
	for_each_class(class) {
		p = class->pick_next_task(rq, prev, rf);
		if (p) {
			if (unlikely(p == RETRY_TASK))
				goto again;
			return p;
		}
	}

	/* The idle class should always have a runnable task: */
	BUG();
}

```

#### BT pick_next_task
```
static struct task_struct *pick_next_task_bt(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
	struct task_struct *p;
	struct bt_rq *bt_rq;
	struct sched_entity *se;

	bt_rq = &rq->bt;
	if (!bt_rq->nr_running)          // 是否还有可运行的BT进程
		return NULL;

	if (bt_rq_throttled(bt_rq))       // BT 节流控制
		return NULL;

	put_prev_task(rq, prev);

	do {
		se = pick_next_bt_entity(bt_rq);
		set_next_bt_entity(bt_rq, se);
		bt_rq = group_bt_rq(se);
	}while(bt_rq);

	p = bt_task_of(se);

	return p;
}

```
---
### load_balance

```
--- a/include/linux/interrupt.h
+++ b/include/linux/interrupt.h
@@ -465,6 +465,9 @@ enum
        IRQ_POLL_SOFTIRQ,
        TASKLET_SOFTIRQ,
        SCHED_SOFTIRQ,
+#ifdef CONFIG_BT_SCHED
+       SCHED_BT_SOFTIRQ,              // 增加一个软中断
+#endif
        HRTIMER_SOFTIRQ, /* Unused, but kept as tools rely on the
                            numbering. Sigh! */
        RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */

```

### find_idlest_cpu_bt
```
/*
 * find_idlest_cpu - find the idlest cpu among the cpus in group.
 */
static int
find_idlest_cpu_bt(struct sched_group *group, struct task_struct *p, int this_cpu)
{
        unsigned long load, min_load = ULONG_MAX;
        int idlest = -1;
        int i;
        struct rq *rq;

        /* Traverse only the allowed CPUs */
        for_each_cpu_and(i, sched_group_cpus(group), tsk_cpus_allowed(p)) {
                rq = cpu_rq(i);
                load = cpu_runnable_load_bt(rq) + cpu_runnable_load_bt(rq);

                if ((load < min_load || (load == min_load && i == this_cpu))&&
                    !cpu_rq(i)->bt.bt_throttled && rq->nr_running == rq->bt_nr_running) {    // 只考虑光有BT任务的CPU
                        min_load = load;
                        idlest = i;
                }
        }

        return idlest;
}

```

### select_idle_sibling_bt

```
static int select_idle_sibling_bt(struct task_struct *p, int target)
{
        struct sched_domain *sd;
        struct sched_group *sg;
        int i = task_cpu(p);
        int dst_cpu = target;
        int new_cpu = -1;
        int loop;

        if (idle_cpu(dst_cpu) && !cpu_rq(dst_cpu)->bt.bt_throttled)    // 如果目标cpu 空闲并且没有被节流， 直接返回目标cpu
                return dst_cpu;

        /*
         * If the prevous cpu is cache affine and idle, don't be stupid.
         */
        if (i != dst_cpu && cpus_share_cache(i, dst_cpu) && idle_cpu(i) &&   // 目标cpu 和 之前运行的cpu共享缓存
                                                                             // 并且之前cpu没有被节流并且空闲，使用之前的cpu
            !cpu_rq(i)->bt.bt_throttled)
                return i;

        /*
         * Otherwise, iterate the domains and find an elegible idle cpu.     // 否则，在目标cpu 所在的(低一级) domain 中找一个合格的cpu
         */
        sd = rcu_dereference(per_cpu(sd_llc, dst_cpu));
        for_each_lower_domain(sd) {
                sg = sd->groups;
                do {
                        if (!cpumask_intersects(sched_group_cpus(sg),         // cpus allowed 的满足
                                                tsk_cpus_allowed(p)))
                                goto next;

                        loop = 0;
                        for_each_cpu(i, sched_group_cpus(sg)) {
                                if (i == dst_cpu || !idle_cpu(i) || cpu_rq(i)->bt.bt_throttled) {   // 找到目标cpu或者cpu 不空闲 或者被节流， 继续查找
                                        loop = 1;
                                        continue;
                                }
                                if (new_cpu == -1)
                                        new_cpu = i;                             // 找到第一个满足条件的cpu
                        }

                        if (loop)
                                goto next;

                        dst_cpu = cpumask_first_and(sched_group_cpus(sg),          // 拿到第一个允许运行的cpu
                                        tsk_cpus_allowed(p));
                        goto done;
next:
                        sg = sg->next;
                } while (sg != sd->groups);
        }
done:
        if (dst_cpu == target && new_cpu != -1)            // 如果第一个可以运行的cpu 就是目标cpu 并且还有更好的选择，使用之
                dst_cpu = new_cpu;

        return dst_cpu;                // 返回第一个可以运行的cpu
}

```

### slelect_task_rq_bt

```

```

### CFS select_idle_cpu/core/sibling
```
        return cpu_rq(cpu)->cpu_capacity;
@@ -5719,7 +5808,11 @@ static int select_idle_core(struct task_struct *p, struct sched_domain *sd, int

                for_each_cpu(cpu, cpu_smt_mask(core)) {
                        cpumask_clear_cpu(cpu, cpus);
+#ifdef CONFIG_BT_SCHED
+                       if (!idle_bt_cpu(cpu))
+#else
                        if (!idle_cpu(cpu))
+#endif
                                idle = false;
                }

@@ -5748,7 +5841,11 @@ static int select_idle_smt(struct task_struct *p, struct sched_domain *sd, int t
        for_each_cpu(cpu, cpu_smt_mask(target)) {
                if (!cpumask_test_cpu(cpu, &p->cpus_allowed))
                        continue;
+#ifdef CONFIG_BT_SCHED
+               if (idle_bt_cpu(cpu))
+#else
                if (idle_cpu(cpu))
+#endif
                        return cpu;
        }

@@ -5811,7 +5908,11 @@ static int select_idle_cpu(struct task_struct *p, struct sched_domain *sd, int t
                        return -1;
                if (!cpumask_test_cpu(cpu, &p->cpus_allowed))
                        continue;
+#ifdef CONFIG_BT_SCHED
+               if (idle_bt_cpu(cpu))
+#else
                if (idle_cpu(cpu))
+#endif
                        break;
        }

@@ -5831,13 +5932,21 @@ static int select_idle_sibling(struct task_struct *p, int prev, int target)
        struct sched_domain *sd;
        int i;

+#ifdef CONFIG_BT_SCHED
+       if (idle_bt_cpu(target))
+#else
        if (idle_cpu(target))
+#endif
                return target;

        /*
         * If the previous cpu is cache affine and idle, don't be stupid.
         */
+#ifdef CONFIG_BT_SCHED
+       if (prev != target && cpus_share_cache(prev, target) && idle_bt_cpu(prev))
+#else
        if (prev != target && cpus_share_cache(prev, target) && idle_cpu(prev))
+#endif
                return prev;


```