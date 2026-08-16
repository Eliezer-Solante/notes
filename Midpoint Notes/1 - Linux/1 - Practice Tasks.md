###### Copilot said:

# Linux Bash Hands-On Practical Exam

## Part 1 – Linux Administration, Networking, Permissions, Security, Scheduling, and Services (30 Tasks)

### Files, Directories, and Redirection

1. Create a directory named `linux_practical_exam` in your home directory. Inside it, create the subdirectories `projects`, `scripts`, `backups`, `logs`, `reports`, and `network`. Verify the directory structure.
    
2. Create the files `web.txt`, `database.txt`, `security.txt`, and `project_notes.txt` inside `projects`. Rename `project_notes.txt` to `master_project_notes.txt`. Verify the changes.
    
3. Copy `web.txt` to `backups`, move `database.txt` to `backups`, and remove `security.txt`. Verify the changes.
    
4. Create the directory structure `scripts/maintenance/daily` and verify the structure.
    
5. Create a file named `user_report.txt` containing your username, then append the current date and time to the same file. Verify the contents.
    
6. Create a file named `shell_info.txt` using `tee` that contains your current shell and hostname. Verify the contents.
    
7. Create a file named `path_variable.txt` containing the PATH environment variable. Verify the contents.
    
8. Create a file named `commands.txt` containing your command history and a file named `available_commands.txt` containing all commands available in your current shell. Verify both files.
    
9. Create a file named `errors.log` that contains only error messages from an invalid command, then create `combined.log` that contains both standard output and standard error from a command. Verify both files.
    
10. Create a file named `system_report.txt` that combines the output of at least three Linux commands using pipes, redirection, and `tee`. Verify the report.
    

---

### Find, Compression, and Archiving

1. Create a file named `employees.txt` containing at least five employee names, then append three additional names without overwriting existing content. Verify the file.
    
2. Find all `.txt` files inside `linux_practical_exam` and save the results to `txt_files.txt`. Verify the results.
    
3. Find all directories inside `linux_practical_exam` and save the results to `directories.txt`. Verify the results.
    
4. Create an archive named `employees_backup.tar` containing `employees.txt`, extract it into `restore_test`, and verify the extraction.
    
5. Compress `employees.txt` using GZIP, restore the original file, and verify the result.
    
6. Create `documents.zip` containing all `.txt` files in `projects`, extract it into `zip_restore`, and verify the extraction.
    
7. Create a compressed archive named `project_backup.tar.gz` containing the entire `projects` directory and verify its contents.
    

---

### System Information and Networking

1. Create a report named `system_info.txt` containing kernel information, memory usage, CPU information, and filesystem usage. Verify the report.
    
2. Create a file named `hostname_info.txt` containing the system hostname and a file named `interface_report.txt` containing information about available network interfaces. Verify both files.
    

1. Create a file named `route_report.txt` containing the system routing table and verify the report.
    
2. Create a file named `dns_resolution.txt` that contains the IP address of a specified domain and a file named `mx_records.txt` containing its MX records. Verify both files.
    
3. Create a file named `connectivity_test.txt` containing the results of a connectivity test to a specified host. Verify the file.
    
4. Create a backup of `/etc/hosts` named `hosts_backup.txt`, add a temporary hostname-to-IP mapping, and verify that the entry was added successfully.
    
5. Create files named `dns_servers.txt` and `name_resolution_order.txt` containing the DNS server configuration and name resolution order configuration. Verify both files.
    
6. Establish an SSH connection to a remote Linux system and transfer `employees.txt` using SCP. Verify successful login and file transfer.
    

---

### Users, Groups, Permissions, Links, and Aliases

1. Create a user named `examuser1` and a group named `developers`, add the user to the group, create a directory named `shared_workspace`, assign ownership to `examuser1:developers`, configure appropriate permissions, and verify the configuration.
    
2. Create a symbolic link named `latest_employees` and a hard link named `employees_archive` that reference `employees.txt`. Verify both links.
    
3. Create an alias named `ll`, make it available for the current session, and verify its functionality.
    

---

### Cron Jobs, iptables, and systemd

1. Create a script named `daily_backup.sh` inside the `scripts` directory. Configure a cron job that executes it daily at 1:00 AM and another cron job that appends the current date and time to `cron_test.log` every hour. Export the configured cron jobs to `cron_report.txt` and verify the configuration.
    
2. Create a custom systemd service named `heartbeat.service` that runs a Bash script named `heartbeat.sh` and continuously writes timestamps to a log file. Configure it to start automatically at boot and automatically restart on failure. Create firewall rules that allow SSH and ICMP traffic while blocking Telnet traffic. Save the firewall configuration to `firewall_report.txt`. Verify the service and firewall configuration.
    

---

# Part 2 – Disk Partitioning, File Systems, and LVM (20 Tasks)

> Assume an unused disk such as `/dev/sdb` is available.

1. Create a file named `storage_inventory.txt` containing information about the available block devices and verify the report.
    
2. Create a new GPT partition table on the disk and verify the partition table.
    
3. Create a 2 GB partition intended for standard filesystem usage and verify the partition.
    
4. Create a second partition using the remaining disk space and verify the partition layout.
    
5. Create a file named `partition_report.txt` containing the partition information and verify the report.
    
6. Format the first partition using the ext4 filesystem and verify the filesystem creation.
    
7. Create a mount point named `/data`.
    
8. Mount the first partition on `/data` and verify the mount.
    
9. Create a file named `application_data.txt` inside `/data` and verify access to the filesystem.
    
10. Create a file named `uuid_report.txt` containing the UUID of the mounted partition and verify the report.
    
11. Initialize the second partition as an LVM Physical Volume and verify the configuration.
    
12. Create a Volume Group named `vgdata` using the Physical Volume and verify the configuration.
    
13. Create a 1 GB Logical Volume named `lvbackup` within `vgdata` and verify the Logical Volume.
    
14. Format `lvbackup` using the ext4 filesystem and verify the filesystem.
    
15. Create a mount point named `/backup_storage`.
    
16. Mount `lvbackup` on `/backup_storage`, create a file named `backup_data.txt`, and verify access.
    
17. Extend `lvbackup` using available free space within `vgdata` and verify the new Logical Volume size.
    
18. Resize the filesystem on `lvbackup` to utilize the newly allocated space and verify the filesystem size.
    
19. Create the files `pv_report.txt`, `vg_report.txt`, and `lv_report.txt` containing information about the configured Physical Volumes, Volume Groups, and Logical Volumes. Verify all reports.
    
20. Configure persistent mounting for both `/data` and `/backup_storage` using `/etc/fstab`, then test the configuration by unmounting and remounting via the configuration file. Verify that both filesystems mount successfully.