# Chapter 3: Linux Fundamentals & Production-Grade Shell Scripting

## Abstract

For any DevOps professional, the Linux operating system and its command-line interface are not just tools; they are the environment. A superficial understanding is a career-limiting liability. FAANG-level interviews test for a deep, intuitive grasp of the Linux philosophy, its core architecture, and the ability to wield its vast array of utilities to diagnose, debug, and automate complex systems under pressure. This chapter provides a definitive, book-level exploration of these fundamentals. We will deconstruct the Filesystem Hierarchy Standard, master powerful text-processing pipelines, delve into system observability with tools like `strace` and `lsof`, and, most critically, learn the principles of writing robust, production-grade shell scripts that are safe, reliable, and maintainable.

---

### Part 1: The Linux Philosophy and Core Architecture

Linux's design is guided by a powerful, minimalist philosophy inherited from its Unix predecessors. Internalizing these principles is the first step toward mastery.

1.  **Everything is a File:** In Linux, nearly every interface to the system is represented as a file-like object in the filesystem. This is a profound abstraction. Your hard drives (`/dev/sda`), your terminal sessions (`/dev/pts/0`), kernel data (`/proc/cpuinfo`), and even network connections are all presented as files that can be read from and written to using the same standard I/O utilities. This unifies the system and makes it incredibly scriptable.

2.  **Small, Sharp Tools:** The Unix philosophy advocates for creating programs that do one thing and do it well. Complex tasks are not solved by monolithic, all-in-one applications but by composing these small, sharp tools together into powerful pipelines. A command like `grep` only finds lines that match a pattern. `sort` only sorts lines. `uniq` only removes adjacent duplicate lines. But chaining them together (`cat log.txt | grep "ERROR" | sort | uniq -c`) creates a sophisticated data processing workflow.

3.  **Write Programs to Handle Text Streams:** Because text is a universal interface, tools are designed to read from a standard input stream (`stdin`) and write to a standard output stream (`stdout`). This allows for the elegant composition mentioned above. Error messages are sent to a separate stream, standard error (`stderr`), so they don't pollute the primary data output.

#### The Filesystem Hierarchy Standard (FHS)

The FHS is the "map" of a Linux system. Knowing this map is non-negotiable. It provides a standardized layout, ensuring that users and scripts can reliably predict the location of files and executables.

*   `/`: The root directory. The top of the entire hierarchy.
*   `/bin`: Essential user command binaries (e.g., `ls`, `cp`, `cat`). Available before `/usr` is mounted.
*   `/sbin`: Essential system binaries (e.g., `reboot`, `fdisk`, `iptables`).
*   `/etc`: **(Editable Text Configuration)**. Host-specific system configuration files. This is one of the most important directories for a system administrator.
*   `/dev`: Device files. This is where the "everything is a file" principle is most visible.
*   `/proc`: A virtual filesystem providing an interface to kernel data structures. It's a window into the live state of the system. For example, `cat /proc/meminfo` shows current memory usage.
*   `/var`: Variable files. Files whose content is expected to grow during the normal operation of the system. This includes logs (`/var/log`), mail spools, and web server content.
*   `/tmp`: Temporary files. Often cleared on reboot.
*   `/usr`: **(Unix System Resources)**. This is one of the largest directories, containing user-facing programs, libraries, and documentation.
    *   `/usr/bin`: Non-essential command binaries for all users.
    *   `/usr/sbin`: Non-essential system binaries.
    *   `/usr/local`: The designated area for software compiled and installed locally by the administrator, to keep it separate from the distribution's packaged software.
*   `/home`: User home directories.
*   `/boot`: Static files of the boot loader (e.g., the Linux kernel itself).
*   `/lib`: Essential shared libraries needed by the binaries in `/bin` and `/sbin`.
*   `/opt`: Optional application software packages.

---

### Part 2: The Power Tools - Beyond `ls` and `cd`

Mastery of the command line comes from fluency with tools that can manipulate and process data.

*   **`find`:** The ultimate tool for searching for files based on their attributes.
    *   `find . -type f -name "*.log"`: Find all files ending in `.log` in the current directory and subdirectories.
    *   `find /var/log -type f -mtime +7 -delete`: Find and delete all files in `/var/log` that haven't been modified in more than 7 days.
    *   `find . -type f -size +100M`: Find files larger than 100MB.
    *   `find . -type f -perm 644`: Find files with exactly the permission `644`.

*   **`grep` (Global Regular Expression Print):** The tool for searching for patterns within files.
    *   `grep -r "ERROR" /var/log/`: Recursively search for "ERROR" in all files under `/var/log`.
    *   `grep -v "DEBUG"`: Invert the match, showing only lines that *do not* contain "DEBUG".
    *   `grep -E "^[0-9]{1,3}\.[0-9]{1,3}"`: Use extended regular expressions to find lines that start with an IP-like pattern.

*   **`sed` (Stream Editor):** For performing text transformations on an input stream.
    *   `sed 's/foo/bar/g' file.txt`: The classic substitution. Replace every instance of 'foo' with 'bar'. The `g` means global (all occurrences on a line).
    *   `sed '/^#/d' /etc/hosts`: Delete (`d`) all lines that start with (`^`) a `#` (comments).

*   **`awk`:** A powerful pattern-scanning and processing language. It processes a file line by line, splitting each line into fields.
    *   `df -h | awk '{print $1, $5}'`: Print the first and fifth columns of the `df` output (Filesystem and Use%).
    *   `awk -F':' '{print $1}' /etc/passwd`: Use `:` as a field separator (`-F`) and print the first field (the username).
    *   `awk '$3 > 1000 {print $0}' access.log`: Print the entire line (`$0`) if the third field is greater than 1000.

*   **`xargs`:** Builds and executes command lines from standard input. It's the glue that connects `find` to other commands.
    *   `find . -name "*.tmp" | xargs rm`: The classic (but unsafe) way to delete files. If a filename contains a space, this will break.
    *   **The Safe Way:** `find . -name "*.tmp" -print0 | xargs -0 rm`: The `-print0` action in `find` separates filenames with a null character. The `-0` flag in `xargs` tells it to expect null-separated input. This is the robust way to handle any possible filename.

---

### Part 3: System Internals and Live Observability

This is what separates a sysadmin from a systems engineer. You must be able to inspect and understand a running system.

*   **Processes:** A process is an instance of a running program. The kernel tracks it via a Process ID (PID).
    *   `ps aux`: The canonical command to list all running processes.
    *   `pstree`: Shows processes in a tree structure, revealing parent-child relationships, which is excellent for debugging.

*   **System Calls (`strace`):** The bridge between a user-space process and the kernel. When a program needs to do anything meaningful—open a file, send data over the network, read from the disk—it must ask the kernel by making a system call. `strace` is a powerful diagnostic tool that intercepts and logs these system calls.
    *   **Use Case:** An application is failing to start with a "Permission Denied" error, but you don't know what file it's trying to access.
    *   **Solution:** `strace -f -e trace=open,openat my_application`
        *   `-f`: Follow any child processes that are forked.
        *   `-e trace=open,openat`: Trace only the `open` and `openat` system calls, which are used to open files.
        *   The output will show you exactly which file path is returning the `EACCES` (Permission Denied) error, instantly revealing the source of the problem.

*   **Open Files (`lsof` - List Open Files):** In Linux, everything is a file. `lsof` lists all files currently opened by processes.
    *   **Use Case:** You try to unmount a filesystem and get the error "target is busy."
    *   **Solution:** `lsof /path/to/mountpoint`. This will show you the exact command and PID of the process that is holding a file open on that mount, preventing it from being unmounted.
    *   `lsof -i :80`: Show which process is listening on TCP port 80.

*   **Disk I/O (`iostat`):** Reports CPU statistics and input/output statistics for devices.
    *   **Use Case:** Users are reporting that an application is slow. You suspect a disk bottleneck.
    *   **Solution:** `iostat -xz 1`. This will show you detailed disk statistics every second (`1`). Key metrics to watch are `%util` (the percentage of time the device was busy, a high value indicates saturation) and `await` (the average time for I/O requests to be served, a high value indicates the disk is struggling to keep up).

*   **Network (`netstat`, `ss`):**
    *   `netstat` is the classic tool, but `ss` (socket statistics) is its modern, faster replacement.
    *   `ss -tulnp`: A great combination of flags.
        *   `-t`: Show TCP sockets.
        *   `-u`: Show UDP sockets.
        *   `-l`: Show only listening sockets.
        *   `-n`: Don't resolve service names (show port numbers).
        *   `-p`: Show the process using the socket.
    *   This command gives you an immediate picture of all the services listening for network connections on your server.

---

### Part 4: Production-Grade Shell Scripting

Writing a simple script is easy. Writing a script that is safe to run in a production environment is an engineering discipline.

#### The Unofficial Bash Strict Mode

Always start your production scripts with this "magic" line:
`set -euo pipefail`

*   `set -e` (or `set -o errexit`): Exit immediately if a command exits with a non-zero status. This prevents your script from continuing in an unpredictable state after a failure.
*   `set -u` (or `set -o nounset`): Treat unset variables as an error when substituting. This catches typos in variable names.
*   `set -o pipefail`: This is critical. By default, the exit code of a pipeline is the exit code of the *last* command. With `pipefail`, the exit code is the exit code of the rightmost command to exit with a non-zero status, or zero if all commands succeed. This ensures that failures in the middle of a pipeline are caught by `set -e`.

#### Temporary Files and Cleanup

Never hardcode temporary files in `/tmp`. This is insecure and can lead to race conditions.

*   **`mktemp`:** The correct tool for creating secure temporary files and directories.
    *   `tmpfile=$(mktemp)`: Creates a unique temporary file.
    *   `tmpdir=$(mktemp -d)`: Creates a unique temporary directory.

*   **`trap`:** A command that allows you to execute a command when your script receives a signal. This is essential for cleanup.

**Putting it all together - A Robust Script Template:**

```bash
#!/bin/bash
set -euo pipefail

# Create a temporary directory
tmpdir=$(mktemp -d)

# Define a cleanup function
cleanup() {
    echo "Cleaning up temporary directory: $tmpdir"
    rm -rf "$tmpdir"
}

# Set the trap. This will call the cleanup function on EXIT, ERR, or INT signals.
trap cleanup EXIT ERR INT

# --- Main script logic goes here ---
echo "Doing work in $tmpdir"
touch "$tmpdir/my_file.txt"
sleep 10
echo "Work finished"
# --- End of main script logic ---

# The trap will automatically handle cleanup when the script exits.
```

---

### ★ FAANG-Level Interview Scenarios ★

*   **Scenario 1: The Mystery Process**
    *   **Question:** "You log into a server and run `top`. You see a process named `kworker` that is consuming 99% of a CPU core. How do you determine if this is a legitimate kernel worker or something malicious? What is this process actually doing?"
    *   **Answer:** "`kworker` processes are legitimate kernel threads that handle work queued up by the kernel, such as timers, I/O, and interrupts. High CPU usage can be normal, but it can also indicate a problem, like a driver bug or a hardware issue.
        1.  **Initial Triage:** First, I'd use `top` and press `1` to see the per-CPU breakdown. Is it one core at 100% or spread across many? This helps narrow the focus.
        2.  **Kernel Symbol Profiling:** The key to understanding what a kernel thread is doing is to profile it. I would use `perf`, the Linux profiler. The command `perf record -g -a sleep 10` would collect system-wide stack traces for 10 seconds. Then, `perf report` would show me a breakdown of which kernel functions are consuming the most CPU time. If I see a specific function, like one related to a network driver or a filesystem, at the top of the list, that's my primary suspect.
        3.  **Tracing with `strace` is not useful here**, because `kworker` is a kernel thread and doesn't make system calls back to itself. `perf` is the correct tool for kernel-space investigation.
        4.  **Check Logs:** I would also immediately check `dmesg` and `/var/log/syslog` for any related error messages from drivers or hardware that might be causing the kernel to spin. If it were a malicious process masquerading as `kworker`, its behavior would likely be different—it might have open network connections or be accessing user files, which I could detect with `ss -p` or `lsof`."

*   **Scenario 2: The Disk Space Black Hole**
    *   **Question:** "You run `df -h` and see that the root filesystem (`/`) is 95% full. You run `du -sh /` and it reports that only 60% of the disk is being used by files. There is a 35% discrepancy. What is the most likely cause of this, and how do you fix it?"
    *   **Answer:** "This is a classic Linux problem. The discrepancy almost always means that a process has a file open that has been deleted from the filesystem.
        *   **The Explanation:** When you `rm` a file in Linux, you are only unlinking its name from the filesystem directory structure. The actual data blocks on the disk are not freed until the last process that has an open file handle to that file closes it. The `du` command walks the filesystem tree, so it can't see the deleted file. The `df` command, however, queries the filesystem's superblock, which knows the true number of allocated blocks.
        *   **The Solution:** The tool to solve this is `lsof` (list open files). I would run the command `lsof +L1`. The `+L1` option specifically tells `lsof` to find files with a link count less than 1—in other words, deleted files that are still held open by a process.
        *   The output will show me the command, the PID of the process, the file descriptor number, and the name of the deleted file (usually appended with `(deleted)`).
        *   **Remediation:** The fix is to get that process to close its file handle. The safest way is to gracefully restart the service associated with the process. For example, if it's the `nginx` process holding a deleted log file open, I would run `systemctl restart nginx`. As soon as the old process terminates, the kernel will release the file handle, and the disk space will be freed. If I can't restart it, killing the specific PID would be the last resort."

*   **Scenario 3: The Unreliable Script**
    *   **Question:** "Review this shell script. It's supposed to back up a directory, but it sometimes fails silently. How would you rewrite it to be production-ready?"
        ```bash
        #!/bin/bash
        # Script to back up my app data
        cd /home/user/appdata
        tar -czf /mnt/backups/app-backup.tar.gz .
        echo "Backup complete."
        ```
    *   **Answer:** "This script has several critical flaws that make it unsafe for production.
        1.  **No Error Handling:** If the `cd` command fails (e.g., the directory doesn't exist), the script will continue. The `tar` command will then run from whatever directory the script was launched from, potentially backing up the wrong data or, worse, the entire root filesystem if run from `/`. If the `tar` command fails (e.g., `/mnt/backups` is full or not mounted), the script will still print 'Backup complete.'
        2.  **Unsafe `cd`:** Relying on `cd` is fragile.
        3.  **Static Filename:** It overwrites the same backup file every time.

        Here is how I would rewrite it for production robustness:
        ```bash
        #!/bin/bash
        # Production-ready backup script for app data
        set -euo pipefail

        # --- Configuration ---
        # Use descriptive variable names.
        # Use readonly for constants to prevent accidental modification.
        readonly SOURCE_DIR="/home/user/appdata"
        readonly BACKUP_DIR="/mnt/backups"
        # Create a timestamped filename.
        readonly FILENAME="app-backup-$(date +%Y-%m-%dT%H-%M-%S).tar.gz"
        readonly DEST_PATH="$BACKUP_DIR/$FILENAME"

        # --- Pre-flight Checks ---
        # Check if the source directory exists and is a directory.
        if [[ ! -d "$SOURCE_DIR" ]]; then
            echo "Error: Source directory $SOURCE_DIR does not exist." >&2
            exit 1
        fi

        # Check if the backup directory exists and is writable.
        if [[ ! -d "$BACKUP_DIR" ]] || [[ ! -w "$BACKUP_DIR" ]]; then
            echo "Error: Backup directory $BACKUP_DIR is not a writable directory." >&2
            exit 1
        fi

        # --- Main Logic ---
        echo "Starting backup of $SOURCE_DIR to $DEST_PATH"
        # Use the -C option in tar to change directory safely for just this command.
        # This is far more robust than a separate 'cd' command.
        tar -C "$SOURCE_DIR" -czf "$DEST_PATH" .

        echo "Backup successfully completed: $DEST_PATH"
        # Optional: Prune old backups
        # find "$BACKUP_DIR" -name "app-backup-*.tar.gz" -mtime +30 -delete
        ```
        This version includes `set -euo pipefail` for strict error handling, uses variables for configuration, performs pre-flight checks to validate its environment, creates unique timestamped backups, and uses `tar -C` for a much safer directory change. It provides meaningful error messages to `stderr` and will only report success if the `tar` command actually succeeds."
