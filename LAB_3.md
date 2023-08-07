```c
diff --git a/kernel/defs.h b/kernel/defs.h
index 3564db4..2893231 100644
--- a/kernel/defs.h
+++ b/kernel/defs.h
@@ -170,6 +170,8 @@ uint64          walkaddr(pagetable_t, uint64);
int             copyout(pagetable_t, uint64, char *, uint64);
int             copyin(pagetable_t, char *, uint64, uint64);
int             copyinstr(pagetable_t, char *, uint64, uint64);
+void vmprintlevel(pagetable_t, int level);
+void vmprint(pagetable_t);
 
 // plic.c
 void            plicinit(void);
@@ -184,3 +186,5 @@ void            virtio_disk_intr(void);
 
 // number of elements in fixed-size array
 #define NELEM(x) (sizeof(x)/sizeof((x)[0]))
+
+uint64 pgaccess(void *pg, int number, void *store);
diff --git a/kernel/proc.c b/kernel/proc.c
index 22e7ce4..07967a0 100644
--- a/kernel/proc.c
+++ b/kernel/proc.c
@@ -140,7 +140,7 @@ found:
   memset(&p->context, 0, sizeof(p->context));
   p->context.ra = (uint64)forkret;
   p->context.sp = p->kstack + PGSIZE;
-
+  p->usyscall->pid = p->pid;
   return p;
 }
 
@@ -151,8 +151,8 @@ static void
 freeproc(struct proc *p)
 {
   if(p->trapframe)
-    kfree((void*)p->trapframe);
:
-  p->trapframe = 0;
+    kfree((void*)p->usyscall);
+  p->usyscall = 0;
   if(p->pagetable)
     proc_freepagetable(p->pagetable, p->sz);
   p->pagetable = 0;
@@ -184,6 +184,7 @@ proc_pagetable(struct proc *p)
   // to/from user space, so not PTE_U.
   if(mappages(pagetable, TRAMPOLINE, PGSIZE,
               (uint64)trampoline, PTE_R | PTE_X) < 0){
+
     uvmfree(pagetable, 0);
     return 0;
   }
@@ -192,6 +193,7 @@ proc_pagetable(struct proc *p)
   if(mappages(pagetable, TRAPFRAME, PGSIZE,
               (uint64)(p->trapframe), PTE_R | PTE_W) < 0){
     uvmunmap(pagetable, TRAMPOLINE, 1, 0);
+    uvmunmap(pagetable, TRAPFRAME, 1, 0);
     uvmfree(pagetable, 0);
     return 0;
   }
@@ -205,7 +207,7 @@ void
 proc_freepagetable(pagetable_t pagetable, uint64 sz)
 {
   uvmunmap(pagetable, TRAMPOLINE, 1, 0);
-  uvmunmap(pagetable, TRAPFRAME, 1, 0);
+  uvmunmap(pagetable, USYSCALL, 1, 0);
:
+  uvmunmap(pagetable, USYSCALL, 1, 0);
   uvmfree(pagetable, sz);
 }
 
@@ -654,3 +656,22 @@ procdump(void)
     printf("\n");
   }
 }
+
+uint64 pgaccess(void *pg, int number, void *store) {
+    struct proc *p = myproc();
+    if (p == 0) {
+        return 1;
+    }
+    pagetable_t pagetable = p->pagetable;
+    int ans = 0;
+    for (int i = 0; i < number; i++) {
+        pte_t *pte;
+        pte = walk(pagetable, ((uint64)pg) + (uint64)PGSIZE * i, 0);
+        if (pte != 0 && ((*pte) & PTE_A)) {
+            ans |= 1 << i;
+            *pte ^= PTE_A;  // clear PTE_A
+        }
+    }
+    // copyout
+    return copyout(pagetable, (uint64)store, (char *)&ans, sizeof(int));
+}
\ No newline at end of file
:
diff --git a/kernel/proc.h b/kernel/proc.h
index f6ca8b7..da9cb15 100644
--- a/kernel/proc.h
+++ b/kernel/proc.h
@@ -105,4 +105,5 @@ struct proc {
   struct file *ofile[NOFILE];  // Open files
   struct inode *cwd;           // Current directory
   char name[16];               // Process name (debugging)
+  struct usyscall *usyscall;
 };
diff --git a/kernel/sysproc.c b/kernel/sysproc.c
index 3bd0007..e6bcf82 100644
--- a/kernel/sysproc.c
+++ b/kernel/sysproc.c
@@ -81,7 +81,13 @@ int
 sys_pgaccess(void)
 {
   // lab pgtbl: your code here.
-  return 0;
+  uint64 buf;
+  int number;
+  uint64 ans;
+  if(argaddr(0, &buf) < 0) return -1;
+  if (argint(1, &number) < 0) return -1;
+  if (argaddr(2, &ans) < 0) return -1;
+  return pgaccess((void*)buf, number, (void*)ans);
 }
 #endif
diff --git a/kernel/vm.c b/kernel/vm.c
index d5a12a0..b828548 100644
--- a/kernel/vm.c
+++ b/kernel/vm.c
@@ -432,3 +432,26 @@ copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
     return -1;
   }
 }
+
+void vmprintlevel(pagetable_t pt, int level) {
+    char *delim = 0;
+    if (level == 2) delim = "..";
+    if (level == 1) delim = ".. ..";
+    if (level == 0) delim = ".. .. ..";
+    for (int i = 0; i < 512; i++) {
+        pte_t pte = pt[i];
+        if ((pte & PTE_V)) {
+            //  this PTE points to a lower level page table.
+            printf("%s%d: pte %p pa %p\n", delim, i, pte, PTE2PA(pte));
+            uint64 child = PTE2PA(pte);
+            if (level != 0) {
+                vmprintlevel((pagetable_t)child, level - 1);
+            }
+        }
+    }
+}
+
+void vmprint(pagetable_t pt) {
+    printf("page table %p\n", pt);
+    vmprintlevel(pt, 2);
+}
+}
\ No newline at end of file
diff --git a/user/sh.c b/user/sh.c
index 83dd513..c96dab0 100644
--- a/user/sh.c
+++ b/user/sh.c
@@ -54,6 +54,7 @@ void panic(char*);
 struct cmd *parsecmd(char*);
 
 // Execute cmd.  Never returns.
+__attribute__((noreturn))
 void
 runcmd(struct cmd *cmd)
 {

```

