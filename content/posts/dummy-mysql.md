---
title: "Dummy Self-Populating MySQL Table"
date: 2024-03-23
draft: false
---
# Requirements:
  1. Docker
# Steps:
1. Run MySQL server on Docker:
   ```bash
   docker image pull mysql:8.3.0;

   docker run --name mysql-local --detach --publish 3306:3306 --env MYSQL_ALLOW_EMPTY_PASSWORD=true mysql:8.3.0;
   ```
2. Connect to MySQL server:
   ```bash
   mysql --host 127.0.0.1 --port 3306 --username root -p
   ```
3. create table that contains all data types of MySQL:
   ```sql
   -- Create dummy database
   CREATE DATABASE dummy_database;
   USE dummy_database;
   
   -- Create dummy table
   CREATE TABLE IF NOT EXISTS dummy_table (
	id BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
	integer_column BIGINT DEFAULT (FLOOR(100000 + RAND() * 5000000000)),
	fixed_decimal_column DECIMAL(10,9) DEFAULT (RAND() * RAND()),
	approximate_decimal_column DOUBLE DEFAULT (RAND() * RAND()),
	bytes_column BINARY(16) DEFAULT (UNHEX(REPLACE(UUID(), '-', ''))),
	string_column TEXT DEFAULT (SUBSTRING(MD5(RAND()),1,20)),
	spatial_column POINT DEFAULT (POINT(FLOOR(RAND()*(70-10+1))+10, FLOOR(RAND()*(70-10+1))+10)),
	json_column JSON DEFAULT (JSON_ARRAY()),
	created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
	updated_at TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
   );
   ```
4. Create a Stored Procedure that will automatically populates `dummy_database.dummy_table`:
   ```sql
   -- Create Stored Procedure.
   -- more info: https://dev.mysql.com/doc/refman/8.3/en/create-procedure.html
   delimiter //

   -- docs: https://dev.mysql.com/doc/refman/8.3/en/sql-compound-statements.html
   CREATE PROCEDURE IF NOT EXISTS auto_populate_dummy_table()
   BEGIN
	DECLARE maxId INT;
	DECLARE maxCountIdThreshold INT;
	SET maxCountIdThreshold = 100000;
	SELECT MAX(id) FROM dummy_table INTO @maxId;

	IF maxId <= maxCountIdThreshold OR maxId IS NULL THEN 
	  INSERT INTO dummy_database.dummy_table () VALUES ();
	ELSE
	  UPDATE dummy_database.dummy_table SET 
		integer_column = (FLOOR(100000 + RAND() * 5000000000)),
		fixed_decimal_column = (RAND() * RAND()),
		approximate_decimal_column = (RAND() * RAND())
	  WHERE id = FLOOR(RAND()*(maxCountIdThreshold-1+1))+1;
	END IF;
   END //

   delimiter ;

   -- Check created procedure
   SHOW PROCEDURE STATUS WHERE DB = 'dummy_database';
  
   -- Test run
   CALL auto_populate_dummy_table();
   ```
5. Turn on [Event Scheduler](https://dev.mysql.com/doc/refman/8.0/en/event-scheduler.html):
   ```sql
   -- Turn on Event Scheduler
   SET GLOBAL event_scheduler = ON; -- set to OFF to turn off Event Scheduler.

   -- Check Event Scheduler is turned on.
   SHOW PROCESSLIST;
   ```
6. Create scheduled event to run the stored procedure:
   ```sql
   -- Schedule auto_populate procedure.
   CREATE EVENT IF NOT EXISTS scheduled_auto_populate_dummy_table 
	ON SCHEDULE EVERY 30 SECOND 
	ON COMPLETION PRESERVE
	ENABLE
	DO
	  CALL auto_populate_dummy_table();

   -- Check event
   SHOW EVENTS;
   ```
7. Periodically checks `dummy_database.dummy_table` if it's get populated:
   ```sql
   SELECT COUNT(id) FROM dummy_databse.dummy_table;
   ```

