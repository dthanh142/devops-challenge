## Diagnose steps: 
Run below commands to get overall disk status: 
- `df -h`

  Get disk usage overview to determine which partition is running low in free space.
---
- `du -xh / --max-depth=2 | sort -rh | head -20` 

  To determine which directory/folder in the partition consuming the most disk space, upto 2 levels down - then narrow down to most disk consume directory. Most common places are nginx related directories:

   `du -xh /var/log/nginx /var/cache/nginx /var/lib/nginx /ect/nginx /tmp` 

  To get directory size specifically for nginx
---
- `df -ih`
 
  Show inode usage. Exhaust free inodes behaves just like full disk space. 

  Then follow up with command to find which directory has the most files:

  `find / -type f -printf "%h\n" 2>/dev/null | sort | uniq -c | sort -rn | head -20`
---
- `find / -type f -size +300M -exec ls -lh {} \;`

  To find uncommonly large files across the filesystem. 
---
- `lsof +L1`

   To show which process is holding the deleted files. If the large file has been deleted but disk space still running low, meaning the process holding that file will need to be restarted to release the space it consumed.


---

## Possible root cause expected to encounter:
 ### 1. Most likely: large nginx log/cache files:
  - Access log or error log can grow quickly if logrotate not set correctly 
  - Debug log may turned on -> large number of log entries 
  - Deleted file still being held by the process 

  #### <u>Diagnose:</u> 
  -  With `du -xh /var/log/nginx /var/cache/nginx` 
  -  Access.log exceed GB+ 
  -  Cat `/ect/logrotate.d/nginx` -> missing or misconfigured 
  -  `lsof +L1` show delted log file held open by nginx
  
  #### <u>Impact:</u>
  - Service unavailability: nginx refuse new connections 
  - No new logs -> no audit data 
  - Affect other process 
  - Prevent SSH 
  
  #### <u>Fix: </u>
  - Configure logrotate correctly 
  - Turn off debug log on production server 
  - Ship log to central log system
  - Truncate log file: `truncate -s 0 /var/log/nginx/access.log `
  - Cleanup cache: `find /var/cache/nginx/ -type f -print0 | xargs -0 -n 1000 rm -f`
  - Restart nginx service is the last resort to solve deleted files held by the process 
  - Set cronjob to periodically clean up the logs 
  
  ---
  ### 2. Inode exhaustion: 
  - Disk space may free, but running out of inodes if tiny files are generated uncontrollably ( cache files, temp files, logs...) 

  
  #### <u>Diagnosis: </u>
  - `df -i ` showing inode usage
  - `find / -type f -printf "%h\n" 2>/dev/null | sort | uniq -c | sort -rn | head -20` : showing whcih directory has the most files.
  
  #### <u>Impact: </u>
  - can not create new file, nginx can not open new connection 
  
  #### <u>Fix: </u>
  - Zipping small related files into a single tar file and delete them to save space 
  - Delete unwanted small files/directory, especially temp, logs, cache ... 
  - Check any script running periodically that create multiple junk files.

---
### 3. Large core dumps: 
- Core dumps will be stored at `/var/lib/systemd/coredump `
- System crash will leave a coredump as a snapshot to troubleshoot. They can be very large and fill up the filesystem quickly.

#### <u>Diagnos: </u>
- `du -sh /var/lib/systemd/coredump `
- `find / -type f -name 'core*' -exec ls -lh {} \;`

#### <u>Impact: </u>
Nginx crash repeatedly will write a large core dumps, also indicate that nginx has problem 

#### <u>Fix: </u>
assume that coredumpctl is installed 

`coredumpct list` : list all coredumps

`coredumpct info`: to investigate nginx crash reason

`rm -rf /var/lib/systemd/coredump`

---
### 4. Accumulated Journald log:
- Journald persists logs for all services, and on a long running VM it can grow up to GB+ log

#### <u>Diagnos: </u>
- `journalctl --disk-usage `
- `du -sh /var/log/journal/`

#### <u>Impact: </u>
System services fail to write, risking full outage 

#### <u>Fix: </u>
- `journalctl --vacuum-size=500M`
- Change the SystemMaxUse parameter in `/etc/systemd/journald.conf`
