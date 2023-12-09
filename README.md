# The-Forex-Data-Pipeline-using-Airflow

# Project Overview
This project implements a Forex data pipeline using Apache Airflow, aiming to automate the extraction, transformation, and loading (ETL) of Forex rates data. The pipeline involves various tasks, including checking APIs, downloading rates, storing data in HDFS, processing with Spark, and sending notifications.


# Project Architecture


![1_A4cejebz5-9HqnTu-NLekA](https://github.com/MohamedMagdyyyy/The-Forex-Data-Pipeline-using-Airflow/assets/153362625/99d90c4f-6447-4903-8437-9358b84e106c)

# Implementation Steps

first create a DAG object corresponding to our data pipeline. Then inside this DAG object, we will implement the different tasks that we want to add to the data pipeline, we will specify the dependencies between our tasks in order to say these tasks should be executed first and then the other one.

'''python
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
'''    
