+++
title = "SUD 00: The setuid flag, how it works and why I don't like it"
date = "2024-12-14"
+++

In Linux, the [setuid (Set User ID)](https://en.wikipedia.org/wiki/Setuid) flag is a special file attribute setting 
that allows a program to execute with the privileges of its file owner, typically root, regardless of the user 
executing it.

When a program with the setuid bit set is run, the process inherits the file owner’s user permissions rather than 
the permissions of the user running the program. This functionality was crucial for commands like 
[su (substitute user)](https://linux.die.net/man/1/su), which allows a user to switch to another user account 
(often root) to perform tasks that require higher privileges.

The setuid flag can be set using the `chmod` command with the `u+s` option. To verify if a file has the setuid flag 
enabled, the `ls -l` command can be used to display the file's permissions. 

If the flag is set, an `s` will appear in the owner's execute field. For example, a file might appear as `rwsr-xr-x`, 
where the `s` indicates the presence of the setuid flag.

An example with file `/usr/bin/su`:
```
$ ls -la /usr/bin/su
-rwsr-xr-x 1 root root 51320 Jul  4 10:24 /usr/bin/su
```

This article explores how the setuid flag works, delves into concepts like RUID, EUID, and SUID, and highlights why 
setuid can pose challenges in security and system management.

# Understanding user permission in Linux: RUID, EUID, and SUID
In Linux, processes are associated with three types of user IDs that control how the system manages permissions for
executing programs. These are the Real User ID (**RUID**), Effective User ID (**EUID**), and Saved User ID (**SUID**).

#### Real User ID (RUID)
The Real User ID (RUID) represents the user who initiated the process. It is used primarily for accounting and is the 
user ID associated with the process when it's started. 

This UID does not change during the life of the process, except in the case where EUID is root in which case the process
can change RUID with the [setuid syscall](https://linux.die.net/man/2/setuid).

#### Effective User ID (EUID)
The Effective User ID (EUID) is primarily used to enforce access control and permission checking. When a process
requests access to a file or resource, the system checks the EUID against the file's owner and its permissions.

A process might temporarily change its EUID to root to perform administrative tasks, then drop those privileges 
(reverting EUID back to the RUID) to perform non-privileged actions. This is common in programs that require both 
elevated and standard privileges during execution.

### Saved User ID (SUID)
The Saved User ID (SUID) is used to store the Effective User ID (EUID) before it is changed, typically when the process 
drops or changes privileges. 

The SUID allows the process to restore its EUID to a previously saved value, enabling it to regain privileges 
if needed later in the execution. This is particularly useful in cases where a process might temporarily drop root 
privileges to perform a task with lower permissions, and later needs to restore them to complete the task.

### Summary
<div class="scrollable-table">
  <table class="custom-table">
    <thead>
      <tr>
        <th>Attribute</th>
        <th>RUID (Real User ID)</th>
        <th>EUID (Effective User ID)</th>
        <th>SUID (Saved User ID)</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>Definition</td>
        <td>Identifies the user who initiated the process.</td>
        <td>Determines the privileges and access rights during the execution of a process.</td>
        <td>Stores the previous <strong>EUID</strong> before it was changed, allowing the process to revert to it.</td>
      </tr>
      <tr>
        <td>Usage</td>
        <td>Used for accounting and logging purposes.</td>
        <td>Used to check permissions and enforce access control during process execution.</td>
        <td>Allows process to revert to its previous <strong>EUID</strong>, enabling temporary privilege 
          escalation.</td>
      </tr>
      <tr>
        <td>Typical Values</td>
        <td>The user ID of the person who started the process.</td>
        <td>The user ID that the system uses for permission checks (often root for setuid programs).</td>
        <td>Stores the <strong>EUID</strong> value before it changes, usually during setuid execution.</td>
      </tr>
      <tr>
        <td>Changes during Execution</td>
        <td>Can be changed via <strong>setuid</strong> syscall only if <strong>EUID</strong> is root.</td>
        <td>It can be changed via <strong>setuid</strong> syscall without restrictions if the current 
          <strong>EUID</strong> is <strong>root</strong>. Otherwise, it can only be changed to <strong>RUID</strong> 
          or <strong>SUID</strong>.</td>
        <td>Changes when the <strong>EUID</strong> changes, allowing for privilege escalation.</td>
      </tr>
    </tbody>
  </table>
</div>


# How the setuid flag affects process UIDs on Linux
When a file is executed that has the setuid flag set, the Linux kernel will create a process with EUID and SUID set 
to the UID of the owner of the executed file and RUID to the UID of the user who executed it.

Consequently, if a `test_setuid` file is owned by UID 0 (root) and has the setuid flag, if it is executed by user
UID 1000 it will have: RUID=1000, EUID=0 and SUID=0.

You can test this behavior with this simple program `test_suid.c`:
```
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>

int main() {
    uid_t ruid, euid, suid;

    getresuid(&ruid, &euid, &suid);
    printf("Before setuid() RUID=%d, EUID=%d, SUID=%d\n", ruid, euid, suid);

    if (setuid(ruid) == -1) {
        perror("setuid() failed");
    }

    getresuid(&ruid, &euid, &suid);
    printf("After setuid(): RUID=%d, EUID=%d, SUID=%d\n", ruid, euid, suid);

    return 0;
}
```

Compile and run it with the following commands:
```
gcc -D_GNU_SOURCE -o test_setuid test_setuid.c
sudo chown root:root test_setuid
sudo chmod u+s test_setuid
./test_setuid
```

You should get output like this:
```
Before setuid() RUID=1000, EUID=0, SUID=0
After setuid(): RUID=1000, EUID=1000, SUID=1000
```

Then the following happened:
- The process was created with RUID=1000 (calling user) and EUID=SUID=0 (file owner)
- The process called setuid(RUID)
- Now the process has RUID=EUID=SUID. Please note, this only happens because EUID=0 otherwise SUID would have remained 
untouched, the process has now effectively lost any root privileges. To temporarily drop permissions with EUID=0 
you need to use [seteuid](https://linux.die.net/man/2/seteuid) and not setuid!

# What's wrong with the setuid flag?
The setuid flag has undeniably played a pivotal role in the development of Unix-like systems, providing a practical 
way to grant temporary elevated privileges for essential tasks. Commands like `su`, `passwd`, and others owe their 
functionality to this mechanism. However, while it was a convenient solution for its time, I believe it's worth 
re-evaluating its continued use in modern systems.

The core issue with setuid is that it introduces a "special case" where a file can execute with privileges that differ 
from the user running it. This behavior, while practical in certain scenarios, can become problematic in terms of 
security and system maintenance. For instance:

1. **Security Concerns**: A malicious binary with the setuid flag enabled could execute commands with elevated 
privileges. While such a scenario assumes the system is already compromised, identifying and auditing malicious 
setuid binaries can be challenging. How often do administrators proactively search for setuid-enabled files on their 
systems using commands like `find / -type f -perm -4000`? For most users, the answer is "rarely, if ever." 
This lack of oversight creates potential blind spots in security.

2. **Lack of Transparency**: The setuid mechanism can make it harder to understand and trace how privileges are 
escalated during program execution. It relies on the executed program managing these privileges responsibly, which 
introduces additional risk if the program's code contains vulnerabilities or is exploited.

3. **Outdated Design**: The setuid flag was a clever solution in the early days of Unix, but modern security practices, 
such as capabilities, namespaces, and privilege separation, offer alternative methods that can achieve similar goals 
with finer granularity and less risk.

Rather than relying on special cases like setuid, I believe we should strive for solutions that integrate seamlessly 
with standard system behavior, making privilege escalation transparent and easier to audit.

These are the motivations behind my work on the [Super User Daemon (SUD)](https://github.com/ErnyTech/sud). SUD aims 
to eliminate the need for the setuid flag by introducing a new approach based entirely on standard, common 
functionality. With SUD, systems could mount filesystems with the `nosuid` option, effectively disabling setuid 
altogether while retaining the ability to perform privileged operations securely.

In the next articles, I will detail SUD's operating principles and how it can simplify privilege management without 
relying on outdated mechanisms like setuid.
