# The-Forex-Data-Pipeline-using-Airflow

# Project Overview
This project implements a Forex data pipeline using Apache Airflow, aiming to automate the extraction, transformation, and loading (ETL) of Forex rates data. The pipeline involves various tasks, including checking APIs, downloading rates, storing data in HDFS, processing with Spark, and sending notifications.


# Project Architecture


![1_A4cejebz5-9HqnTu-NLekA](https://github.com/MohamedMagdyyyy/The-Forex-Data-Pipeline-using-Airflow/assets/153362625/99d90c4f-6447-4903-8437-9358b84e106c)

# Implementation Steps





first create a DAG object corresponding to our data pipeline. Then inside this DAG object, we will implement the different tasks that we want to add to the data pipeline, we will specify the dependencies between our tasks in order to say these tasks should be executed first and then the other one.

```python
from airflow import DAG

from datetime import datetime, timedelta

default_args = {
    "owner": "airflow",
    "email_on_failure": False,
    "email_on_retry": False,
    "email": "admin@localhost.com",
    "retries": 1,
    "retry_delay": timedelta(minutes=5)
}

with DAG("forex_data_pipeline", start_date=datetime(2021, 1 ,1), 
    schedule_interval="@daily", default_args=default_args, catchup=False) as dag:
    None
```

# Docker setup

Then i executed the docker-compose file in order to run my services like HIVE, SPARK, HDFS, Airflow
to run all these containers in one network

this is my docker-compose-file

```yaml
version: '2.1'
services:

######################################################
# DATABASE SERVICE
######################################################
  postgres:
    build: './docker/postgres'
    restart: always
    container_name: postgres
    logging:
      driver: "json-file"
      options:
          max-file: "5"
          max-size: "10m"
    ports:
      - "32769:5432"
    #volumes:
      #- ./mnt/postgres:/var/lib/postgresql/data/pgdata
    environment:
      - POSTGRES_USER=airflow
      - POSTGRES_PASSWORD=airflow
      - POSTGRES_DB=airflow_db
      #- PGDATA=/var/lib/postgresql/data/pgdata
    healthcheck:
      test: [ "CMD", "pg_isready", "-q", "-d", "airflow_db", "-U", "airflow" ]
      timeout: 45s
      interval: 10s
      retries: 10

  adminer:
    image: wodby/adminer:latest
    restart: always
    container_name: adminer
    logging:
      driver: "json-file"
      options:
          max-file: "5"
          max-size: "10m"
    ports:
      - "32767:9000"
    environment:
      - ADMINER_DEFAULT_DB_DRIVER=psql
      - ADMINER_DEFAULT_DB_HOST=postgres
      - ADMINER_DEFAULT_DB_NAME=airflow_db
    healthcheck:
      test: [ "CMD", "nc", "-z", "adminer", "9000" ]
      timeout: 45s
      interval: 10s
      retries: 10

######################################################
# HADOOP SERVICES
######################################################
  namenode:
    build: ./docker/hadoop/hadoop-namenode
    restart: always
    container_name: namenode
    logging:
      driver: "json-file"
      options:
          max-file: "5"
          max-size: "10m"
    ports:
      - "32763:9870"
    volumes:
      - ./mnt/hadoop/namenode:/hadoop/dfs/name
    environment:
      - CLUSTER_NAME=hadoop_cluster
    healthcheck:
      test: [ "CMD", "nc", "-z", "namenode", "9870" ]
      timeout: 45s
      interval: 10s
      retries: 10

  datanode:
    build: ./docker/hadoop/hadoop-datanode
    restart: always
    container_name: datanode
    logging:
      driver: "json-file"
      options:
          max-file: "5"
          max-size: "10m"
    depends_on:
      - namenode
    volumes:
      - ./mnt/hadoop/datanode:/hadoop/dfs/data
    environment:
      - SERVICE_PRECONDITION=namenode:9870
    healthcheck:
      test: [ "CMD", "nc", "-z", "datanode", "9864" ]
      timeout: 45s
      interval: 10s
      retries: 10

  hive-metastore:
    build: ./docker/hive/hive-metastore
    restart: always
    container_name: hive-metastore
    logging:
      driver: "json-file"
      options:
          max-file: "5"
          max-size: "10m"
    depends_on:
      - namenode
      - datanode
      - postgres
    environment:
      - SERVICE_PRECONDITION=namenode:9870 datanode:9864 postgres:5432
    ports:
      - "32761:9083"
    healthcheck:
      test: [ "CMD", "nc", "-z", "hive-metastore", "9083" ]
      timeout: 45s
      interval: 10s
      retries: 10

  hive-server:
    build: ./docker/hive/hive-server
    restart: always
    container_name: hive-server
    logging:
      driver: "json-file"
      options:
          max-file: "5"
          max-size: "10m"
    depends_on:
      - hive-metastore
    environment:
      - SERVICE_PRECONDITION=hive-metastore:9083
    ports:
      - "32760:10000"
      - "32759:10002"
    healthcheck:
      test: [ "CMD", "nc", "-z", "hive-server", "10002" ]
      timeout: 45s
      interval: 10s
      retries: 10

  hive-webhcat:
    build: ./docker/hive/hive-webhcat
    restart: always
    container_name: hive-webhcat
    logging:
      driver: "json-file"
      options:
          max-file: "5"
          max-size: "10m"
    depends_on:
      - hive-server
    environment:
      - SERVICE_PRECONDITION=hive-server:10000
    healthcheck:
      test: [ "CMD", "nc", "-z", "hive-webhcat", "50111" ]
      timeout: 45s
      interval: 10s
      retries: 10

  hue:
    build: ./docker/hue
    restart: always
    container_name: hue
    logging:
      driver: "json-file"
      options:
          max-file: "5"
          max-size: "10m"
    depends_on:
      - hive-server
      - postgres
    ports:
      - "32762:8888"
    volumes:
      - ./mnt/hue/hue.ini:/usr/share/hue/desktop/conf/z-hue.ini
    environment:
      - SERVICE_PRECONDITION=hive-server:10000 postgres:5432
    healthcheck:
      test: [ "CMD", "nc", "-z", "hue", "8888" ]
      timeout: 45s
      interval: 10s
      retries: 10

######################################################
# SPARK SERVICES
######################################################

  spark-master:
    build: ./docker/spark/spark-master
    restart: always
    container_name: spark-master
    logging:
      driver: "json-file"

      
      options:
          max-file: "5"
          max-size: "10m"
    ports:
      - "32766:8082"
      - "32765:7077"
    volumes:
      - ./mnt/spark/apps:/opt/spark-apps
      - ./mnt/spark/data:/opt/spark-data
    healthcheck:
      test: [ "CMD", "nc", "-z", "spark-master", "8082" ]
      timeout: 45s
      interval: 10s
      retries: 10

  spark-worker:
    build: ./docker/spark/spark-worker
    logging:
      driver: "json-file"
      options:
          max-file: "5"
          max-size: "10m"
    depends_on:
      - spark-master
    ports:
      - "32764:8081"
    volumes:
      - ./mnt/spark/apps:/opt/spark-apps
      - ./mnt/spark/data:/opt/spark-data
    healthcheck:
      test: [ "CMD", "nc", "-z", "spark-worker", "8081" ]
      timeout: 45s
      interval: 10s
      retries: 10

  livy:
    build: ./docker/livy
    restart: always
    container_name: livy
    logging:
      driver: "json-file"
      options:
          max-file: "5"
          max-size: "10m"
    depends_on:
      - spark-worker
    ports:
      - "32758:8998"
    environment:
      - SPARK_MASTER_ENDPOINT=spark-master
      - SPARK_MASTER_PORT=7077
      - DEPLOY_MODE=client
    healthcheck:
      test: [ "CMD", "nc", "-z", "livy", "8998" ]
      timeout: 45s
      interval: 10s
      retries: 10

######################################################
# AIRFLOW
######################################################

  airflow:
    build: ./docker/airflow
    restart: always
    container_name: airflow
    volumes:
      - ./mnt/airflow/airflow.cfg:/opt/airflow/airflow.cfg
      - ./mnt/airflow/dags:/opt/airflow/dags
    ports:
      - 8080:8080
    healthcheck:
      test: [ "CMD", "nc", "-z", "airflow", "8080" ]
      timeout: 45s
      interval: 10s
      retries: 10

######################################################
# NETWORK
######################################################

# Change name of default network otherwise URI invalid for HIVE
# because of the _ contained by default network
networks:
  default:
    name: airflow-network
```
