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
