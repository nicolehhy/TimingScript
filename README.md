# TimingScript
How to establish data monitoring system using LINUX SCRIPT and HIVE

### Background
Because `Tableau` can only synchronize data in mysql, we need to create a script file in `Hadoop` environment, input the results of SQL code we execute in hive into `TXT` file, and import them into tables in `MySQL` environment, so as to synchronize to tableau.

### Methodology
   * Create a new table in Mysql
    * Create a new file in `hdfs` using hive 
     * Write the quering script `*.sh` in this new file
      * Create a timing task using `Crontab -e`
       * Save the timing task(Then we are done at this step)
      
#### Create new table in Database (MySQL)
  * Log in to Hadoop at first (take XShell for example) and enter MySQL environment in the Xshell first
  ```Linux
  mysql -h rm-2ze6406t9hkur76rk.mysql.rds.aliyuncs.com -P 3306 -u inke_db_user -p # the command to enter Mysql
passward: *******.  # get the right entering password 
  ```
  * Enter the database where you need to create your new table 
  ```Linux
  use db_test; 
  ```
  * Create the new table using SQL language
  ```SQL
  create table liuliang( 
           ymd varchar(15),
           show_pv varchar(10),
           show_count varchar(10),
           show_uv int,
           view_uv int,
           view_pv int,
           short_uv int,
           short_pv int,
           alltime int,
           enter_uv int
          )engine = innodb DEFAULT CHARSET=utf8;   #Setting Engine and Character Set to Prevent Chinese Scrambling
   ```
   
#### Write a script in Hive
Here, what we need to do is to query the data we need and copy them into the new table`liuliang` we just created in Mysql
  * Exit Mysql
  ```SQL
  exit mysql
  ```
  * Enter the command: `pwd`. Get the path to the current directory and create a file to store your output
  ```LINUX
  pwd
  mkdir liuliang
  ```
  
  * Create a new shell script under this directory
Input command: `vi liuliang. sh` (vi command for editing a file). If test. sh already exists, it can be modified; if test. sh does not exist, a new test. sh file will be created automatically.
 ```SQL
 vi liuliang. sh
 ```
  * Input the quering code to this `liuliang.sh` script file
 Here I need to query the daily tracking metrics like `Exposure times`, `number of people exposured`, `number of viewers`, `times of people veiwing the page` 
 ```SQL
 date=`date -d "yesterday" +%Y%m%d` # this is for query limits--I just want to extract the information of yesterday

hive -e"

select b.ymd,a.show_pv,a.show_count,a.show_uv,b.view_uv,b.view_pv,b.short_uv,b.short_pv,b.time,c.enter_uv
from
(select ymd,count(uid) as show_pv,count(distinct token)as show_count,count(distinct uid)as show_uv
from hds.newapplog_seach_live_show
where ymd=$date and token rlike 'rec_9_2_1_0' and cv rlike  '6.0.0' 
group by ymd)a
left join
(select ymd,count(uid) as view_pv,count(distinct uid)as view_uv,count(distinct case when unix_timestamp(exit_time)-unix_timestamp(enter_time)>60 then uid end) as short_uv,count(case when unix_timestamp(exit_time)-unix_timestamp(enter_time)>60 then uid end) as short_pv,sum(unix_timestamp(exit_time)-unix_timestamp(enter_time)) as time
from ana.dw_live_view_info_everyday_enter
where ymd=$date and enter='search_new_rec' and token rlike 'rec_9_2_1_0' and cv rlike  '6.0.0'
group by ymd)b
on a.ymd=b.ymd
left join
(select ymd,sum(uv) as enter_uv
from bi.recommend_rec_basic_os 
where cv rlike'6.0.0' and ymd=$date and level rlike '全部' and logid rlike 'all'
group by ymd)c
on a.ymd=c.ymd

;" >liuliang.txt          
sed -i '/^WARN.*/d' liuliang.txt  # delete the last sentence of output containing 'warning....'
mysql -hrm-2ze6406t9hkur76rk.mysql.rds.aliyuncs.com -P3306 -uinke_db_user -p'Nx$X6^soiPi' -e "LOAD DATA LOCAL INFILE '/home/huanghongyu/liuliang.txt' INTO TABLE db_inke_pm_strategy.liuliang CHARACTER SET utf8 fields terminated by '\t' lines terminated by '\n';"    
```
`mysql -hrm-2ze6406t9hkur76rk.mysql.rds.aliyuncs.com -P3306 -uinke_db_user -p'Nx$X6^soiPi'` this part goes into Mysql database<br>
`-e "LOAD DATA LOCAL INFILE '/home/huanghongyu/liuliang.txt' INTO TABLE db_inke_pm_strategy.liuliang` this part means I put the query result in `liuliang.txt` into the table `liuliang` in Mysql

#### Create the `Crontab` task (Automatic monitoring system)
Performing crontab timing tasks in Linux system on `pwd` path 
  * Enter the command `crontab -e` and enter `i` into the editing state to add a timing task
  ```LINUX
  30 08 * * * sh /home/hongyuhuang/liuliang.sh > /home/hongyuhuang/iiuliang/log_`date -d "yesterday" +%Y%m%d`.log 2>&1
  ```
The meaning of the example is:<br>
Every morning at 8:30, execute the test.sh script in the `/home/hongyuhuang` directory, redirect the output log to the `/home/hongyuhuang/liuliang` directory, and name the file log_yesterday's date.
  * If you don't want the log thing, you only need to type this:
  ```LINUX
  30 08 * * * sh /home/hongyuhuang/liuliang.sh
  ```
  * exit the editting state and save the crontab task
    * After editing, press `esc` to exit the editing status
    * Enter `wq` to save the modified content
      * If you don't want to save the modified content, enter: `q`
    * Timely update settings completed
 
 #### Check the status of your timing task
You can view daily log files in the `/home/hongyuhuang/iiuliang` directory
  * After landing in hadoop, enter the command `cd ../hongyuhuang/liuliang` 
  enter into the test folder (if you want to return to the previous directory, enter the command `cd..`
  * Enter command `ls` to view all files under the `liuliang` folder
  * Download the required log files locally,for example: 
  ```linux
  sz log_20180524.log
  ```
Logging is the process of execution of this script. The purpose of saving the log is to retrieve the results of your SQL query from the log on the one hand. On the other hand, if there is a problem with the data on a certain day, you can locate the problem by looking at the log and troubleshooting errors.
  



