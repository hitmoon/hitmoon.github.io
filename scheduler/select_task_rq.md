
## 调用时机
* #### try_to_wake_up                    // 唤醒进程
* #### wake_up_new_task              // 唤醒新创建的进程
* #### sched_exec                          //  执行execve

```
/*
 * select_task_rq_fair: Select target runqueue for the waking task in domains
 * that have the 'sd_flag' flag set. In practice, this is SD_BALANCE_WAKE,
 * SD_BALANCE_FORK, or SD_BALANCE_EXEC.
 *
 * Balances load by selecting the idlest CPU in the idlest group, or under
 * certain conditions an idle sibling CPU if the domain has SD_WAKE_AFFINE set.
 *
 * Returns the target CPU number.
 *
 * preempt must be disabled.
 */
static int
select_task_rq_fair(struct task_struct *p, int prev_cpu, int sd_flag, int wake_flags)
{
	struct sched_domain *tmp, *sd = NULL;
	int cpu = smp_processor_id();
	int new_cpu = prev_cpu;
	int want_affine = 0;
	int sync = (wake_flags & WF_SYNC) && !(current->flags & PF_EXITING);

	if (sd_flag & SD_BALANCE_WAKE) {
		record_wakee(p);

		if (static_branch_unlikely(&sched_energy_present)) {
			new_cpu = find_energy_efficient_cpu(p, prev_cpu);
			if (new_cpu >= 0)
				return new_cpu;
			new_cpu = prev_cpu;
		}

		want_affine = !wake_wide(p) && !wake_cap(p, cpu, prev_cpu) &&
			      cpumask_test_cpu(cpu, &p->cpus_allowed);
	}

	rcu_read_lock();
	for_each_domain(cpu, tmp) {
		if (!(tmp->flags & SD_LOAD_BALANCE))
			break;

		/*
		 * If both 'cpu' and 'prev_cpu' are part of this domain,
		 * cpu is a valid SD_WAKE_AFFINE target.
		 */
		if (want_affine && (tmp->flags & SD_WAKE_AFFINE) &&
		    cpumask_test_cpu(prev_cpu, sched_domain_span(tmp))) {
			if (cpu != prev_cpu)
				new_cpu = wake_affine(tmp, p, cpu, prev_cpu, sync);

			sd = NULL; /* Prefer wake_affine over balance flags */
			break;
		}

		if (tmp->flags & sd_flag)
			sd = tmp;
		else if (!want_affine)
			break;
	}

	if (unlikely(sd)) {
		/* Slow path */
		new_cpu = find_idlest_cpu(sd, p, cpu, prev_cpu, sd_flag);    // 选择最空闲的cpu
	} else if (sd_flag & SD_BALANCE_WAKE) { /* XXX always ? */
		/* Fast path */

		new_cpu = select_idle_sibling(p, prev_cpu, new_cpu);

		if (want_affine)
			current->recent_used_cpu = cpu;
	}
	rcu_read_unlock();

	return new_cpu;
}
```

