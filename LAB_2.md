```c
// sysproc.c
uint64
sys_sysinfo(void) {
    uint64  addr;
    if (argaddr(0, &addr) < 0)
        return -1;
    struct proc *p = myproc();
    struct sysinfo info;
    info.freemem = freemem();
    info.nproc = unusedproc();
    if (copyout(p -> pagetable, addr, (char *)&info, sizeof(info)) < 0)
        return -1;
    return 0;
}
```

```c
// kalloc.c
uint64
freemem(void)
{
    struct run *r;
    uint64 freepage = 0;
    acquire(&kmem.lock);
    r = kmem.freelist;
    while (r)
    {
        freepage += 1;
        r = r->next;
    }
    release(&kmem.lock);
    return (freepage << 12);
}
```

```c
// proc.c
uint64
unusedproc(void)
{
    struct proc *p;
    uint64 unused = 0;

    for(p = proc; p < &proc[NPROC]; p++)
    {
        if(p->state != UNUSED) {
            unused++;
        }
    }

    return unused;
}
```
