# Apache Airflow  

Apache Airflow is a way to programmatically author, schedule, and monitor data pipelines.  

## Overview  

### Components    

Web Server  
Scheduler  
Metadata Database  
Executor  
Worker  

### Key Concepts  

Concept | Explanation
---|---
DAG | A graph object representing your data pipeline
Operator | Describe a single task in your data pipeline
Task | An instance of an operator
TaskInstance | A specific run of a task = DAG + TASK + POINT IN TIME
Workflow | Combination of all the above

In DAG, 
Node: task, Edge: dependency between two tasks.  

Operator: Bash Operator, etc. A task in the DAG.  

### Features  

Pipelines developed in Python.  
Graphical representation of DAGs.  
Scalable.  
Catchup and Backfill.  
Extensible.  
Etc.  


Executor: 
Metastore DB: data related to jobs, DAGs, etc.  

Executors use queueing system. Tasks are added to a queue.  
In cluster mode, queue is external, managed by RabbitMQ/Redis.    

Best practice: Scheduler and web server on same node, metastore external independent node.  

Worker Node != Worker (process)  

### How Airflow Works  

1. The Scheduler reads the DAG folder.  
2. Your DAG is parsed by a process to create a DagRun based on the scheduling parameters of your DAG.  
3. A TaskInstance is instantiated for each Task that needs to be executed and ﬂagged to “Scheduled” in the metadata database.  
4. The Scheduler gets all TaskInstances ﬂagged “Scheduled” from the metadata database, changes the state to “Queued” and sends them to the executors to be executed.  
5. Executors pull out Tasks from the queue ( depending on your execution setup ), change the state from “Queued” to “Running” and Workers start executing the TaskInstances.  
6. When a Task is ﬁnished, the Executor changes the state of that task to its ﬁnal state (success, failed, etc) in the database and the DAGRun is updated by the Scheduler with the state “Success” or “Failed”. Of course, the web server periodically fetch data from the metadatabase to update the UI.

## DAGs  

- Finite directed graph that doesn't have any cycles.  
- In Airflow, it represents a collection of tasks to run, organized in a way that represent their dependencies and relationships.  
- Each **node** is a **Task**.
- Each **edge** is a **Dependency**.

DAGs are in Python files placed in DAG_FOLDER (usually ~/airflow/dags).  
dag_id, description, start_date, schedule_interval, depend_on_past, default_args  

## Operators  

While DAGs describe how to run a workflow, Operators determine what actually gets done.  

- An operator describes a single task in a workflow. 
- Operators are usually (not always) atomic, meaning they can stand on their own, 
and don't need to share resources with other operators.  
- Operators generally run independently, may even run on separate machines.  


- Definition of a single task.  
- Should be idempotent: should produce same result regardless of how many times it is run.  
- Retry automatically in case of failure.  
- Task is created by instantiating an Operator class.  
- An operator defines the nature of the Task and how it should be executed.  
- When an Operator is instantiated, the task becomes a node in the DAG.  

Operators  
- BashOperator  
- PythonOperator  
- EmailOperator  
- MySqlOperator, SqliteOperator, PostgreOperator  

All Operators inherit from BaseOperator.  

#### Types of Operators  

Action Operators  
- Perform an action ( Bash, Python, Email operators).  

Transfer Operators
- Move data from one system to another (between dbs).  
- Source -> machine -> target system  
- Don't use for large amount of data.  

Sensor Operators  
- Waiting for data to arrive at a location.  
- Useful for monitoring external processes.  
- Poke method is called repeatedly until it returns True.  
- Inherit BaseSensorOperator.    

## Scheduler  

The scheduler monitors all tasks and DAGs to ensure execution based on 
the **start_date** and **schedule_interval**.  

execution_date: last time the DAG has been executed.  

Periodically scans DAG folder to inspect tasks and verifies if they can be triggered or not.  

### DagRun  

Instantiation of a DAG in time according to the start_date and the schedule_interval.  

Key Parameters: start_date, end_date, retries, retry_delay, schedule_interval  

#### schedule_interval  

schedule_interval receives a 
- CRON expression as a string or  
- a datetime.timedelta object.  

Alternatively, use a cron "preset":  

Preset | Meaning
---|---
None | Don't schedule. Manually triggered.
@once | Schedule once and only once.
@hourly | Run once an hour at the beginning of the hour.
@daily | Run once a day at midnight.
@weekly | Run once a week at midnight on Sunday morning.
@monthly | Run once a month at midnight of the first day of the month.
@yearly | Run once a year at midnight of Jan 1.

Scheduler runs job soon after the start_date + scheduler interval, 
at the end of the period. 

E.g., scheule_interval = 1 day, start_date = 2016-01-01: runs soon after 2016-01-01 23:59.  

The first DagRun is created based on the minimum start_date of the tasks in the DAG.  


#### Backfill and Catchup  

In case start_date is earlier than actual starting date of the DAG.  
Airflow will run past the DAGs for any intervals that has not been run.  
Allows you to backfill DB with your data as if it was run from the past.  
To avoid this behavior, `catchup=False`.  


#### airflow.cfg parameters  

worker_refresh_interval = 30  
Time took for the webserver to be aware of the new DAGs from the DAG folder.  

dag_dir_list_interval = 300  
How often scheduler has to scan DAG folder for new files.  

#### Tree View  

On the right side, 
- each column: DAG runs,  
- circle: information about the DAG run,  
- square: tasks    


## Executors and Metadata Databases  

Apache Airflow in distributed environment  

![image](./images/airflow_distributed.png)

#### Executor  

A **message queue process** which determines the worker process 
that execute each scheduled task.  

#### Worker  

A process where the task is executed.  


### Sequential Executor with SQLite  

SQLite  
- Non-distributed, ACID-compliant relational database. 
- Supports unlimited number of simultaneous readers, but only one writer.    

#### Sequential Executor  

- Basic executor.  
- Runs one task at a time (sequential).  
- Useful for debugging.  
- Only executor that can be used with SQLite, since SQLite doesn't support 
multiple writers.  
- Default executor.  

#### Configuration file  

```shell script
executor=SequentialExecutor
sql_alchemy_conn=sqlite:////home/airflow/airflow/airflow.db
# If SqlAlchemy should pool database connections
sql_alchemy_pool_enabled=True
broker_url = sqla+mysql://airflow:airflow@localhost:3306/airflow
result_backend = db+mysql://airflow:airflow@localhost:3306/airflow
```

Connection pool: maintains long running connections in memory for efficient re-use & 
provide management for total number of connections an application might use simultaneously.  

SQLite + Sequential Executor: No parallelism/concurrency. Not scalable.  

### Local Executor with PostgreSQL  

#### Concurrency vs. Parallelism  

Concurrency: A condition that exists when at least two threads are making progress. 
A more generalized form of parallelism that can include time-slicing as a form of virtual parallelism.

Parallelism: A condition that arises when at least two threads are executing simultaneously.

[reference](https://medium.com/swift-india/concurrency-parallelism-threads-processes-async-and-sync-related-39fd951bc61d)    


Configuration file for local and celery executors,  

```shell script
# Number of workers the scheduler can run for ALL DAGRuns
parallelism=32
# Number of DAGRuns per dag to allow running at once
max_active_runs_per_dag=16
# Number of TASKs allowed to run per DAGRun at once
dag_concurrency=16
```

![image](./images/concurrency_parallelism.png)

PostgreSQL  
- Object-relational database, ACID compliant.  
- Client-server DB.  
- Concurrent writing and reading.  

#### Local Executor  

Executes tasks locally in parallel. Uses multiprocessing Python library and queues to parallelize 
execution of tasks.  

parallelism == 0: unlimited parallelism. Every task submitted to the Local Executor 
will be executed in its own process.
parallelism > 0: set to number  

```shell script
executor = LocalExecutor
result_backend = db+mysql://airflow:airflow@localhost:3306/airflow
sql_alchemy_conn = postgresql+psycopg2://airflow@localhost:5432/airflow_mdb

# Stop webserver then reset metadata db 
airflow resetdb
```

In UI, admin > connections > create, use [configurations](./config/postgres_connection) for connection.  

In Gantt View, you can see how the tasks are executed, in parallel etc.  

Gantt view of dynamic_dag with dag_concurrency=16    
![image](./images/Gantt_dynamic_dag.png)  

Gantt view of dynamic_dag with dag_concurrency=1  
![image](./images/Gantt_dynamic_dag2.png)  

Data Profiling > Ad Hoc Query, choose connection, run query.  


### Celery Executor with PostgreSQL and RabbitMQ  

- Recommended for production use of Airflow.  
- Distributes the execution of task instances to multiple worker nodes.  
- Uses a queue (RabbitMQ) to order tasks.  
- Producer: scheduler, Broker: RabbitMQ, Message: task, Consumer: Celery worker nodes  

![image](./images/distributed_system.png)  

```shell script
executor = CeleryExecutor
sql_alchemy_conn = postgresql+psycopg2://airflow@localhost:5432/airflow_mdb
broker_url = pyamqp://admin:rabbitmq@localhost/
result_backend = db+postgresql://airflow@localhost:5432/airflow_mdb
worker_log_server_port = 8794

# Stop webserver then initiate metadata db 
airflow resetdb
```

```shell script
# Run a celery worker
airflow worker
```

Parent process: manage running tasks.  
Sending/receiving queue messages, track status, register, kill tasks, etc.  
Spawns N child worker processes that execute tasks.  

Setting number of child worker processes:  
```shell script
airflow worker -c 2
# or in airflow.cfg
worker_concurrency = 2
```
RabbitMQ UI port: 15672  


## Minimizing Repetitive patterns with SubDAGs  

- To create a subDAG, use factory function that returns a DAG object and 
the SubDagOperator to attach the subDAG to the main DAG.  
- The factory function returns an instantiated DAG with the associate tasks. 
The function should be in a different file from where the main DAG is stored.
- SubDags must be scheduled the same as their parent DAG.    

Example of main DAG file: [subdag_dag.py](./dags/subdag_dag.py)  
Example of SubDAG file: [subdag_factory.py](./dags/subdag_factory.py)  

## Interact with External Sources Using Hooks  

#### Hook  

Interface to interact with external systems such as Hive, PostgreSQL, 
Spark, SFTP, etc.  

E.g., PostgreOperator uses PostgreHook to interact with PostgreSQL. It handles 
the connection and allows you to execute SQL like on command line interface.  

`schema` parameter corresponds to the name of database.  

Example DAG: [hook_dag.py](./dags/hook_dag.py)  

## Sharing Data between Tasks with XCOMs  

- Stands for "cross-communication".  
- XCOMs are used to share key-value information between tasks.  
- Consists of key, value, and timestamp.  
- Stored in metadata db with execution_date, task_id, and dag_id.  

XCOMs can be **pushed** (sent) or **pulled** (received).  

Push  
- Can push using `xcom_push()`.  
- If task returns a value (either through operator's execute() method or python function), 
a XCOM containing that value is automatically pushed.   

Pull  
- Tasks receive messanges through `xcom_pull()` based on parameters 
`key`, `task_ids`, and `dag_id`.  

Notes  
- XCOMs can be used to share any serializable object.  
- Some operators such as BashOperator and SimpleHTTPOperator has `xcom_push=False` 
as default.  
- `execution_date` in XCOM means hide key value until this date.  
- When there are two XCOMs with same key, dag id, and task id, the XCOM with 
the most recent `execution_date` will be used.  
- If `execution_date` was not set, same date as `execution_date` of the DagRun.  






 

