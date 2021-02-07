

## CFS 如何选择下个运行的task

```
pick_next_task_fair（struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
```

分为两类情况
1. 组调度
2. 普通调度

### 组调度的情况
选择下个se：
###### pick_next_entity
原则：
```
/*
   * Pick the next process, keeping these things in mind, in this order:
   * 1) keep things fair between processes/task groups
   * 2) pick the "next" process, since someone really wants that to run
   * 3) pick the "last" process, for cache locality
   * 4) do not run the "skip" process, if something else is available
   */
   ```
   check_cfs_rq_runtime 会进行必要的带宽控制

然后调用:
put_prev_entity 
set_next_entity

##### put_prev_entity:
```

if (prev->on_rq)
    update_curr(cfs_rq);

if (prev->on_rq) {
    __enqueue_entity(cfs_rq, prev);    // 重新加入到红黑树中
    update_laod_avg(cfs_rq, prev, 0);  // 更新load_avg
}
cfs-rq->curr = NULL;    // curr 清空
```

##### set_next_entity:
curr 并不在红黑树中
```
if (se->on_rq) {
    update_stats_wait_end(cfs_rq, se);   // 更新等待时间
    __dequeue_entity(cfs_rq, se);         // 从红黑树中删除
    update_load_avg(cfs_rq, se, UPDATE_TG); // 更新load_avg
}
update_stats_curr_start(cfs_rq, se);   // 记录运行开始时间
cfs_rq->curr = se;                     // 设置curr
```

### 非组调度的情况
```
put_prev_task(rq, prev);   // 这里调用 prev->sched_class->put_prev_task, CFS 对应为: put_prev_task_fair
do {
    se = pick_next_entity(cfs_rq, NULL);
    set_next_entity(cfs_rq, se);
    cfs_rq = group_cfs_rq(se);
 } while (cfs_rq);       // 如果属于另外一个组的队列，则进入相应的队列重新开始
 p =task_of(se);
```