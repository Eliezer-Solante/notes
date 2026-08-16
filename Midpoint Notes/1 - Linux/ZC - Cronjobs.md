![[Pasted image 20260716102933.png]]
The goal is that the command `uptime >> /tmp/system-report.txt` would automatically run everyday at 9 PM by using CRON or crond service.

![[Pasted image 20260716103656.png]]
NOTE: do not use `sudo` command in using crontab as it will schedule the job for the root user.

![[Pasted image 20260716103956.png]]

step time `*/[value(minute,hour,day,month,weekday)]` 
    - example:  if you want to run the command every other 2 minutes add on the minute field `*/2`.
    ![[Pasted image 20260716104330.png]]

So,
![[Pasted image 20260716104413.png]]

reference table:
![[Pasted image 20260716104445.png]]


Job checking and verifying
![[Pasted image 20260716104635.png]]
