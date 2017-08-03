# Airflow Dockerfile

This repository contains **Dockerfile** of [airflow](https://github.com/airbnb/airflow) for [Docker](https://www.docker.com/)'s automated build.

### Information

* Based on Debian Jessie official Image [debian:jessie](https://registry.hub.docker.com/_/debian/)
* Install [Docker](https://www.docker.com/)
* Install [Docker Compose](https://docs.docker.com/compose/install/)
* Following the Airflow release from [Python Package Index](https://pypi.python.org/pypi/airflow)


### Build

For example, if you need to install [Extra Packages](http://pythonhosted.org/airflow/installation.html#extra-package), edit the Dockerfile and than build-it.
      ```
        docker build --rm -t motiv/docker-airflow .
      ```

## Usage

Start the stack (postgresql, redis, airflow-webserver, airflow-scheduler airflow-flower & airflow-worker) :
      ```
        ./script/docker_compose_run.sh
      ```


It does `docker-compose build` followed by `docker-compose up -d`, and then
starts monitoring airflow webserver and celery flower.


### UI Links

- Airflow: [localhost:8080](http://localhost:8080/)
- Flower: [localhost:5555](http://localhost:5555/)

(with boot2docker, use: open http://$(boot2docker ip):8080)


________________________________________________________________________________



# Airflow Nomenclature
http://airflow.apache.org/concepts.html

  * A DAG, or 'directed acyclic graph', specifies work to be done in a particular order at a given time or interval

  * Airflow DAGs are composed of Tasks

  * Each Task is created by instantiating an Operator class.
    * More specifically, a configured instance of an Operator becomes a Task, i.e. `task_1 = MyOperator(...)`

  * When a DAG is started, Airflow creates a DAG Run entry in the meta-database

  * When a Task is executed during a DAG Run, a Task Instance is created

  * `AIRFLOW_HOME` is the directory where you store your DAG definition files and Airflow plugins


________________________________________________________________________________


# Airflow Layout

`dags` houses all of the Directed Acyclic Graphs (DAGs) that have been defined and will be scheduled and run.
  A DAG specifies the workflow for a set of tasks. In these files, the schedule, operators, order/dependency are defined.
  The `rawsync_dag` will use a PostgresOperator to connect to the Redshift and run the SQL query.

`airflow.cfg` defines all the specifics for Airflow - everything from which executor to use (Sequential, Local, or Celery), to the Postgres meta-database connection, and the message broker connection and everything else is specified here.

`logs` the logs files are not tracked with git. This folder contains logs for DAG runs (DAGs that have been initalized and run) as well as the tasks (Operators that have been initialized and run) in separate folders and files by date.


________________________________________________________________________________


# Writing a DAG file

  1. Create dag file in `dags` folder

  2. Import necessary modules
    ```
    from airflow import DAG
    from airflow.operators.bash_operator import BashOperator
    from datetime import datetime, timedelta
    ```

  3. Define default arguments
    ```
    default_args = {
        'owner': 'airflow',
        'depends_on_past': False,
        'start_date': datetime(2015, 6, 1),
        'email': ['airflow@airflow.com'],
        'email_on_failure': False,
        'email_on_retry': False,
        'retries': 1,
        'retry_delay': timedelta(minutes=5),
        # 'queue': 'bash_queue',
        # 'pool': 'backfill',
        # 'priority_weight': 10,
        # 'end_date': datetime(2016, 1, 1),
    }
    ```

  4. Instantiate the DAG
    ```
    dag = DAG('tutorial', default_args=default_args, schedule_interval=timedelta(1))
    ```

  5. Create tasks for the DAG
    ```
    t1 = BashOperator(
      task_id='print_date',
      bash_command='date',
      dag=dag)

    t2 = BashOperator(
      task_id='sleep',
      bash_command='sleep 5',
      retries=3,
      dag=dag)

    templated_command = """
      {% for i in range(5) %}
          echo "{{ ds }}"
          echo "{{ macros.ds_add(ds, 7)}}"
          echo "{{ params.my_param }}"
      {% endfor %}
    """

    t3 = BashOperator(
      task_id='templated',
      bash_command=templated_command,
      params={'my_param': 'Parameter I passed in'},
      dag=dag)
    ```

  6. Setting up dependencies between tasks
    ```
    t2.set_upstream(t1)
    t3.set_upstream(t1)
    ```
    OR
    ```
    t1 >> t2
    t1 >> t3
    ```


________________________________________________________________________________


# Airflow Command Line
http://airflow.apache.org/cli.html

### Airflow Components
`airflow webserver` starts the webserver that powers the UI (usually port 8080)

`airflow scheduler` starts the scheduler (and worker depending on the Executor that was selected)

`airflow worker` starts a worker that can be used with the CeleryExecutor

`airflow flower` starts a server to show the status of any workers (usually port 5555)


### Running DAG Script
  `python ~/airflow/dags/tutorial.py`


### Testing
  Running a DAG's tasks on a specific date
  Command template: `airflow test dag_id task_id date`

  `airflow test tutorial print_date 2015-06-01` # testing `print_date` task

  `airflow test tutorial sleep 2015-06-01` # testing `sleep` task

##### Note: `airflow test` command runs task instances locally, outputs their log to stdout (on screen), doesn’t bother with dependencies, and doesn’t communicate state (running, success, failed, ...) to the database.


### Backfill
  Sample command `airflow backfill tutorial -s 2015-06-01 -e 2015-06-07`

  This will run a DAG at the correct intervals for the dates specified, respect task order/dependency, emit logs, and record status in the meta-database


________________________________________________________________________________


# Airflow Web UI
http://airflow.apache.org/ui.html

  1. DAGs tab
    * Turn DAG `On` or `Off`
    * DAG name
    * Schedule - specifies the when the DAG is scheduled to run
    * Owner
    * Counts for recent tasks (success, running, failed, upstream_failed, up_for_retry, queued)
    * Lastest run datetime
    * Status of all previous DAG runs
      * Links to graph View
        * Click on a task to see menu for
          * Task instance details
          * Rendered
          * Task Instances
          * View logs
          * Etc.
    * Links
      * "Play icon" - manually trigger a DAG run
      * "Tree icon" - Tree View shows dependencies between tasks, operators used, and individual statuses of tasks within a DAG
      * "Star icon" - Graph View shows a more explicit view of all tasks and their dependencies/order for a given DAG run
      * "Vertical bar chart icon" - Task Duration shows task duration over time
      * "Papers icon" - Task Tries shows number of attempts for a task over time
      * "Plane icon" - Landing Times shows
      * "Horizontal bar chart icon" - Gantt Chart shows dependencies of tasks for a DAG run as well as start and times
      * "Lightning icon" - shows code for DAG
      * "Horizontal bars" - shows logs overview for DAG
      * "Refresh icon" - refreshes that DAG
  2. Data Profiling
    * Ad Hoc Query
      * Interface for querying database connections
    * Charts
    * Known Events
  3. Browse
    * SLA Misses
    * Task Instances
      * View all tasks that have been instantiated in a DAG run
    * Logs
      * View all logs
    * Jobs
      * View all jobs and details
        * State, Job Type, Start Date, End Date, Latest Heartbeat, Executor Class, Hostname, Unixname
    * DAG Runs
      * Create or view all DAG Runs
  4. Admin
    * Pools
      * Create or view pools
    * Configuration
      * Airflow configuration file (default is for this to be hidden)
    * Users
      * Create or view users
      * Users credentials can be added for web auth
    * Connections
      * Create or view database connections
        * Example: connection to Postgres referenced in PostgresOperator
    * Variables
      * Create or view variables stored in Airflow
    * XComs
      * Xcom details: key, value, timestamp, execution date, task id, dag id
  5. Docs
    * Documentation
    * Github
  6. About
    * Version


________________________________________________________________________________


# Notes

  * Airflow can even be stopped entirely and running workflows will resume by restarting the last unfinished task.

### Connections
  * Airflow needs to know how to connect to your environment. Information such as hostname, port, login and passwords to other systems and services is handled in the Admin->Connection section of the UI. The pipeline code you will author will reference the ‘conn_id’ of the Connection objects.

### Workers
  * The worker needs to have access to its DAGS_FOLDER, and you need to synchronize the filesystems by your own means.
    A common setup would be to store your DAGS_FOLDER in a Git repository and sync it across machines using Chef, Puppet, Ansible, or whatever you use to configure machines in your environment.
    If all your boxes have a common mount point, having your pipelines files shared there should work as well


________________________________________________________________________________


# Links

### Writing DAGs
  * http://airflow.apache.org/tutorial.html#
  * http://michal.karzynski.pl/blog/2017/03/19/developing-workflows-with-apache-airflow/


### Airflow Architecture
  * https://stlong0521.github.io/20161023%20-%20Airflow.html
  * http://www.clairvoyantsoft.com/assets/whitepapers/GuideToApacheAirflow.pdf


### Airflow ETL Example
  * https://gtoonstra.github.io/etl-with-airflow/
