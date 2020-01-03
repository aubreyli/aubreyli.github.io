---
layout: post
#comments: true
categories: kernel
---

## __schedule() is the main scheduler function.

	static void __sched notrace __schedule(bool preempt)
	{
		/*
		 * Obtain the current task on the runqueue
		 */
	        cpu = smp_processor_id();
	        rq = cpu_rq(cpu);
	        prev = rq->curr;
	
		/*
		 * Disable interrupt
		 */
	        local_irq_disable();

		/*
		 * Lock up runqueue
		 */
		rq_lock(rq, &rf);

	        /*
		 * Promote REQ to ACT, and update runqueue clock
		 */
	        rq->clock_update_flags <<= 1;
	        update_rq_clock(rq);

		/*
		 * Handle signal pending:
		 * - if the task has signal pending, change its state to TASK_RUNNING,
		 * - else the task stopped/exited, remove the task from runqueue
		 */
		if (!preempt && prev->state) {
	                if (signal_pending_state(prev->state, prev)) {
	                        prev->state = TASK_RUNNING;
	                } else {
	                        deactivate_task(rq, prev, DEQUEUE_SLEEP | DEQUEUE_NOCLOCK);
	                        }               
	                }
		}

		/*
		 * select next task
		 */
		next = pick_next_task(rq, prev, &rf);

		/*
		 * Clear need_resched flag of previous task
		 */
	        clear_tsk_need_resched(prev);
	        clear_preempt_need_resched();
	
	        if (likely(prev != next)) {
			/*
			 * Context switch if next task is not previous task
			 */
	                /* Also unlocks the rq: */
	                rq = context_switch(rq, prev, next, &rf);
	        } else {
			/*
			 * If next task is previous task, clear clock update flag and unlock runqueue
			 */
	                rq->clock_update_flags &= ~(RQCF_ACT_SKIP|RQCF_REQ_SKIP);
	                rq_unlock_irq(rq, &rf);
	        }
		/*
		 * Call balance callback
		 */	
	        balance_callback(rq);
	}

## pick_next_task() picks up the highest-prio task

	static inline struct task_struct *
	pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
	{
		/*
		 * Previous task is idle or fair, call fair pick_next_task directly
		 */
	        if (likely((prev->sched_class == &idle_sched_class ||
	                    prev->sched_class == &fair_sched_class) &&
			   /*
			    * all tasks are in the fair class
			    */
	                   rq->nr_running == rq->cfs.h_nr_running)) {
	
	                p = pick_next_task_fair(rq, prev, rf);
			/*
			 * RETRY_TASK indicates higher priority task appears
			 */
	                if (unlikely(p == RETRY_TASK))
	                        goto restart;
	
	                /* Assumes fair_sched_class->next == idle_sched_class */
	                if (!p) {
				/*
				 * Need to put previous task if sched class is changed
				 */
	                        put_prev_task(rq, prev);
	                        p = pick_next_task_idle(rq);
	                }
	
	                return p;
	        }
	restart:
	        for_class_range(class, prev->sched_class, &idle_sched_class) {
	                if (class->balance(rq, prev, rf))
	                        break;
	        }
		/*
		 * Need to put previous task if sched class is changed
		 */
	        put_prev_task(rq, prev);
	        for_each_class(class) {
	                p = class->pick_next_task(rq);
	                if (p)
	                        return p;
	        }
        }

## put_prev_task_fair() account for descheduled task

	static void put_prev_task_fair(struct rq *rq, struct task_struct *prev)
	{
		struct sched_entity *se = &prev->se;
		struct cfs_rq *cfs_rq;
	
		for_each_sched_entity(se) {
			cfs_rq = cfs_rq_of(se);
			put_prev_entity(cfs_rq, se);
		}
	}

	static void put_prev_entity(struct cfs_rq *cfs_rq, struct sched_entity *prev)
	{
		/*
		 * If still on the runqueue then deactivate_task()
		 * was not called and update_curr() has to be done:
		 */
		if (prev->on_rq)
			update_curr(cfs_rq);
	
		/* throttle cfs_rqs exceeding runtime */
		check_cfs_rq_runtime(cfs_rq);
	
		check_spread(cfs_rq, prev);
	
		if (prev->on_rq) {
			update_stats_wait_start(cfs_rq, prev);
			/* Put 'current' back into the tree. */
			__enqueue_entity(cfs_rq, prev);
			/* in !on_rq case, update occurred at dequeue */
			update_load_avg(cfs_rq, prev, 0);
		}
		cfs_rq->curr = NULL;
	}

## Select task in fair class (simple case only)
	struct task_struct *
	pick_next_task_fair(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
	{
		/*
		 * if no runnable task on the runqueue, goto idle
		 */
		if (!sched_fair_runnable(rq))
			goto idle;
	
		/*
		 * put previous task back to the rb tree
		 */
		if (prev)
			put_prev_task(rq, prev);
	
		/*
		 * The following loop executes only once if no CONFIG_FAIR_GROUP_SCHED
		 */
		do {
			se = pick_next_entity(cfs_rq, NULL);
			set_next_entity(cfs_rq, se);
			cfs_rq = group_cfs_rq(se);
		} while (cfs_rq);
	
		p = task_of(se);
	
	      /*
	       * Move the next running task to the front of
	       * the list, so our cfs_tasks list becomes MRU
	       * one.
	       */
	      list_move(&p->se.group_node, &rq->cfs_tasks);
	
	      if (hrtick_enabled(rq))
		      hrtick_start_fair(rq, p);
	
	      update_misfit_status(p, rq);
	
	      return p;
	
	idle:
	      if (!rf)
		      return NULL;
	
	      new_tasks = newidle_balance(rq, rf);
	
	      /*
	       * Because newidle_balance() releases (and re-acquires) rq->lock, it is
	       * possible for any higher priority task to appear. In that case we
	       * must re-start the pick_next_entity() loop.
	       */
	      if (new_tasks < 0)
		      return RETRY_TASK;
	
	      /*
	       * rq is about to be idle, check if we need to update the
	       * lost_idle_time of clock_pelt
	       */
	      update_idle_rq_clock_pelt(rq);
	
	      return NULL;
	}
