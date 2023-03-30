
# Project : Using Apache Hudi Deltastreamer and AWS DMS Hands on Labs
![image](https://user-images.githubusercontent.com/39345855/228927370-f7264d4a-f026-4014-9df4-b063f000f377.png)



------------------------------------------------------------------
Video Tutorials 
Part 1: Project Overview : https://www.youtube.com/watch?v=D9L0NLSqC1s
Part 2: Aurora Setup : https://youtu.be/HR5A6iGb4LE

# Steps 
### Step 1:  Create Aurora Source Database and update the seetings to enable CDC on Postgres 
*  Read More : https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Source.PostgreSQL.html
* Create new Create parameter group as shows in video and make sure to edit these two setting as shown in video part 2
```
rds.logical_replication  1
wal_sender_timeout   300000
```
once done please apply these to database and reboot your Database 


