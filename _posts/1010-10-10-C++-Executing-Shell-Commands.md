# Under construction(done soon!)
# Welcome to executing shell commands in C++!

This file is meant to contain a copy and paste 
example of a secure way to execute shell commands
in C++.  

NOTE:  There is some personal modification you
       SHOULD do.  The stock implementation is
       still more secure than using system()
       or popen() due to the fact that both methods
       "use the shell to launch the program, passing 
       the command to execute to the shell and 
       leaving the task of breaking up the command’s 
       arguments to the shell."[1]  This is extremely
       danger because certain special characters, 
       like ';', would allow an attack to run their
       own command along with whatever the programmer
       puts in their shell.  SO USE THIS INSTEAD


## commandInjection.cpp

```
#include <stdio.h>
#include <errno.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/wait.h>
   
typedef struct {
  FILE  *read_fd;
  FILE  *write_fd;
  pid_t child_pid;
} SPC_PIPE;
   
SPC_PIPE *spc_popen(const char *path, char *const argv[], char *const envp[]) {
  int      stdin_pipe[2], stdout_pipe[2];
  SPC_PIPE *p;
   
  if (!(p = (SPC_PIPE *)malloc(sizeof(SPC_PIPE)))) return 0;
  p->read_fd = p->write_fd = 0;
  p->child_pid = -1;
   
  if (pipe(stdin_pipe) =  = -1) {
    free(p);
    return 0;
  }
  if (pipe(stdout_pipe) =  = -1) {
    close(stdin_pipe[1]);
    close(stdin_pipe[0]);
    free(p);
    return 0;
  }
   
  if (!(p->read_fd = fdopen(stdout_pipe[0], "r"))) {
    close(stdout_pipe[1]);
    close(stdout_pipe[0]);
    close(stdin_pipe[1]);
    close(stdin_pipe[0]);
    free(p);
    return 0;
  }
  if (!(p->write_fd = fdopen(stdin_pipe[1], "w"))) {
    fclose(p->read_fd);
    close(stdout_pipe[1]);
    close(stdin_pipe[1]);
    close(stdin_pipe[0]);
    free(p);
    return 0;
  }
   
  if ((p->child_pid = spc_fork(  )) =  = -1) {
    fclose(p->write_fd);
    fclose(p->read_fd);
    close(stdout_pipe[1]);
    close(stdin_pipe[0]);
    free(p);
    return 0;
  }
   
  if (!p->child_pid) {
    /* this is the child process */
    close(stdout_pipe[0]);
    close(stdin_pipe[1]);
    if (stdin_pipe[0] != 0) {
      dup2(stdin_pipe[0], 0);
      close(stdin_pipe[0]);
    }
    if (stdout_pipe[1] != 1) {
      dup2(stdout_pipe[1], 1);
      close(stdout_pipe[1]);
    }
    execve(path, argv, envp);
    exit(127);
  }
   
  close(stdout_pipe[1]);
  close(stdin_pipe[0]);
  return p;
}
   
int spc_pclose(SPC_PIPE *p) {
  int   status;
  pid_t pid;
   
  if (p->child_pid != -1) {
    do {
      pid = waitpid(p->child_pid, &status, 0);
    } while (pid =  = -1 && errno =  = EINTR);
  }
  if (p->read_fd) fclose(p->read_fd);
  if (p->write_fd) fclose(p->write_fd);
  free(p);
  if (pid != -1 && WIFEXITED(status)) return WEXITSTATUS(status);
  else return (pid =  = -1 ? -1 : 0);
}

pid_t spc_fork(void) {
  pid_t childpid;

  if ((childpid = fork(  )) =  = -1) return -1;

  /* Reseed PRNGs in both the parent and the child */
  /* See Chapter 11 for examples */

  /* If this is the parent process, there's nothing more to do */
  if (childpid != 0) return childpid;

  /* This is the child process */
  spc_sanitize_files(  );   /* Close all open files.  See Recipe 1.1 */
  spc_drop_privileges(1); /* Permanently drop privileges.  See Recipe 1.3 */

  return 0;
}
```

## commandInjection.h
```

```

# References (To the guys who are way smarter than me)
1. https://www.oreilly.com/library/view/secure-programming-cookbook/0596003943/ch01s07.html




