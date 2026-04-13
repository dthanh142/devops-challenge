## Diagnose steps: 
Run below command to get overall disk status: 
- `df -h`: get total disk usage to determine which partition is running low in free space.
- `du -sh / --max-depth=2 | sort -rh | head -20`: to determine which directory/folder consuming the most disk space, upto 2 levels down - then narrow down to most disk consume directory.
- `du-sh /var/log/nginx /var/cache/nginx /var/lib/nginx /ect/nginx/tmp` : check directory size specifically for nginx 
- `df -i`: show inode usage. Exhaust free inodes behaves just like full disk space.
- `find / -type f -size +300M -exec ls -lh {} \; 2>/dev/null`: find uncommonly large files 
- `lsof +L1`: showing hidden deleted files that consume space. Need process restart after deleting the file.
- `ps auxf`: list all running process and find the suspicious one that could consuming the disk.


## Possible root cause expected to encounter:
 ### 1. Most likely: large nginx log files:
  - Access log or error log can grow quickly if logrotate or file limit not set correctly 
  - Debug log may turned on -> large number of log entries 
  - Deleted file still being held by the process 

  #### <u>Diagnose:</u> 
  -  With `du -sh /var/log/nginx` 
  -  Access.log exceed GB+ 
  -  Cat `/ect/logrotate.d/nginx` -> missing or misconfigured 
  -  `lsof +L1` show delted files held open
  
  #### <u>Impact:</u>
  - Service unavailability: nginx refuse new connections 
  - No new logs -> no audit data 
  - Affect other process 
  - Prevent SSH 
  
  #### <u>Fix: </u>
  - Configure logrotate correctly 
  - Turn off debug log on production server 
  - Ship log to central log system
  - Truncate log file: truncate-s 0/var/log/nginx/access.log 
  - Restart nginx service is the last resort to solve deleted files held by the process 
  - Set cronjob to periodically clean up the logs 
  
  ---
  ### 2. Inode exhaustion: 
  - Disk space may free, but running out of inode if tiny files are generated uncontrollably ( cache files, temp files, logs...) 
  
  #### <u>Diagnosis: </u>
  - `df -i ` showing inode usage
  
  #### <u>Impact: </u>
  - can not create new file, nginx can not open new connection 
  
  #### <u>Fix: </u>
  - Zipping small related files into a single tar file and delete them to save space 
  - Delete unwanted small files/directory, especially temp, logs, cache ... 
  - Check any script running periodically that create multiple junk

---
### 3. Large core dumps: 
- Core dumps will be stored at `/var/lib/systemd/coredume `

#### Diagnos: 
- `du -sh /var/lib/systemd/coredump `

#### Impact: 
Nginx crash repeatedly will write a large core dumps, also indicate that nginx has problem 

#### Fix: 
assume that coredumpctl is installed 
`coredumpct list` 
`coredumpct info`: to investigate nginx crash reason
`rm -rf /var/lib/systemd/coredump`