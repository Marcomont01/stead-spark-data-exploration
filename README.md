# stead-spark-data-exploration
Spark based data exploration of the Stanford Earthquake Dataset (STEAD) for DSC 232 Group Project.

##SDSC Expanse Enviroment Setup
- Account: TG-SEF260003
- Partition: shared
- Time limit: 120 minutes
- Number of cores: 8
- Memory per node: 18 GB
- GPUs: 0
- Container image: ~/esolares/spark_py_latest_jupyter_dsc232r.sif
- Module loaded: singularitypro
- Working directory: home

##SparkSession Configuration
- Total cores = 8
- Total memory = 18 GB
- Driver memory reserved = 4 GB

- Executor instances = 8 − 1 = **7**
- Executor memory = (18 − 4) / 7 = 14 / 7 = **2 GB per executor**

from pyspark.sql import SparkSession

spark = (
    SparkSession.builder
    .appName("STEAD Spark Exploration")
    .config("spark.executor.instances", "7")
    .config("spark.executor.memory", "2g")
    .config("spark.driver.memory", "4g")
    .getOrCreate()
)
