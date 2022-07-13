# Under construction(done soon!)
## Welcome to executing shell commands in C++!

This file is meant to contain a copy and paste 
example of a secure way to execute shell commands
in C++.  

NOTE:   There is some personal modification you<br>
       &emsp;&emsp;&emsp;&emsp;SHOULD do.  The stock implementation is <br>
       &emsp;&emsp;&emsp;&emsp;still more secure than using system() <br>
       &emsp;&emsp;&emsp;&emsp;or popen() due to the fact that both methods <br>
       &emsp;&emsp;&emsp;&emsp;"use the shell to launch the program, passing  <br>
       &emsp;&emsp;&emsp;&emsp;the command to execute to the shell and  <br>
       &emsp;&emsp;&emsp;&emsp;leaving the task of breaking up the command’s  <br>
       &emsp;&emsp;&emsp;&emsp;arguments to the shell."[1]  This is extremely <br>
       &emsp;&emsp;&emsp;&emsp;danger because certain special characters,  <br>
       &emsp;&emsp;&emsp;&emsp;like ';', would allow an attack to run their  <br>
       &emsp;&emsp;&emsp;&emsp;own command along with whatever the programmer  <br>
       &emsp;&emsp;&emsp;&emsp;puts in their shell.  SO USE THIS INSTEAD  <br>

### This file contains
1. commandExecution.cpp
2. commandExecution.h
3. Examples on how to propperly call the command execution function

## ExecuteShellCommand.cc

```
#include <stdio.h>
#include <errno.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/wait.h>
#include "ExecuteShellCommand.hh" 

SPC_PIPE *spc_popen(const char *path, char *const argv[], char *const envp[]);
int spc_pclose(SPC_PIPE *p);
pid_t spc_fork(void);

typedef struct {
  FILE  *read_fd;
  FILE  *write_fd;
  pid_t child_pid;
} SPC_PIPE;

/*
 * Definition:
 *   This is the main peopen function rewritten
 *   in a secure manner.  
 *
 * @params:
 *   path: path to the executable: "/bin/ls"
 *   argv: arguments to pass to the command "-l"
 *   envp: environment variables the you would like 
 *         to set EGG=chicken
 * @return:
 *   an SPC_PIP structure that contains pipes to 
 *   read/write data to the shell command executed.
 **/
SPC_PIPE *spc_popen(const char *path, char *const argv[], char *const envp[]) {
  int      stdin_pipe[2], stdout_pipe[2];
  SPC_PIPE *p;
   
  if (!(p = (SPC_PIPE *)malloc(sizeof(SPC_PIPE)))) return 0;
  p->read_fd = p->write_fd = 0;
  p->child_pid = -1;
   
  if (pipe(stdin_pipe) == -1) {
    free(p);
    return 0;
  }
  if (pipe(stdout_pipe) == -1) {
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
   
  if ((p->child_pid = spc_fork(  )) == -1) {
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
  
/*
 * Definition:
 *   This method properly closes all the pipes
 *   once the programmer is done reading and 
 *   writing to the process.  It also waits
 *   for the command to end before closing
 *
 * @params
 *   p: the SPC_PIPE struct returned from the 
 *      spc_popen function
 * @return
 *   returns 0 if successful, -1 otherwise
 */  
int spc_pclose(SPC_PIPE *p) {
  int   status;
  pid_t pid;
   
  if (p->child_pid != -1) {
    do {
      pid = waitpid(p->child_pid, &status, 0);
    } while (pid == -1 && errno == EINTR);
  }
  if (p->read_fd) fclose(p->read_fd);
  if (p->write_fd) fclose(p->write_fd);
  free(p);
  if (pid != -1 && WIFEXITED(status)) return WEXITSTATUS(status);
  else return (pid == -1 ? -1 : 0);
}

/*
 * Definition:
 *   This is the method you can alter!  If you need your
 *   shell command to drop privileges or anything like that
 *   add that those methods here.  Example methods are give
 *   but I can't write a copy and paste example for your 
 *   specific needs;)  That said it is very easy to do!
 *
 * @params: NONE
 * @return: 0 if successful
 */
pid_t spc_fork(void) {
  pid_t childpid;

  if ((childpid = fork()) == -1) return -1;

  /* Reseed PRNGs in both the parent and the child */
  /* See Chapter 11 for examples */

  /* If this is the parent process, there's nothing more to do */
  if (childpid != 0) return childpid;

  /* This is the child process */
  // spc_sanitize_files();   /* Close all open files.  See Recipe 1.1 */
  // spc_drop_privileges(1); /* Permanently drop privileges.  See Recipe 1.3 */

  return 0;
}
```

## ExecuteShellCommand.hh
```
#ifndef MY_ExecuteShellCommand_H // include guard
#define ExecuteShellCommand_H

SPC_PIPE *spc_popen(const char *path, char *const argv[], char *const envp[]); 

#endif
```

## Sample Caller
```
// define executable and options
char executable[] = "/usr/bin/ls";

char var1[] = "-a";
char *const args[] = {executable, var1, NULL};
char *const environmentVariables[] = {NULL};

// call the method
SPC_PIP *myP = spc_popen(executable, args, environmentVariables);

// read the output of the spc_popen method
char buffer[64];
std::string completeResults = "";

while(fgets(buffer, 64, myP->read_fd) != NULL) 
{
    completeResults += buffer;
}

spc_pclose(myP);
```

Important Takeaways:
* sanitize your input variables if you are going to take any sort of input that is not hard coded
* using system and the default popen is dangerous

# References (To the guys who are way smarter than me)
1. https://www.oreilly.com/library/view/secure-programming-cookbook/0596003943/ch01s07.html




