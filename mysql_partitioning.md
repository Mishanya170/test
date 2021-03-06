

{{note|This guide is not complete yet}}




== Yet Another MySQL Partitioning (YAMP) ==

This MySQL partitioning guide was originally prepared by DotNeft.

Partitioning is performed by logically dividing one large table into small physical fragments. Partitioning may bring several advantages:

* In some situations query performance can be significantly increased, especially when the most intensively used table area is a separate partition or a small number of partitions. Such a partition and its indexes are more easily placed in the memory than the index of the whole table.
* When queries or updates are using a large percentage of one partition, the performance may be increased simply through a more beneficial sequential access to this partition on the disk, instead of using the index and random read access for the whole table. In our case the B-Tree (itemid, clock) type of indexes are used that substantially benefit in performance from partitioning.
* Mass INSERT and DELETE can be performed by simply deleting or adding partitions, as long as this possibility is planned for when creating the partition. The ALTER TABLE statement will work much faster than any statement for mass insertion or deletion.
* It is not possible to use tablespaces for InnoDB tables in MySQL. You get one directory - one database. Thus, to transfer a table partition file it must by physically copied to another medium and then referenced using a symbolic link.
{{note|Starting with [http://dev.mysql.com/doc/refman/5.6/en/tablespace-placing.html MySQL 5.6] there is a possibility of specifying the location of a tablespace. However, [http://dev.mysql.com/doc/refman/5.6/en/tablespace-placing.html there are some limitations].}}

These benefits usually become apparent only when a table gets very large. Whether a table is going to benefit from partitioning ultimately depends on the application, however, there is a rule of thumb that it will benefit once the size of the table exceeds the memory size of the database server.

Currently MySQL supports partitioning out of the box. Partitioning support starts with the MySQL 5.1 version, so if you have an older version of MySQL you will have to upgrade it. 
Additionally, MySQL server must be compiled with partitioning support. Whether this support currently exists can be verified with:

for MySQL (before v5.6 only):
<pre>
SELECT variable_value FROM information_schema.global_variables WHERE variable_name = 'have_partitioning';
</pre>
for MySQL v5.6+:
<pre>
SELECT plugin_status FROM information_schema.plugins WHERE plugin_name = 'partition';
</pre>
Make sure you get YES or ACTIVE.

=== Partitioning types in MySQL ===

The following partitioning types can be performed in MySQL:
* Range partitioning
: A table is divided by ranges set in the key column or column list; the value ranges designated for separate partitions must not overlap. For example, ranges of dates or ranges of separate business object identifiers.
* Other partitioning types
: There are also hash, list and key types of partitioning. Discussing these partitioning types would be outside the scope of this article, as we are going to perform only range partitioning. Similarly we are not going to discuss subpartitioning.

=== Choosing a way to manage partitions ===

Two solutions to manage partitions are provided here:
* using [[#Partitioning_with_stored_procedures|MySQL procedures]]
* using an external script
While stored procedures might seem like a more contained solution, debugging any problems might be notably more complicated. Currently, external script approach is suggested.

=== Historical data in Zabbix ===

There are several tables in Zabbix that keep various types of historical data. If you know the purpose of each table, it becomes easier to estimate the periods that needs to be stored. 
Below is a list of tables for keeping data gathered by items.

{| class=wikitable
|+
! !! Purpose !! Data type !! Maximum size
|-
!scope="row"|history
| Keeps raw history || Numeric (float) || double(16,4) - 999999999999.9999
|-
!scope="row"|history_uint 
| Keeps raw history   || Numeric (unsigned) || bigint(20) - 2<sup>64</sup>+1 
|-
!scope="row"|history_str
| Keeps raw short string data || Character || varchar(255) - 255
|-
!scope="row"|history_text
| Keeps raw long string data || Text || text - 65535 
|-
!scope="row"|history_log
| Keeps raw log strings || Log || text - 65535 
|-
!scope="row"|trends
| Keeps reduced dataset (trends) || Numeric (float) || double(16,4) - 999999999999.9999 
|-
!scope="row"|trends_uint
| Keeps reduced dataset (trends) || Numeric (unsigned)  || bigint(20) - 2<sup>64</sup>+1 
|}


{{note|Data gathered by items with {{guitext|Character}}, {{guitext|Log}} and {{guitext|Text}} type of information, kept in '''history_str''', '''history_log''' and '''history_text''' tables, don't have trends. This fact must be considered when partitioning these tables.}}

Tables '''trends''' and '''trends_uint''' are used for reduced dataset, collected for one-hour periods for each item. Each record contains the average of the hour, the minimum and maximum value of the hour, selected from the complete history of that hour as well as the number of raw values in that hour.

There are also other tables in Zabbix for keeping data not generated by items. Let us have a look at these.

{| class=wikitable
|+
! !! Purpose
|-
!scope="row"|acknowledges
| Keeps acknowledgement messages in response events
|-
!scope="row"|alerts
| Keeps notification history 
|-
!scope="row"|auditlog
| Keeps user action history
|-
!scope="row"|events
| Keeps event data. For example, trigger firing, new device discovered 
|-
!scope="row"|service_alarms
| Keeps history of [https://www.zabbix.com/documentation/2.2/manual/it_services IT service] state changes
|}

note: these 5 tables do not have much sense to be partitioned starting from zabbix 2.2, see more details below.

=== Partitioning decisions ===

Before performing partitioning in Zabbix, several aspects must be considered: 
# Range partitioning will be used for table partitioning. 
# Housekeeper will not be needed for some data types anymore. This Zabbix functionality for clearing old history and trend data from the database can be controlled in {{guiloc|Administration -> General -> Housekeeping}}.
# The values of {{guitext|History storage period (in days)}} and {{guitext|Trend storage period (in days)}} fields in item configuration will not be used anymore as old data will be cleared by the range i.e. the whole partition. They can (and should be) overridden in {{guiloc|Administration -> General -> Housekeeping}} - the period should match the period for which we are expecting to keep the partitions.
# If data need to be stored for a longer period, while disk space is limited, it is possible to make use of [http://dev.mysql.com/doc/refman/5.1/en/symbolic-links-to-tables.html symbolic links] for outdated partitions. To verify whether this functionality is present in MySQL, run:
<pre>
mysql> show variables like 'have_symlink';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| have_symlink  | YES   |
+---------------+-------+
1 row in set (0.00 sec)
</pre>
{{note|It is strongly discouraged to make use of the symbolic links feature. Symbolic links hardly work [http://www.mysqlperformanceblog.com/2010/12/25/spreading-ibd-files-across-multiple-disks-the-optimization-that-isnt/ correctly] with any table, apart from MyISAM, and Zabbix uses InnoDB}}.
# Even with the housekeeping for items disabled, Zabbix server and web interface will keep writing housekeeping information for future use into the housekeeper table. To avoid this, you can set ENGINE = [http://dev.mysql.com/doc/refman/5.0/en/blackhole-storage-engine.html Blackhole] for this table:
  ALTER TABLE housekeeper ENGINE = BLACKHOLE;
{{note|In some MySQL compilations the BlackHole storage engine is not present by default. To verify whether it is present, execute the SHOW ENGINES statement.}}

=== Partitioning Zabbix database ===

A tutorial for automatic table partitioning follows. There will be partitioning examples based on the procedures and Event Scheduler of MySQL as well as using a wrapper script. The drawing illustrates a partitioning-by-the-table example, you, however, may choose the type of partitioning (by-the-month or by-the-day) that fits your preferences.

{| class=wikitable
|+
!Daily partitioning !! Monthly partitioning
|-
|history
history_log

history_str

history_text

history_uint
| acknowledges
alerts

auditlog

events

service_alarms

trends

trends_uint
|}

==== Getting Ready ====
Ensure first that the event scheduler is enabled.

<tt><pre>
mysql>  SHOW GLOBAL VARIABLES LIKE 'event_scheduler';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| event_scheduler | OFF   |
+-----------------+-------+
1 row in set (0.00 sec)

mysql>  SET GLOBAL event_scheduler = ON;
Query OK, 0 rows affected (0.00 sec)
</pre></tt>

You should also put a line in the '<tt>my.cnf</tt>' file like "<tt>event_scheduler=ON</tt>" in case of reboot.


{{note|If the tables for partitioning are large then SQL queries below may take a while!}}

1. Originally partitioning is started to be used for zabbix 1.8. To keep details as clear as possible here are mentioned differences between versions.
Depending on used zabbix version different tables are suggested to be partitioned. Basically it's because zabbix 2.2 got improved [https://www.zabbix.com/documentation/2.2/manual/introduction/whatsnew220#finer_control_over_housekeeping_tasks housekeeper control] and as a result only big historical tables now are reasonable to be partitioned.
For versions before 2.2 - more tables suggested to be partitioned (say, all the tables mentioned), for 2.2 and next - only big historical tables (history, history_uint, history_str, history_text, history_log, trends, trends_uint).

Since there are internal MySQL limitations regarding the use of unique indexes, primary keys, etc, before starting the actual partitioning several indexes have to be altered in the Zabbix database.

For zabbix 1.8

<syntaxhighlight lang=sql>
ALTER TABLE `acknowledges` DROP PRIMARY KEY, ADD KEY `acknowledges_0` (`acknowledgeid`);
ALTER TABLE `alerts` DROP PRIMARY KEY, ADD KEY `alerts_0` (`alertid`);
ALTER TABLE `auditlog` DROP PRIMARY KEY, ADD KEY `auditlog_0` (`auditid`);
ALTER TABLE `events` DROP PRIMARY KEY, ADD KEY `events_0` (`eventid`);
ALTER TABLE `service_alarms` DROP PRIMARY KEY, ADD KEY `service_alarms_0` (`servicealarmid`);
ALTER TABLE `history_log` DROP PRIMARY KEY, ADD INDEX `history_log_0` (`id`);
ALTER TABLE `history_log` DROP KEY `history_log_2`;
ALTER TABLE `history_text` DROP PRIMARY KEY, ADD INDEX `history_text_0` (`id`);
ALTER TABLE `history_text` DROP KEY `history_text_2`;
</syntaxhighlight>

Because of foreign keys introduced in Zabbix 2.0 next queries have to be additionally performed to partition all the suggested tables in Zabbix 2.0: 

<syntaxhighlight lang=sql>
ALTER TABLE `acknowledges` DROP FOREIGN KEY `c_acknowledges_1`, DROP FOREIGN KEY `c_acknowledges_2`;
ALTER TABLE `alerts` DROP FOREIGN KEY `c_alerts_1`, DROP FOREIGN KEY `c_alerts_2`, DROP FOREIGN KEY `c_alerts_3`, DROP FOREIGN KEY `c_alerts_4`;
ALTER TABLE `auditlog` DROP FOREIGN KEY `c_auditlog_1`;
ALTER TABLE `service_alarms` DROP FOREIGN KEY `c_service_alarms_1`;
ALTER TABLE `auditlog_details` DROP FOREIGN KEY `c_auditlog_details_1`;
</syntaxhighlight>

For Zabbix 2.2 and later versions only these queries are required:

<syntaxhighlight lang=sql>
ALTER TABLE `history_log` DROP PRIMARY KEY, ADD INDEX `history_log_0` (`id`);
ALTER TABLE `history_log` DROP KEY `history_log_2`;
ALTER TABLE `history_text` DROP PRIMARY KEY, ADD INDEX `history_text_0` (`id`);
ALTER TABLE `history_text` DROP KEY `history_text_2`;
</syntaxhighlight>

For Zabbix 3.2 and later versions any of those changes listed above are NOT required.

2. Now it's time to start partition for each table. 
As partitioning is usually performed for a database with existing historical data - for every table you must specify partitions starting from a minimum value of the '''clock''' field and up to the current moment (day, month) of tables to be partitioned. 
The minimum value of the clock in a table can be found out by a query like this: 

<syntaxhighlight lang=sql>
SELECT FROM_UNIXTIME(MIN(clock)) FROM `history_uint`;
</syntaxhighlight>

perform similar queries for all the tables you are going to partition.

{{note|Total number of partitions for a table [https://dev.mysql.com/doc/refman/5.1/en/partitioning-limitations.html may not exceed 1024] due to an internal MySQL limitation. This limit has been increased to 8192 since MySQL [https://dev.mysql.com/doc/refman/5.6/en/partitioning-limitations.html 5.6.7].}}

3. Using the minimal clock values received in previous step you can prepare a set of SQLs to partition every table.
Examples for adding partitions to already existing tables.

Partitioning a table by day:

<syntaxhighlight lang=sql>
ALTER TABLE `history_uint` PARTITION BY RANGE ( clock)
(PARTITION p2011_10_23 VALUES LESS THAN (UNIX_TIMESTAMP("2011-10-24 00:00:00")) ENGINE = InnoDB,
 PARTITION p2011_10_24 VALUES LESS THAN (UNIX_TIMESTAMP("2011-10-25 00:00:00")) ENGINE = InnoDB,
 PARTITION p2011_10_25 VALUES LESS THAN (UNIX_TIMESTAMP("2011-10-26 00:00:00")) ENGINE = InnoDB,
 PARTITION p2011_10_26 VALUES LESS THAN (UNIX_TIMESTAMP("2011-10-27 00:00:00")) ENGINE = InnoDB,
...
 PARTITION p2011_11_20 VALUES LESS THAN (UNIX_TIMESTAMP("2011-11-21 00:00:00")) ENGINE = InnoDB,
 PARTITION p2011_11_21 VALUES LESS THAN (UNIX_TIMESTAMP("2011-11-22 00:00:00")) ENGINE = InnoDB,
 PARTITION p2011_11_22 VALUES LESS THAN (UNIX_TIMESTAMP("2011-11-23 00:00:00")) ENGINE = InnoDB,
 PARTITION p2011_11_23 VALUES LESS THAN (UNIX_TIMESTAMP("2011-11-24 00:00:00")) ENGINE = InnoDB,
 PARTITION p2011_11_24 VALUES LESS THAN (UNIX_TIMESTAMP("2011-11-25 00:00:00")) ENGINE = InnoDB);
</syntaxhighlight>

Partitioning a table by month:

<syntaxhighlight lang=sql>
ALTER TABLE `trends_uint` PARTITION BY RANGE ( clock)
(PARTITION p2010_10 VALUES LESS THAN (UNIX_TIMESTAMP("2010-11-01 00:00:00")) ENGINE = InnoDB,
 PARTITION p2010_11 VALUES LESS THAN (UNIX_TIMESTAMP("2010-12-01 00:00:00")) ENGINE = InnoDB,
 PARTITION p2010_12 VALUES LESS THAN (UNIX_TIMESTAMP("2011-01-01 00:00:00")) ENGINE = InnoDB,
...
 PARTITION p2011_08 VALUES LESS THAN (UNIX_TIMESTAMP("2011-09-01 00:00:00")) ENGINE = InnoDB,
 PARTITION p2011_09 VALUES LESS THAN (UNIX_TIMESTAMP("2011-10-01 00:00:00")) ENGINE = InnoDB,
 PARTITION p2011_10 VALUES LESS THAN (UNIX_TIMESTAMP("2011-11-01 00:00:00")) ENGINE = InnoDB,
 PARTITION p2011_11 VALUES LESS THAN (UNIX_TIMESTAMP("2011-12-01 00:00:00")) ENGINE = InnoDB,
 PARTITION p2011_12 VALUES LESS THAN (UNIX_TIMESTAMP("2012-01-01 00:00:00")) ENGINE = InnoDB);
</syntaxhighlight>

Note: in examples above some lines are trimmed to keep examples shorter, but you have to specify complete ranges for every partitioned table.

Example commands for manual partition initiating/adding/deletion, just in case:
<syntaxhighlight lang=sql>
ALTER TABLE `history` PARTITION BY RANGE (clock) (PARTITION p2011_11_24 VALUES LESS THAN (UNIX_TIMESTAMP("2011-11-25 00:00:00")));
ALTER TABLE `history` ADD PARTITION p2011_10_23 VALUES LESS THAN (UNIX_TIMESTAMP("2011-10-24 00:00:00")) ENGINE = InnoDB;
ALTER TABLE `history` DROP PARTITION p2011_06;
</syntaxhighlight>
NOTE: for clean/new installations (without collected history) it's enough to repeat the initiating SQL as is (1st in the list) for each required history/trends table. After the script is executed the first time, these old partitions will be dropped and new ones will be created.

=== Partitioning with stored procedures ===

{{note|Before creating procedures or event scheduler rules, read through the whole solution and understand how it works.}}

==== Partitioning settings table ====

This table will hold information on how long the data should be kept for each partitioned table.

<syntaxhighlight lang=sql>
CREATE TABLE `manage_partitions` (
  `tablename` VARCHAR(64) NOT NULL COMMENT 'Table name',
  `period` VARCHAR(64) NOT NULL COMMENT 'Period - daily or monthly',
  `keep_history` INT(3) UNSIGNED NOT NULL DEFAULT '1' COMMENT 'For how many days or months to keep the partitions',
  `last_updated` DATETIME DEFAULT NULL COMMENT 'When a partition was added last time',
  `comments` VARCHAR(128) DEFAULT '1' COMMENT 'Comments',
  PRIMARY KEY (`tablename`)
) ENGINE=INNODB;
</syntaxhighlight>

<syntaxhighlight lang=sql>
INSERT INTO manage_partitions (tablename, period, keep_history, last_updated, comments) VALUES ('history', 'day', 30, now(), '');
INSERT INTO manage_partitions (tablename, period, keep_history, last_updated, comments) VALUES ('history_uint', 'day', 30, now(), '');
INSERT INTO manage_partitions (tablename, period, keep_history, last_updated, comments) VALUES ('history_str', 'day', 120, now(), '');
INSERT INTO manage_partitions (tablename, period, keep_history, last_updated, comments) VALUES ('history_text', 'day', 120, now(), '');
INSERT INTO manage_partitions (tablename, period, keep_history, last_updated, comments) VALUES ('history_log', 'day', 120, now(), '');
INSERT INTO manage_partitions (tablename, period, keep_history, last_updated, comments) VALUES ('trends', 'month', 24, now(), '');
INSERT INTO manage_partitions (tablename, period, keep_history, last_updated, comments) VALUES ('trends_uint', 'month', 24, now(), '');
--
-- STOP here if you partition zabbix database starting from 2.2
-- next queries usually used for zabbix database before 2.2
--
INSERT INTO manage_partitions (tablename, period, keep_history, last_updated, comments) VALUES ('acknowledges', 'month', 6, now(), '');
INSERT INTO manage_partitions (tablename, period, keep_history, last_updated, comments) VALUES ('alerts', 'month', 6, now(), '');
INSERT INTO manage_partitions (tablename, period, keep_history, last_updated, comments) VALUES ('auditlog', 'month', 6, now(), '');
INSERT INTO manage_partitions (tablename, period, keep_history, last_updated, comments) VALUES ('events', 'month', 6, now(), '');
INSERT INTO manage_partitions (tablename, period, keep_history, last_updated, comments) VALUES ('service_alarms', 'month', 6, now(), '');
</syntaxhighlight>

==== Verifying the existence of required partition ====

<syntaxhighlight lang=sql>
DELIMITER $$
 
USE `zabbix`$$
 
DROP PROCEDURE IF EXISTS `create_next_partitions`$$
 
CREATE PROCEDURE `create_next_partitions`(IN_SCHEMANAME VARCHAR(64))
BEGIN
    DECLARE TABLENAME_TMP VARCHAR(64);
    DECLARE PERIOD_TMP VARCHAR(12);
    DECLARE DONE INT DEFAULT 0;
 
    DECLARE get_prt_tables CURSOR FOR
        SELECT `tablename`, `period`
            FROM manage_partitions;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;
 
    OPEN get_prt_tables;
 
    loop_create_part: LOOP
        IF DONE THEN
            LEAVE loop_create_part;
        END IF;
 
        FETCH get_prt_tables INTO TABLENAME_TMP, PERIOD_TMP;
 
        CASE WHEN PERIOD_TMP = 'day' THEN
                    CALL `create_partition_by_day`(IN_SCHEMANAME, TABLENAME_TMP);
             WHEN PERIOD_TMP = 'month' THEN
                    CALL `create_partition_by_month`(IN_SCHEMANAME, TABLENAME_TMP);
             ELSE
            BEGIN
                            ITERATE loop_create_part;
            END;
        END CASE;
 
                UPDATE manage_partitions set last_updated = NOW() WHERE tablename = TABLENAME_TMP;
    END LOOP loop_create_part;
 
    CLOSE get_prt_tables;
END$$
 
DELIMITER ;
</syntaxhighlight>

==== Creating partitions by day ====

<syntaxhighlight lang=sql>
DELIMITER $$
 
USE `zabbix`$$
 
DROP PROCEDURE IF EXISTS `create_partition_by_day`$$
 
CREATE PROCEDURE `create_partition_by_day`(IN_SCHEMANAME VARCHAR(64), IN_TABLENAME VARCHAR(64))
BEGIN
    DECLARE ROWS_CNT INT UNSIGNED;
    DECLARE BEGINTIME TIMESTAMP;
        DECLARE ENDTIME INT UNSIGNED;
        DECLARE PARTITIONNAME VARCHAR(16);
        SET BEGINTIME = DATE(NOW()) + INTERVAL 1 DAY;
        SET PARTITIONNAME = DATE_FORMAT( BEGINTIME, 'p%Y_%m_%d' );
 
        SET ENDTIME = UNIX_TIMESTAMP(BEGINTIME + INTERVAL 1 DAY);
 
        SELECT COUNT(*) INTO ROWS_CNT
                FROM information_schema.partitions
                WHERE table_schema = IN_SCHEMANAME AND table_name = IN_TABLENAME AND partition_name = PARTITIONNAME;
 
    IF ROWS_CNT = 0 THEN
                     SET @SQL = CONCAT( 'ALTER TABLE `', IN_SCHEMANAME, '`.`', IN_TABLENAME, '`',
                                ' ADD PARTITION (PARTITION ', PARTITIONNAME, ' VALUES LESS THAN (', ENDTIME, '));' );
                PREPARE STMT FROM @SQL;
                EXECUTE STMT;
                DEALLOCATE PREPARE STMT;
        ELSE
        SELECT CONCAT("partition `", PARTITIONNAME, "` for table `",IN_SCHEMANAME, ".", IN_TABLENAME, "` already exists") AS result;
        END IF;
END$$
 
DELIMITER ;
</syntaxhighlight>

==== Creating partitions by month ====

<syntaxhighlight lang=sql>
DELIMITER $$

USE `zabbix`$$

DROP PROCEDURE IF EXISTS `create_partition_by_month`$$
 
CREATE PROCEDURE `create_partition_by_month`(IN_SCHEMANAME VARCHAR(64), IN_TABLENAME VARCHAR(64))
BEGIN
    DECLARE ROWS_CNT INT UNSIGNED;
    DECLARE BEGINTIME TIMESTAMP;
        DECLARE ENDTIME INT UNSIGNED;
        DECLARE PARTITIONNAME VARCHAR(16);
        SET BEGINTIME = DATE(NOW() - INTERVAL DAY(NOW()) DAY + INTERVAL 1 DAY + INTERVAL 1 MONTH);
        SET PARTITIONNAME = DATE_FORMAT( BEGINTIME, 'p%Y_%m' );
 
        SET ENDTIME = UNIX_TIMESTAMP(BEGINTIME + INTERVAL 1 MONTH);
 
        SELECT COUNT(*) INTO ROWS_CNT
                FROM information_schema.partitions
                WHERE table_schema = IN_SCHEMANAME AND table_name = IN_TABLENAME AND partition_name = PARTITIONNAME;
 
    IF ROWS_CNT = 0 THEN
                     SET @SQL = CONCAT( 'ALTER TABLE `', IN_SCHEMANAME, '`.`', IN_TABLENAME, '`',
                                ' ADD PARTITION (PARTITION ', PARTITIONNAME, ' VALUES LESS THAN (', ENDTIME, '));' );
                PREPARE STMT FROM @SQL;
                EXECUTE STMT;
                DEALLOCATE PREPARE STMT;
        ELSE
        SELECT CONCAT("partition `", PARTITIONNAME, "` for table `",IN_SCHEMANAME, ".", IN_TABLENAME, "` already exists") AS result;
        END IF;
END$$
 
DELIMITER ;
</syntaxhighlight>

==== Verifying and deleting old partitions ====

<syntaxhighlight lang=sql>
DELIMITER $$
 
USE `zabbix`$$
 
DROP PROCEDURE IF EXISTS `drop_partitions`$$
 
CREATE PROCEDURE `drop_partitions`(IN_SCHEMANAME VARCHAR(64))
BEGIN
    DECLARE TABLENAME_TMP VARCHAR(64);
    DECLARE PARTITIONNAME_TMP VARCHAR(64);
    DECLARE VALUES_LESS_TMP INT;
    DECLARE PERIOD_TMP VARCHAR(12);
    DECLARE KEEP_HISTORY_TMP INT;
    DECLARE KEEP_HISTORY_BEFORE INT;
    DECLARE DONE INT DEFAULT 0;
    DECLARE get_partitions CURSOR FOR
        SELECT p.`table_name`, p.`partition_name`, LTRIM(RTRIM(p.`partition_description`)), mp.`period`, mp.`keep_history`
            FROM information_schema.partitions p
            JOIN manage_partitions mp ON mp.tablename = p.table_name
            WHERE p.table_schema = IN_SCHEMANAME
            ORDER BY p.table_name, p.subpartition_ordinal_position;
 
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;
 
    OPEN get_partitions;
 
    loop_check_prt: LOOP
        IF DONE THEN
            LEAVE loop_check_prt;
        END IF;
 
        FETCH get_partitions INTO TABLENAME_TMP, PARTITIONNAME_TMP, VALUES_LESS_TMP, PERIOD_TMP, KEEP_HISTORY_TMP;
        CASE WHEN PERIOD_TMP = 'day' THEN
                SET KEEP_HISTORY_BEFORE = UNIX_TIMESTAMP(DATE(NOW() - INTERVAL KEEP_HISTORY_TMP DAY));
             WHEN PERIOD_TMP = 'month' THEN
                SET KEEP_HISTORY_BEFORE = UNIX_TIMESTAMP(DATE(NOW() - INTERVAL KEEP_HISTORY_TMP MONTH - INTERVAL DAY(NOW())-1 DAY));
             ELSE
            BEGIN
                ITERATE loop_check_prt;
            END;
        END CASE;
 
        IF KEEP_HISTORY_BEFORE >= VALUES_LESS_TMP THEN
                CALL drop_old_partition(IN_SCHEMANAME, TABLENAME_TMP, PARTITIONNAME_TMP);
        END IF;
        END LOOP loop_check_prt;
 
        CLOSE get_partitions;
END$$
 
DELIMITER ;
</syntaxhighlight>

==== Deleting designated partition ====

<syntaxhighlight lang=sql>
DELIMITER $$
 
USE `zabbix`$$
 
DROP PROCEDURE IF EXISTS `drop_old_partition`$$
 
CREATE PROCEDURE `drop_old_partition`(IN_SCHEMANAME VARCHAR(64), IN_TABLENAME VARCHAR(64), IN_PARTITIONNAME VARCHAR(64))
BEGIN
    DECLARE ROWS_CNT INT UNSIGNED;
 
        SELECT COUNT(*) INTO ROWS_CNT
                FROM information_schema.partitions
                WHERE table_schema = IN_SCHEMANAME AND table_name = IN_TABLENAME AND partition_name = IN_PARTITIONNAME;
 
    IF ROWS_CNT = 1 THEN
                     SET @SQL = CONCAT( 'ALTER TABLE `', IN_SCHEMANAME, '`.`', IN_TABLENAME, '`',
                                ' DROP PARTITION ', IN_PARTITIONNAME, ';' );
                PREPARE STMT FROM @SQL;
                EXECUTE STMT;
                DEALLOCATE PREPARE STMT;
        ELSE
        SELECT CONCAT("partition `", IN_PARTITIONNAME, "` for table `", IN_SCHEMANAME, ".", IN_TABLENAME, "` not exists") AS result;
        END IF;
END$$
 
DELIMITER ;
</syntaxhighlight>

==== Event scheduler ====

<syntaxhighlight lang=sql>
DELIMITER $$
 
USE `zabbix`$$

CREATE EVENT IF NOT EXISTS `e_part_manage`
       ON SCHEDULE EVERY 1 DAY
       STARTS '2011-08-08 04:00:00'
       ON COMPLETION PRESERVE
       ENABLE
       COMMENT 'Creating and dropping partitions'
       DO BEGIN
            CALL zabbix.drop_partitions('zabbix');
            CALL zabbix.create_next_partitions('zabbix');
       END$$
DELIMITER ;
</syntaxhighlight>

See more details about event creation in the [http://dev.mysql.com/doc/refman/5.1/en/create-event.html CREATE EVENT Syntax] section on the MySQL website.

=== Partitioning with an external script ===

An external script can be used instead of stored procedures. While it should be scheduled outside the database, it usually will be simpler and easier to debug. It is suggested to add a daily cron job. The script supports creating new partitions and deleting the old ones.

{{note|Before adding the script as a cron job, the desired storage period and partition type for each table must be set in the script itself. Setting '''keep_history''' to 0 will only keep one, currently active partition.}}

{{note|The script currently assumes usage Zabbix before version 2.2 (without a way to enable specific housekeeper controls). For zabbix starting from 2.2 comment 5 lines as suggested inside the script.}}

{{note|The script currently assumes usage of MySQL version before 5.6. For MySQL version 5.6 and above, 2 lines inside the script should be commented/uncommented.}}

{{note|If using systemd/journald you can see the script output in syslog or by using {{guitext|journalctl -t mysql_zbx_part}}.}}

<syntaxhighlight lang=perl>
#!/usr/bin/perl

use strict;
use Data::Dumper;
use DBI;
use Sys::Syslog qw(:standard :macros);
use DateTime;
use POSIX qw(strftime);

openlog("mysql_zbx_part", "ndelay,pid", LOG_LOCAL0);

my $db_schema = 'zabbix';
my $dsn = 'DBI:mysql:'.$db_schema.':mysql_socket=/var/lib/mysql/mysql.sock';
my $db_user_name = 'zbx_srv';
my $db_password = '<password here>';
my $tables = {	'history' => { 'period' => 'day', 'keep_history' => '30'},
		'history_log' => { 'period' => 'day', 'keep_history' => '30'},
		'history_str' => { 'period' => 'day', 'keep_history' => '30'},
		'history_text' => { 'period' => 'day', 'keep_history' => '30'},
		'history_uint' => { 'period' => 'day', 'keep_history' => '30'},
		'trends' => { 'period' => 'month', 'keep_history' => '2'},
		'trends_uint' => { 'period' => 'month', 'keep_history' => '2'},

# comment next 5 lines if you partition zabbix database starting from 2.2
# they usually used for zabbix database before 2.2

		'acknowledges' => { 'period' => 'month', 'keep_history' => '23'},
		'alerts' => { 'period' => 'month', 'keep_history' => '6'},
		'auditlog' => { 'period' => 'month', 'keep_history' => '24'},
		'events' => { 'period' => 'month', 'keep_history' => '12'},
		'service_alarms' => { 'period' => 'month', 'keep_history' => '6'},
		};
my $amount_partitions = 10;

my $curr_tz = 'Europe/London';

my $part_tables;

my $dbh = DBI->connect($dsn, $db_user_name, $db_password, {'ShowErrorStatement' => 1});

unless ( check_have_partition() ) {
	print "Your installation of MySQL does not support table partitioning.\n";
	syslog(LOG_CRIT, 'Your installation of MySQL does not support table partitioning.');
	exit 1;
}

my $sth = $dbh->prepare(qq{SELECT table_name, partition_name, lower(partition_method) as partition_method,
					rtrim(ltrim(partition_expression)) as partition_expression,
					partition_description, table_rows
				FROM information_schema.partitions
				WHERE partition_name IS NOT NULL AND table_schema = ?});
$sth->execute($db_schema);

while (my $row =  $sth->fetchrow_hashref()) {
	$part_tables->{$row->{'table_name'}}->{$row->{'partition_name'}} = $row;
}

$sth->finish();

foreach my $key (sort keys %{$tables}) {
	unless (defined($part_tables->{$key})) {
		syslog(LOG_ERR, 'Partitioning for "'.$key.'" is not found! The table might be not partitioned.');
		next;
	}

	create_next_partition($key, $part_tables->{$key}, $tables->{$key}->{'period'});
	remove_old_partitions($key, $part_tables->{$key}, $tables->{$key}->{'period'}, $tables->{$key}->{'keep_history'})
}

delete_old_data();

$dbh->disconnect();

sub check_have_partition {
	my $result = 0;
# MySQL 5.5
	my $sth = $dbh->prepare(qq{SELECT variable_value FROM information_schema.global_variables WHERE variable_name = 'have_partitioning'});
# MySQL 5.6
	#my $sth = $dbh->prepare(qq{SELECT plugin_status FROM information_schema.plugins WHERE plugin_name = 'partition'});

	$sth->execute();

	my $row = $sth->fetchrow_array();

	$sth->finish();

# MySQL 5.5
	return 1 if $row eq 'YES';
# MySQL 5.6
	#return 1 if $row eq 'ACTIVE';
}

sub create_next_partition {
	my $table_name = shift;
	my $table_part = shift;
	my $period = shift;

	for (my $curr_part = 0; $curr_part < $amount_partitions; $curr_part++) {
		my $next_name = name_next_part($tables->{$table_name}->{'period'}, $curr_part);
		my $found = 0;

		foreach my $partition (sort keys %{$table_part}) {
			if ($next_name eq $partition) {
				syslog(LOG_INFO, "Next partition for $table_name table has already been created. It is $next_name");
				$found = 1;
			}
		}

		if ( $found == 0 ) {
			syslog(LOG_INFO, "Creating a partition for $table_name table ($next_name)");
			my $query = 'ALTER TABLE '."$db_schema.$table_name".' ADD PARTITION (PARTITION '.$next_name.
						' VALUES less than (UNIX_TIMESTAMP("'.date_next_part($tables->{$table_name}->{'period'}, $curr_part).'") div 1))';
			syslog(LOG_DEBUG, $query);
			$dbh->do($query);
		}
	}
}

sub remove_old_partitions {
	my $table_name = shift;
	my $table_part = shift;
	my $period = shift;
	my $keep_history = shift;

	my $curr_date = DateTime->now;
	$curr_date->set_time_zone( $curr_tz );

	if ( $period eq 'day' ) {
		$curr_date->add(days => -$keep_history);
		$curr_date->add(hours => -$curr_date->strftime('%H'));
		$curr_date->add(minutes => -$curr_date->strftime('%M'));
		$curr_date->add(seconds => -$curr_date->strftime('%S'));
	}
	elsif ( $period eq 'week' ) {
		$curr_date->add(weeks => -$keep_history);
		$curr_date->add(days => -$curr_date->strftime('%d')+1);
		$curr_date->add(hours => -$curr_date->strftime('%H'));
		$curr_date->add(minutes => -$curr_date->strftime('%M'));
		$curr_date->add(seconds => -$curr_date->strftime('%S'));
	}
	elsif ( $period eq 'month' ) {
		$curr_date->add(months => -$keep_history);

		$curr_date->add(days => -$curr_date->strftime('%d')+1);
		$curr_date->add(hours => -$curr_date->strftime('%H'));
		$curr_date->add(minutes => -$curr_date->strftime('%M'));
		$curr_date->add(seconds => -$curr_date->strftime('%S'));
	}

	foreach my $partition (sort keys %{$table_part}) {
		if ($table_part->{$partition}->{'partition_description'} <= $curr_date->epoch) {
			syslog(LOG_INFO, "Removing old $partition partition from $table_name table");

			my $query = "ALTER TABLE $db_schema.$table_name DROP PARTITION $partition";

			syslog(LOG_DEBUG, $query);
			$dbh->do($query);
		}
	}
}

sub name_next_part {
	my $period = shift;
	my $curr_part = shift;

	my $name_template;

	my $curr_date = DateTime->now;
	$curr_date->set_time_zone( $curr_tz );

	if ( $period eq 'day' ) {
		my $curr_date = $curr_date->truncate( to => 'day' );
		$curr_date->add(days => 1 + $curr_part);

		$name_template = $curr_date->strftime('p%Y_%m_%d');
	}
	elsif ($period eq 'week') {
		my $curr_date = $curr_date->truncate( to => 'week' );
		$curr_date->add(days => 7 * $curr_part);

		$name_template = $curr_date->strftime('p%Y_%m_w%W');
	}
	elsif ($period eq 'month') {
		my $curr_date = $curr_date->truncate( to => 'month' );
		$curr_date->add(months => 1 + $curr_part);

		$name_template = $curr_date->strftime('p%Y_%m');
	}

	return $name_template;
}

sub date_next_part {
	my $period = shift;
	my $curr_part = shift;

	my $period_date;

	my $curr_date = DateTime->now;
	$curr_date->set_time_zone( $curr_tz );

	if ( $period eq 'day' ) {
		my $curr_date = $curr_date->truncate( to => 'day' );
		$curr_date->add(days => 2 + $curr_part);
		$period_date = $curr_date->strftime('%Y-%m-%d');
	}
	elsif ($period eq 'week') {
		my $curr_date = $curr_date->truncate( to => 'week' );
		$curr_date->add(days => 7 * $curr_part + 1);
		$period_date = $curr_date->strftime('%Y-%m-%d');
	}
	elsif ($period eq 'month') {
		my $curr_date = $curr_date->truncate( to => 'month' );
		$curr_date->add(months => 2 + $curr_part);

		$period_date = $curr_date->strftime('%Y-%m-%d');
	}

	return $period_date;
}

sub delete_old_data {
	$dbh->do("DELETE FROM sessions WHERE lastaccess < UNIX_TIMESTAMP(NOW() - INTERVAL 1 MONTH)");
	$dbh->do("TRUNCATE housekeeper");
	$dbh->do("DELETE FROM auditlog_details WHERE NOT EXISTS (SELECT NULL FROM auditlog WHERE auditlog.auditid = auditlog_details.auditid)");
}
</syntaxhighlight>

=== Considerations ===

==== Limitations ====

* Make sure the partition ranges do not overlap when creating/adding new partitions, otherwise the effort will return an error.
* A MySQL table is either partitioned completely or not at all. No such records must be left over that do not fit in any of the created partitions.
* When trying to create tables with a large number of partitions you will probably encounter errors like: "Can't create/write to file". To avoid similar situations, enlarge the value of the [http://dev.mysql.com/doc/refman/5.1/en/server-options.html#option_mysqld_open-files-limit open_files_limit] parameter in MySQL configuration file.
* Partitioned tables do not support foreign keys due to [http://dev.mysql.com/doc/refman/5.1/en/partitioning-limitations.html an internal MySQL limitation]. Preparing for partitioning requires foreign keys to be deleted.
* Partitioned tables do not support query cache due to [http://dev.mysql.com/doc/refman/5.1/en/partitioning-limitations.html an internal MySQL limitation].
* All columns, used in the expression for selecting sections of partitioned table, must be part of each unique key the table may have. In other words, each unique key in the table must use each column in the expression for selecting table sections. That is why when getting ready for partitioning primary keys are deleted and ordinary indexes created instead.
* The maximum number of partitions may not exceed 1024 (8192 since MySQL 5.6.7), including subpartitions.
* There are multiple other limitations when using partitioning, the discussion of which would go outside the scope of this article. See more details about these [http://dev.mysql.com/doc/refman/5.1/en/partitioning-limitations.html limitations] on the MySQL website.

==== Recommendations ====

* Use MySQL 5.5 or later. It is has been optimised for and is more stable with partitioned tables.
* Consider using Use XtraDB, rather than pure InnoDB. This backend engine is included in such MySQL forks as MariaDB and Percona.
* TokuDB does not seem to be well suited for the workload Zabbix generates. It does not seem perform so well for selecting from tables (with very large numbers).
Optimise, optimise and optimise again. It ha been a while now since MySQL has become much more than a basic spreadsheet and it now requires a careful fine tuning of configuration parameters.

==== Known issues ====

* When using a wrapper script for partition management, partitions may fail to be created. This occurs when the database is inaccessible at the moment of running the cron script.
* It may be worthwhile to create several further partitions in one go.
* Table ''ids'' can be a bottleneck in Zabbix. It was introduced for object identifier management in distributed monitoring. It's use will be greatly reduced with the removal of node-based distributed monitoring in Zabbix 2.4.

[[Category:Howto|mysql partitioning]]
[[Category:Table partitioning|mysql partitioning]]
[[Category:MySQL|mysql partitioning]]
