## 将task 加入队列
```
static inline void enqueue_task(struct rq *rq, struct task_struct *p, int flags)
{
    p->sched_class->enqueue_task(rq, p, flags);  // CFS 对应调用enqueue_task_fair
}
```

CFS 实现
```
/*
 * The enqueue_task method is called before nr_running is
 * increased. Here we update the fair scheduling stats and
 * then put the task into the rbtree:
 */
static void enqueue_task_fair(struct rq *rq, struct task_struct *p, int flags)
```

调用 enqueue_entiry(cfs_rq, se, flags)
```
/*
 * MIGRATION
 *
 *	dequeue
 *	  update_curr()
 *	    update_min_vruntime()
 *	  vruntime -= min_vruntime
 *
 *	enqueue
 *	  update_curr()
 *	    update_min_vruntime()
 *	  vruntime += min_vruntime
 *
 * this way the vruntime transition between RQs is done when both
 * min_vruntime are up-to-date.
 *
 * WAKEUP (remote)
 *
 *	->migrate_task_rq_fair() (p->state == TASK_WAKING)
 *	  vruntime -= min_vruntime
 *
 *	enqueue
 *	  update_curr()
 *	    update_min_vruntime()
 *	  vruntime += min_vruntime
 *
 * this way we don't have the most up-to-date min_vruntime on the originating
 * CPU and an up-to-date min_vruntime on the destination CPU.
 */
 
static void enqueue_entity(struct cfs_rq *rq, struct sched_entity *se, int flags)
{
	bool renorm = !(flags & ENQUEUE_WAKEUP) || (flags & ENQUEUE_MIGRATED);
	bool curr = cfs_rq->curr == se;

	/*
	 * If we're the current task, we must renormalise before calling
	 * update_curr().
	 */
	if (renorm && curr)                  // 是当前进程，为什么要先normalize ？？？
		se->vruntime += cfs_rq->min_vruntime;

	update_curr(cfs_rq);

	/*
	 * Otherwise, renormalise after, such that we're placed at the current
	 * moment in time, instead of some random moment in the past. Being
	 * placed in the past could significantly boost this task to the
	 * fairness detriment of existing tasks.
	 */
	if (renorm && !curr)                 // 不是当前进程，在这里normalize
		se->vruntime += cfs_rq->min_vruntime;

	/*
	 * When enqueuing a sched_entity, we must:
	 *   - Update loads to have both entity and cfs_rq synced with now.
	 *   - Add its load to cfs_rq->runnable_avg
	 *   - For group_entity, update its weight to reflect the new share of
	 *     its group cfs_rq
	 *   - Add its new weight to cfs_rq->load.weight
	 */
	update_load_avg(cfs_rq, se, UPDATE_TG | DO_ATTACH);
	update_cfs_group(se);
	enqueue_runnable_load_avg(cfs_rq, se);
	account_entity_enqueue(cfs_rq, se);   // 更新cfs_rq 的load， 可运行个数加1

	if (flags & ENQUEUE_WAKEUP)
		place_entity(cfs_rq, se, 0);      // 如何是唤醒进程，更新se 的vruntime 到合适的值

	check_schedstat_required();
	update_stats_enqueue(cfs_rq, se, flags);
	check_spread(cfs_rq, se);
	if (!curr)                           // 不是当前进程的话，加入红黑树中
		__enqueue_entity(cfs_rq, se);
	se->on_rq = 1;                       // 已经在队列中了

	if (cfs_rq->nr_running == 1) {
		list_add_leaf_cfs_rq(cfs_rq);       // 这里是做了什么？？？
		check_enqueue_throttle(cfs_rq);
	}
}
```