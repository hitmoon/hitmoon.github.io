
## 处理不是running 状态的task


调用 deactivate_task, p->sched_class->dequeue_task, 对于cfs 就是 dequeue_task_fair
```
__schedule(bool preempt)
if (!preempt && prev->state) {
    if (unlikely(signal_pending_state(prev->state, prev))) {
        prev->state = TASK_RUNNING;
    } else {
        deactivate_task(rq, prev, DEQUEUE_SLEEP | DEQUEUE_NOCLOCK);
    }
}

void deactivate_task(struct rq *rq, struct task_struct *p, int flags)
{
    p->on_rq = (flags & DEQUEUE_SLEEP) ? 0 : TASK_ON_RQ_MIGRATING;
    dequeue_task(rq, p, flags);
}

static inline void dequeue_task(struct rq *rq, struct task_struct *p, int flags)
{
    if (!(flags & DEQUEUE_SAVE)) {
        sched_info_dequeued(rq, p);
        psi_dequeue(p, flags & DEQUEUE_SLEEP);
    }
    p->sched_class->dequeue_task(rq, p, flags);
}
```

### dequeue_task_fair

###### 1. update_curr
```
/*
 * Update run-time statistics of the 'current'.
 */
 update_curr(cfs_rq)
 
	/*
	 * When dequeuing a sched_entity, we must:
	 *   - Update loads to have both entity and cfs_rq synced with now.
	 *   - Substract its load from the cfs_rq->runnable_avg.
	 *   - Substract its previous weight from cfs_rq->load.weight.
	 *   - For group entity, update its weight to reflect the new share
	 *     of its group cfs_rq.
	 */
if (se != cfs_rq->curr)       // 如果se 不是当前运行的task，说明它在rb tree 里面，移除之
    __dequeue_entity(cfs_rq, se);
se->on_rq = 0;          // 表明已经不在队列中了
account_entity_dequeue(cfs_rq, se);  // 更新cfs_rq 状态

/*
 * Recomputes the group entity based on the current state of its group
 * runqueue.
 */
update_cfs_group(se);   // 更新组相关的信息
```
```
static void account_entity_dequeue(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
    upload_load_sub(&cfs_rq->load, se->load.weight);  // 从cfs_rq 队列load 减去 se 的load
    if (entity_is_task(se)) {
        account_numa_dequeue(rq_of(cfs_rq), task_of(se));   // 更新rq 中numa 相关的记录
        list_del_init(&se->group_node);
    }
    cfs_rq->nr_running--;     // 队列的可运行进程数减1
}
```

##### min_vruntime 的更新
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
```