# PUBLISH TO SNOWFLAKE FROM DELTA - STREAM

This service provides the ability to publish data from one or more source Delta tables into target Snowflake tables via stage using structured streaming.

For full documentation visit [databricks.com/what-is-structured-streaming](https://www.databricks.com/glossary/what-is-structured-streaming).

## Pre-Requisites
* The source table must be available and setup as NRT streaming
* Only one transformation from a streaming source to target snowflake table is allowed
* Snowflake Stage, Target database, and schemas should exist
* The `update_ts_column` must be present in sql clause

## Configuration
    job_steps:
        - job_step:
            service_name: nrt_snowflake_from_delta
            service_config:
                environments:
                    nonprod:
                        env_suffix: _nonprod
                        sf_target_database: bi_dev
                        sf_stage_database: source_dev
                        trigger:
                            once: true
                    preprod:
                        env_suffix: _preprod
                        sf_target_database: bi_preprod
                        sf_stage_database: source_preprod
                    prod:
                        env_suffix:
                        sf_target_database: bi_prod_donothing
                        sf_stage_database: source
                transformation_step:
                    sf_stage_schema: stage
                    sf_target_schema: client_analytics
                    sf_target_table: entitlement2
                    primary_key: related_entity_id,related_entity_type,version,permission_type,permission_holder_type,
    permission_holder_id
                    exclude_cols: cur_ind
                    update_condition: source.cur_ind = true
                    delete_condition: source.cur_ind = false
                    insert_condition: source.cur_ind = true
                    source_database: refinery_event{env_suffix}
                    source_table: ges_entitlement
                    source_sql:
                        select
                            related_entity_id,
                            related_entity_type,
                            version,
                            permission_type,
                            permission_holder_type,
                            permission_holder_id,
                            cur_ind,
                            insert_ts,
                            update_ts
                        from
                            stream_source_table
!!! note
The following parameter values are required but can be left empty:  
`update_condition`  
`delete_condition`  
`insert_condition`
!!! note
The following parameter values are optional but cannot be left empty:  
`exlude_cols`  
`merge_sql`  
`partition_columns`
## Schedule
For lenient latency requirements use the optional **trigger** parameter and set the expression `once: true`. The trigger option is situated in the environment to allow non-prod work streams to be scheduled intraday for cost saving.

## Full Publish Support
    job_steps:
        - job_step:
            service_name: nrt_snowflake_from_delta
            service_config:
                environments:
                    nonprod:
                        env_suffix: _nonprod
                        sf_target_database: ods_dev
                    preprod:
                        env_suffix: _preprod
                        sf_target_database: ods_preprod
                    prod:
                        env_suffix:
                        sf_target_database: ods_production
                transformation_step:
                    sf_target_schema: event
                    sf_target_table: genre
                    overwrite_target: true
                    source_sql:
                        select
                            genre_id,
                            segment_id,
                            genre_nm,
                            insert_ts,
                            update_ts,
                            last_update_dttm
                        from
                            default.discovery_genre
!!! note
1. `update_ts` column will be used for deduping
2. The sql statement should use/reference the temporary table `stream_source_table`
3. Parameter values for `update_condition`, `delete_condition`, `insert_condition` should use alias target source to create the filter `expression: target`. Example:  
`<column_name><any_comparision_operators><source_column_name> | <literal value>`
4. Use full publish support for once a day whole table refresh requirement

## Service Configuration
1. Clone to you home folder the development/testing notebook: `workspace/cds_nonprod/stream_snowflake_from_Delta_Develop`
2. Update the configuration to meet the requirement
3. Test the configuration
4. QA target table
5. Checkin the configuration to GIT
6. Merge the changes to develop the branch
