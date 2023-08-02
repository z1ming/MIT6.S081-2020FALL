# Lab 1

## sleep(easy)

```c
// sleep.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void
main(int argc, char const *argv[])
{
if (argc != 2) {
fprintf(2, "Usage: sleep seconds\n");
exit(1);
}
int time = atoi(argv[1]);
sleep(time);

exit(0);
}
```

```c
//
// Created by hb17021_1 on 2023/7/27.
//
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
int
main(int argc, char const *argv[]) {
    int pid;
    int p[2];
    pipe(p);

    if (fork() == 0) {
        // child
        pid = getpid();
        char buf[2];
        if (read(p[0], buf, 1) != 1) {
            fprintf(2, "failed to read in child\n");
            exit(1);
        }
        close(p[0]);
        printf("%d: received ping\n", pid);
        if (write(p[1], buf, 1) != 1) {
            fprintf(2, "failed to write in child\n");
            exit(1);
        }
        close(p[1]);
        exit(0);
    } else {
        // parent
        pid = getpid();
        char info[2] = "a";
        char buf[2];
        buf[1] = 0;
        if (write(p[1], info, 1) != 1) {
            fprintf(2, "failed to write in parent\n");
            exit(1);
        }
        close(p[1]);
        wait(0);
        if (read(p[0], buf, 1) != 1) {
            fprintf(2, "failed to read in parent\n");
            exit(1);
        }

        printf("%d: received pong\n", pid);
        close(p[0]);
        exit(0);
    }
}
```

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
__attribute__((noreturn))
void new_proc(int p[2]){
    int prime;
    int flag;
    int n;
    close(p[1]);
    if(read(p[0], &prime, 4) != 4){
        fprintf(2, "child process failed to read\n");
        exit(1);
    }
    printf("prime %d\n", prime);

    flag = read(p[0], &n, 4);
    if(flag){
        int newp[2];
        pipe(newp);
        if (fork() == 0)
        {
            new_proc(newp);
        }else
        {
            close(newp[0]);
            if(n%prime)write(newp[1], &n, 4);

            while(read(p[0], &n, 4)){
                if(n%prime)write(newp[1], &n, 4);
            }
            close(p[0]);
            close(newp[1]);
            wait(0);
        }
    }
    exit(0);
}

int main(int argc, char const *argv[])
{
    int p[2];
    pipe(p);
    if (fork() == 0)
    {
        new_proc(p);
    }else
    {
        close(p[0]);
        for(int i = 2; i <= 35; i++)
        {
            if (write(p[1], &i, 4) != 4)
            {
                fprintf(2, "first process failed to write %d into the pipe\n", i);
                exit(1);
            }
        }
        close(p[1]);
        wait(0);
        exit(0);
    }
    return 0;
}
```

```c
//
// Created by hb17021_1 on 2023/8/1.
//
//
// Created by hb17021_1 on 2023/8/1.
//
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

void
find(char const *path, char const *target)
{
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if((fd = open(path, 0)) < 0){
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }

    if(fstat(fd, &st) < 0){
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }

    switch(st.type){
        case T_FILE:
            fprintf(2, "Usage: find dir file \n");
            exit(1);

        case T_DIR:
            if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
                printf("find: path too long\n");
                break;
            }
            strcpy(buf, path);
            p = buf+strlen(buf);
            *p++ = '/';
            while(read(fd, &de, sizeof(de)) == sizeof(de)){
                if(de.inum == 0 || strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0)
                    continue;
                memmove(p, de.name, DIRSIZ);
                p[DIRSIZ] = 0;
                if(stat(buf, &st) < 0){
                    printf("find: cannot stat %s\n", buf);
                    continue;
                }
                if (st.type == T_DIR) {
                    find(buf, target);
                } else if (st.type == T_FILE) {
                    if (strcmp(de.name, target) == 0) {
                        printf("%s\n", buf);
                    }
                }
            }
            break;
    }
    close(fd);
}

int
main(int argc, char const *argv[]) {
    if (argc != 3) {
        fprintf(2, "Usage: find dir file \n");
        exit(1);
    }
    char const *path = argv[1];
    char const *target = argv[2];
    find(path, target);
    exit(0);
}
```