# stead-spark-data-exploration
Spark based data exploration of the Stanford Earthquake Dataset (STEAD) for DSC 232 Group Project.

## Dataset
Stanford Earthquake Dataset (STEAD):  
https://www.kaggle.com/datasets/stanford-earthquake-dataset/stead

## Environment
All work is performed on SDSC Expanse using Apache Spark.

## Notebook
The data exploration notebook will be located in the `notebooks/` directory.
## SDSC Expanse Enviroment Setup
- Account: TG-SEF260003
- Partition: shared
- Time limit: 120 minutes
- Number of cores: 8
- Memory per node: 18 GB
- GPUs: 0
- Container image: ~/esolares/spark_py_latest_jupyter_dsc232r.sif
- Module loaded: singularitypro
- Working directory: home

## SparkSession Configuration
- Total cores (CPU) = 8
- Total memory = 18 GB
  We set aside some memory  for the Spark driver since it handles coordination, scheduling, and job management.

- Driver memory reserved = 4 GB
  
This leaves 14 GB available memory for executors: 18 - 4 = **14GB**

- Executor instances = 8 − 1 = **7**
- Executor memory = (18 − 4) / 7 = 14 / 7 = **2 GB per executor**

We left 1 core for the driver to avoid resource contention and keep scheduling smooth.
Leaving 4GB to the driver gives it enough memory to manage tasks and metadata without running into memory issues.

The remaining memory is evenly divided by the 7 executors so that Spark can load and process data in parralel.
  

from pyspark.sql import SparkSession

spark = (
    SparkSession.builder
    .appName("STEAD Spark Exploration")
    .config("spark.executor.instances", "7")
    .config("spark.executor.memory", "2g")
    .config("spark.driver.memory", "4g")
    .getOrCreate()
)
