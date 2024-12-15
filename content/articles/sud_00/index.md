+++
title = "SUD 00: The setuid flag, how it works and why I don't like it"
date = "2024-12-14"
+++

In Linux, the [setuid (Set User ID)](https://en.wikipedia.org/wiki/Setuid) flag is a permission setting that allows a 
program to execute with the privileges of its file owner, typically root, regardless of the user executing it.

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
<div style="overflow-x: auto; -webkit-overflow-scrolling: touch;">
  <table style="border-collapse: collapse; width: 100%; border: 1px solid #ddd;">
    <thead>
      <tr>
        <th style="text-align: left; padding: 8px; background-color: #f4f4f4;">Attribute</th>
        <th style="text-align: left; padding: 8px; background-color: #f4f4f4;">RUID (Real User ID)</th>
        <th style="text-align: left; padding: 8px; background-color: #f4f4f4;">EUID (Effective User ID)</th>
        <th style="text-align: left; padding: 8px; background-color: #f4f4f4;">SUID (Saved User ID)</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td style="text-align: left; padding: 8px; word-wrap: break-word;">Definition</td>
        <td style="text-align: left; padding: 8px; word-wrap: break-word;">Identifies the user who initiated the 
          process.</td>
        <td style="text-align: left; padding: 8px; word-wrap: break-word;">Determines the privileges and access rights 
          during the execution of a process.</td>
        <td style="text-align: left; padding: 8px; word-wrap: break-word;">Stores the previous <strong>EUID</strong> 
          before it was changed, allowing the process to revert to it.</td>
      </tr>
      <tr>
        <td style="text-align: left; padding: 8px; word-wrap: break-word;">Usage</td>
        <td style="text-align: left; padding: 8px; word-wrap: break-word;">Used for accounting and logging 
          purposes.</td>
        <td style="text-align: left; padding: 8px; word-wrap: break-word;">Used to check permissions and enforce access 
          control during process execution.</td>
        <td style="text-align: left; padding: 8px; word-wrap: break-word;">Allows process to revert to its previous 
          <strong>EUID</strong>, enabling temporary privilege escalation.</td>
      </tr>
      <tr>
        <td style="text-align: left; padding: 8px; word-wrap: break-word;">Typical Values</td>
        <td style="text-align: left; padding: 8px; word-wrap: break-word;">The user ID of the person who started the 
          process.</td>
        <td style="text-align: left; padding: 8px; word-wrap: break-word;">The user ID that the system uses for 
          permission checks (often root for setuid programs).</td>
        <td style="text-align: left; padding: 8px; word-wrap: break-word;">Stores the <strong>EUID</strong> value 
          before it changes, usually during setuid execution.</td>
      </tr>
      <tr>
        <td style="text-align: left; padding: 8px; word-wrap: break-word;">Changes during Execution</td>
        <td style="text-align: left; padding: 8px; word-wrap: break-word;">Can be changed via <strong>setuid</strong> 
          syscall only if <strong>EUID</strong> is root.</td>
        <td style="text-align: left; padding: 8px; word-wrap: break-word;">It can be changed via 
          <strong>setuid</strong> syscall without restrictions if the current <strong>EUID</strong> is 
          <strong>root</strong>. Otherwise, it can only be changed to <strong>RUID</strong> or 
          <strong>SUID</strong>.</td>
        <td style="text-align: left; padding: 8px; word-wrap: break-word;">Changes when the <strong>EUID</strong> 
          changes, allowing for privilege escalation.</td>
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
I believe there is a fundamental issue with the use of the setuid flag: Why should a file be special and executed with 
an EUID different from the calling user's? The reason is simple: it was the most convenient way to create programs 
like su, passwd, etc. 

I don't think it's a good reason to continue using it today. I don't like special cases, and I truly believe they 
shouldn't exist at all.

The setuid flag is essentially a permission escalation managed by the executed process, what happens if a file is added 
with the setuid flag enabled and then a malicious unprivileged process calls it specifically to execute privileged 
commands? 

Surely the operating system has already been compromised at this point (the malicious actor has gained root 
before being able to do this) but I invite you to think: "Can you find out if there is a malicious binary with 
setuid enabled?"

This malicious binary may not run for days, how many check their system for new setuid binaries 
(e.g. `find / -type f -perm -4000`)? As far as I'm concerned auditing and monitoring a possible malicious setuid binary 
is extremely complex and is essentially not checked by anyone.

These were the reasons that led me to start the [Super User Daemon (SUD)](https://github.com/ErnyTech/sud) project, 
the idea behind it is to create a user privilege manager that does not use the setuid flag or any other special 
function but instead use a novel approach that relies on absolutely common functionality. Thanks to SUD it will be 
possible to mount the filesystem with `nosuid` option completely eliminating the setuid permission flag functionality!

Its operating principle will be explained in detail in the next dedicated articles.
