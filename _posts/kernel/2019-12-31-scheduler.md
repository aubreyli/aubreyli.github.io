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
	
	        local_irq_disable();
	        rcu_note_context_switch(preempt);
	}
