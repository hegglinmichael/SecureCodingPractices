# Under construction(done soon!)
## Welcome to executing shell commands in Python!

This file is meant to contain a copy and paste 
example of a secure way to execute shell commands
in Python.

NOTE: This was made for python3

### This file contains
* commandExecution.py

## commandExecution.py
```
import subprocess

class ExecuteShellCommand:

    def __init__(self, command_args):
        self.command_args = command_args

    def execute_shell_command():
        return_code = subprocess.run(self.command_args, stderr=subprocess.PIPE,stdout=subprocess.PIPE, shell=False)
        if return_code:
            return False
        else:
            return True


```

## How to call
```

```

# References (To the guys who are way smarter then me)
* https://levelup.gitconnected.com/how-to-execute-shell-commands-properly-in-python-5b90c1a9213f


