+++
title = "How to setup Airflow and Spark within Docker Compose"
date = "2023-04-13T21:23:39+02:00"

description = "A guide on how to setup Airflow with Spark operator and Spark inside Docker Compose."

tags = ["docker", "docker-compose", "airflow", "spark"]
+++

## Abstract
In this guide we'll setup local Airflow and Spark containers within Docker Compose. Making use of Airflow's `DockerOperator` to run Spark jobs.

### Why
There are certain advantages to running Airflow + Spark using docker containers:

- You can share your development with co-worker or other parties with all the dependencies included
- Containers are platform-agnostic, meaning you can build and deploy easily on any cloud provider
- You have the ability to package and deploy multiple applications on the same machine
- You can aggregate the amount of RAM and CPU usage allocated to each service

**Disclaimer** This guide is not intended for production use, rather it's more suitable for local environments or research and development type tasks.

## Overview
In this guide we will use docker compose to orchestrate the docker containers the setup is based on Airflow's official [Running Airflow in Docker](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#running-airflow-in-docker) guide. The full code base can be found in this [GitHub repo](https://github.com/digitamo/docker-airflow-compose)

### Project structure

```
├── dags                    # Airflow DAGs
│   ├── docker_job
│   │   ├── docker-job.py
├── spark                   # Spark jobs
│   ├── app
│   │   ├── Dockerfile
│   │   ├── pi-estimate.py
├── docker-compose.yaml 
```

### Docker Compose

Our `docker-compose.yaml` is a based on [compose file](https://airflow.apache.org/docs/apache-airflow/2.5.3/docker-compose.yaml) from Airflow's [Running Airflow in Docker](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#running-airflow-in-docker) guide

Since we're planning to run Airflow and Spark in isolated containers by using Airflow's `DockerOperator` meaning we will run a docker container from within a docker container. The most straightforward of doing so is by exposing docker daemon via a tcp proxy in `docker-compose.yaml` like so: 
```yaml
# Makes docker daemon accessible from docker containers (mainly docker operators)
# ref: https://onedevblog.com/how-to-fix-a-permission-denied-when-using-dockeroperator-in-airflow/
  docker-proxy:
    image: bobrik/socat
    networks:
      - default_network
    command: "TCP4-LISTEN:2375,fork,reuseaddr UNIX-CONNECT:/var/run/docker.sock"
    ports:
      - "2376:2375"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```
The `docker-compose-yaml` template should look like [docker-compose.yaml](https://github.com/digitamo/docker-airflow-compose/blob/master/docker-compose.yaml)

### Airflow DAG

The `DockerOperator` used in the Airflow DAG would look like as follows
```python
with DAG(
    "estimate_pi_dag",
    default_args=default_args,
    schedule_interval="@daily",
    catchup=False,
) as dag:
    start_dag = DummyOperator(task_id="start_dag")

    spark_job = DockerOperator(
        task_id="docker-estimate-pi",
        image="pi-estimate-task",
        container_name="task___estimate-pi",
        api_version="auto",
        auto_remove=True,
        command="spark-submit ./pi-estimate.py",
        docker_url="tcp://docker-proxy:2375",   # Docker daemon url
        mount_tmp_dir=False,
        environment={"NUM_SAMPLES": 1000},
        network_mode="default_network",
    )

    end_dag = DummyOperator(task_id="end_dag")

    start_dag >> spark_job >> end_dag
```
Notice the `docker_url` value using the same TCP port exposing the docker daemon

### Spark Job

The Spark job is executed in a docker container who's image is define in a `Dockerfile` similar to the [Dockerfile](https://github.com/digitamo/docker-airflow-compose/blob/master/spark/app/Dockerfile) from the example repo in combination with a [Spark job](https://github.com/digitamo/docker-airflow-compose/blob/master/spark/app/pi-estimate.py). The Spark job in this example is a simple job that estimates the value of π (Pi))

Alternatively, you can use a different image as long as it's possible to run the Spark job via `spark-submit`

###s Running locally

1. Set up env vars `echo -e "AIRFLOW_UID=$(id -u)\nAIRFLOW_GID=0" > .env`
2. Build the Spark job
   ```bash
   docker build -f spark/app/Dockerfile -t pi-estimate-task spark/app
   ```
3. Run `docker-compose up airflow-init` which essentially runs `airflow db init`
4. Start the Airflow web service and other components via `docker-compose up`
   4.1. (Optional) you can access the airflow web server by navigating to "localhost:8080" with the default user and password:
      - User: `airflow`
      - Password: `airflow`
5. Once you're done, clean up by running: `docker-compose down --volumes --rmi all`