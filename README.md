# mysql_sys.x-processlist

SELECT 
    `pps`.`THREAD_ID` AS `thd_id`,
    `pps`.`PROCESSLIST_ID` AS `conn_id`,
    IF((`pps`.`NAME` IN ('thread/sql/one_connection' , 'thread/thread_pool/tp_one_connection')),
        CONCAT(`pps`.`PROCESSLIST_USER`,
                '@',
                CONVERT( `pps`.`PROCESSLIST_HOST` USING UTF8MB4)),
        REPLACE(`pps`.`NAME`, 'thread/', '')) AS `user`,
    `pps`.`PROCESSLIST_DB` AS `db`,
    `pps`.`PROCESSLIST_COMMAND` AS `command`,
    `pps`.`PROCESSLIST_STATE` AS `state`,
    `pps`.`PROCESSLIST_TIME` AS `time`,
    `pps`.`PROCESSLIST_INFO` AS `current_statement`,
    IF((`esc`.`END_EVENT_ID` IS NULL),
        `esc`.`TIMER_WAIT`,
        NULL) AS `statement_latency`,
    IF((`esc`.`END_EVENT_ID` IS NULL),
        ROUND((100 * (`estc`.`WORK_COMPLETED` / `estc`.`WORK_ESTIMATED`)),
                2),
        NULL) AS `progress`,
    `esc`.`LOCK_TIME` AS `lock_latency`,
    `esc`.`ROWS_EXAMINED` AS `rows_examined`,
    `esc`.`ROWS_SENT` AS `rows_sent`,
    `esc`.`ROWS_AFFECTED` AS `rows_affected`,
    `esc`.`CREATED_TMP_TABLES` AS `tmp_tables`,
    `esc`.`CREATED_TMP_DISK_TABLES` AS `tmp_disk_tables`,
    IF(((`esc`.`NO_GOOD_INDEX_USED` > 0)
            OR (`esc`.`NO_INDEX_USED` > 0)),
        'YES',
        'NO') AS `full_scan`,
    IF((`esc`.`END_EVENT_ID` IS NOT NULL),
        `esc`.`SQL_TEXT`,
        NULL) AS `last_statement`,
    IF((`esc`.`END_EVENT_ID` IS NOT NULL),
        `esc`.`TIMER_WAIT`,
        NULL) AS `last_statement_latency`,
    `sys`.`mem`.`current_allocated` AS `current_memory`,
    `ewc`.`EVENT_NAME` AS `last_wait`,
    IF(((`ewc`.`END_EVENT_ID` IS NULL)
            AND (`ewc`.`EVENT_NAME` IS NOT NULL)),
        'Still Waiting',
        `ewc`.`TIMER_WAIT`) AS `last_wait_latency`,
    `ewc`.`SOURCE` AS `source`,
    `etc`.`TIMER_WAIT` AS `trx_latency`,
    `etc`.`STATE` AS `trx_state`,
    `etc`.`AUTOCOMMIT` AS `trx_autocommit`,
    `conattr_pid`.`ATTR_VALUE` AS `pid`,
    `conattr_progname`.`ATTR_VALUE` AS `program_name`
FROM
    (((((((`performance_schema`.`threads` `pps`
    LEFT JOIN `performance_schema`.`events_waits_current` `ewc` ON ((`pps`.`THREAD_ID` = `ewc`.`THREAD_ID`)))
    LEFT JOIN `performance_schema`.`events_stages_current` `estc` ON ((`pps`.`THREAD_ID` = `estc`.`THREAD_ID`)))
    LEFT JOIN `performance_schema`.`events_statements_current` `esc` ON ((`pps`.`THREAD_ID` = `esc`.`THREAD_ID`)))
    LEFT JOIN `performance_schema`.`events_transactions_current` `etc` ON ((`pps`.`THREAD_ID` = `etc`.`THREAD_ID`)))
    LEFT JOIN `sys`.`x$memory_by_thread_by_current_bytes` `mem` ON ((`pps`.`THREAD_ID` = `sys`.`mem`.`thread_id`)))
    LEFT JOIN `performance_schema`.`session_connect_attrs` `conattr_pid` ON (((`conattr_pid`.`PROCESSLIST_ID` = `pps`.`PROCESSLIST_ID`)
        AND (`conattr_pid`.`ATTR_NAME` = '_pid'))))
    LEFT JOIN `performance_schema`.`session_connect_attrs` `conattr_progname` ON (((`conattr_progname`.`PROCESSLIST_ID` = `pps`.`PROCESSLIST_ID`)
        AND (`conattr_progname`.`ATTR_NAME` = 'program_name'))))
ORDER BY `pps`.`PROCESSLIST_TIME` DESC , `last_wait_latency` DESC
