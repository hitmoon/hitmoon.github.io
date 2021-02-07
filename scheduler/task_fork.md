## task_fork_fair

```
/*
 * called on fork with the child task as argument from the parent's context
 *  - child not yet on the tasklist
 *  - preemption disabled
 */
static void task_fork_fair(struct task_struct *p)
{
	struct cfs_rq *cfs_rq;
	struct sched_entity *se = &p->se, *curr;
	struct rq *rq = this_rq();
	struct rq_flags rf;

	rq_lock(rq, &rf);
	update_rq_clock(rq);

	cfs_rq = task_cfs_rq(current);
	curr = cfs_rq->curr;
	if (curr) {                  // 当前有运行的进程
		update_curr(cfs_rq);
		se->vruntime = curr->vruntime;    // 继承父进程的vruntime ?
	}
	place_entity(cfs_rq, se, 1);     // 初次加入，更新 vruntime

	if (sysctl_sched_child_runs_first && curr && entity_before(curr, se)) {  // 子进程需要先运行，怎么处理？
		/*
		 * Upon rescheduling, sched_class::put_prev_task() will place
		 * 'current' within the tree based on its new key value.
		 */
		swap(curr->vruntime, se->vruntime);         // 交换父子进程的vruntime
		resched_curr(rq);                           // 重新调度 当前进程
	}

	se->vruntime -= cfs_rq->min_vruntime;
	rq_unlock(rq, &rf);
}
```
