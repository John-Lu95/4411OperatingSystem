## Report of Project3 Process Scheduler

### Implementation
To support the Priority Scheduling, the frist is to add `priority` for each process, as shown below.
```
// Per-process state
struct proc {
  uint sz;                     // Size of process memory (bytes)
  pde_t* pgdir;                // Page table
  char *kstack;                // Bottom of kernel stack for this process
  enum procstate state;        // Process state
  int pid;                     // Process ID
  struct proc *parent;         // Parent process
  struct trapframe *tf;        // Trap frame for current syscall
  struct context *context;     // swtch() here to run process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
  int priority; //zhlu: Add a priority for each process
};
```
Then as asked, two system calls `sys_getnice` and `sys_setnice` are introduced.

```
int
sys_getnice(void)
{
  return myproc()->priority;
}
```

```
int
sys_setnice(void)
{

  int pid, pr;
  if(argint(0, &pid) < 0)
    return -1;
  if(argint(1, &pr) < 0)
    return -1;
  
  return setnice(pid,pr);
}
```
`sys_setnice` further invoke `setnice` in `proc.*` to be able to update `priority`.
```

int setnice(int pid, int nice)
{
  if(nice<0 || nice>31)
    return -1;
  
  struct proc *p;

  acquire(&ptable.lock);
  for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
    if(p->pid == pid) {
        p->priority = nice;
        break;
    }
  }
  release(&ptable.lock);

  if(p== &ptable.proc[NPROC]) return -1;

  return p->priority;
}
```
To support Priority Scheduler, method `scheduler` in `proc.c` needs to be modified. In addition, to do aging of priority, there needs a counter able to record the time of scheduling. Thus, `ptable` is extended with `scheduler_times`. In schduler, additional loop is introduced to compare the priority of all `RUNNABLE` processes and chooses the process with highest priority. 
```
struct {
  struct spinlock lock;
  struct proc proc[NPROC];
  int scheduler_times;
} ptable;
```
In schduler, additional loop is introduced to compare the priority of all `RUNNABLE` processes and chooses the process with highest priority.  Once there is an available process, the priority of all `RUNNABLE` processes are updated with aging policy.

```
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  c->proc = 0;
  
  for(;;){
    // Enable interrupts on this processor.
    sti();

    // Loop over process table looking for process to run.
    acquire(&ptable.lock);
    for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
      if(p->state != RUNNABLE)
        continue;
      
      //zhlu:Scheduler Proc with highese Priority
      struct proc *prioP = 0;
      struct proc *tmP = 0;
      prioP =p;
      for(tmP = ptable.proc; tmP< &ptable.proc[NPROC]; tmP++)
      {
        if( tmP->state== RUNNABLE &&\
            tmP->priority < prioP->priority)
        {
          prioP = tmP;
        }
      }
      if(prioP!=0)
      {
        p = prioP;
        //aging
        ptable.scheduler_times++;
        if(ptable.scheduler_times%2==0)
        {
          for(tmP = ptable.proc; tmP< &ptable.proc[NPROC]; tmP++)
          {
            if(tmP==p)
            {
              continue;
            }
            if(tmP->priority>0)
            {
              tmP->priority--;
            }
          }

          p->priority++;
        }
      }

      // Switch to chosen process.  It is the process's job
      // to release ptable.lock and then reacquire it
      // before jumping back to us.
      c->proc = p;
      switchuvm(p);
      p->state = RUNNING;

      swtch(&(c->scheduler), p->context);
      switchkvm();

      // Process is done running for now.
      // It should have changed its p->state before coming back.
      c->proc = 0;
    }
    release(&ptable.lock);

  }
}
```

### Results
#### Priority Scheduling
```
$ p3test
1  nice 20

2  nice 20

$ 3  nice 24

zombie!
4  nice 28

zombie!

```
#### Default - Round Robin
```
$ p3test
1  nice 20

4  nice 30

2  nice 20

3  nice 25

zombie!
$ zombie!

```

As we can see, the execution order is different. In Priority Scheduling, processes are executed based on the priority. In RR, processes execution is not affected by their priority.


