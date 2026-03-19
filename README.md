# Introduction
This group sought out seismic data with the intent to predict earthquakes with high accuracy. Living in California, we understand the destruction that undetected earthquakes can cause. Earthquakes can strike anytime and with varying intensity, so developing a model that can make accurate predictions could potentially save countless lives.

This dataset is 100GB, meaning that we require concepts of big data and distributed computing. With this amount of data, we could not feasibly do it on a single computer. At this size, even with distributed computing, we needed to make decisions based on feature engineering and efficiency in performance, as an inefficient approach would lead to substantial waste of computation resources and time.

### Fundamentals 
We are operating with the following resources.

##### Dataset
Stanford Earthquake Dataset (STEAD):  
https://www.kaggle.com/datasets/isevilla/stanford-earthquake-dataset-stead

##### Environment
All work is performed on SDSC Expanse using Apache Spark.

##### Notebook
The data exploration notebook will be located in the `notebooks/` directory.
##### SDSC Expanse Enviroment Setup
- Account: TG-SEF260003
- Partition: shared
- Time limit: 120 minutes
- Number of cores: 16
- Memory per node: 150 GB
- GPUs: 0
- Container image: ~/esolares/spark_py_latest_jupyter_dsc232r.sif
- Module loaded: singularitypro
- Working directory: home

##### SparkSession Configuration
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
```
spark = (
    SparkSession.builder
    .appName("STEAD Spark Exploration")
    .config("spark.executor.instances", "7")
    .config("spark.executor.memory", "2g")
    .config("spark.driver.memory", "4g")
    .getOrCreate()
)
```
---

# Methods

Our notebook is linked here: [Code](Notebook.ipynb)

## Data Exploration

The data contained 1,268,314 rows and 35 columns, including both station metadata and signal-derived variables. Summary statistics were computed for numerical features, and missingness was analyzed across all columns. See an example of our look in the unique values of a column [here](example_exploring.png). We confirmed there were no duplicate rows.

Geographic visualization of receiver locations was performed using latitude and longitude, revealing clusters of stations in regions with active seismic monitoring. See the image [here](earthquake_locations.png).

## Preprocessing (using Spark)

Missingness analysis showed that certain signal variables were partially missing, while some columns were entirely null and removed during preprocessing. Station locations were globally distributed but concentrated in well-monitored regions.

Data cleaning steps included:

- Removing rows with missing `trace_category` values  
- Filtering out rows with inconsistent `p_status` values
    - This gives 
- Dropping fully null columns: `snr_db` and `coda_end_sample`
- Filling missing signal-related values (`p_arrival_sample`, `p_travel_sec`, `s_arrival_sample`, `s_weight`) with 0
- Many numerical columns are cast to double.

Missingness analysis showed that certain signal variables were partially missing, while some columns were entirely null and removed during preprocessing. Station locations were globally distributed but concentrated in well-monitored regions.

After filtering, we lose 5314 rows and now have 1263000 remaining.

Class distribution was:

- earthquake: 1,027,574  
- noise: 235,426  

All modeling features were free of missing values after preprocessing.


Categorical features (`receiver_type`, `network_code`) were encoded using `StringIndexer` and `OneHotEncoder`. Numerical features were standardized using `StandardScaler`.

Feature vectors were constructed using `VectorAssembler`, combining station features and signal features into a single representation.

## Model 1 (Distributed Random Forest)

The first model was a Random Forest classifier. We initially constructed a feature set consisting of station-based variables:

- receiver_latitude  
- receiver_longitude  
- receiver_elevation_m  
- receiver_type  
- network_code  

The model was trained using:

- Number of trees: 100  
- Maximum depth: 10  

A comparison model was also trained with:

- Number of trees: 50  
- Maximum depth: 5  

Data was split into training, validation, and test sets using a 60/20/20 random split.

## Model 2 (PCA + Random Forest)

The second model incorporated dimensionality reduction using Principal Component Analysis (PCA). The feature set was expanded to include signal-based variables:

- p_arrival_sample  
- p_travel_sec  
- s_arrival_sample  
- s_weight  

After scaling, PCA was tested on our validation set with values of k ranging from 1 to 9, meaning we would get a model with K principal components. For each value of k, a Random Forest classifier was trained using the same hyperparameters as Model 1.

---

# Results

## Model 1

The Random Forest model achieved:

- Train Accuracy: 0.9055  
- Validation Accuracy: 0.9056  
- Test Accuracy: 0.9054  
- Train F1 Score: 0.8926  
- Validation F1 Score: 0.8926  
- Test F1 Score: 0.8924  
- Train AUC: 0.9148  
- Validation AUC: 0.9154  
- Test AUC: 0.9150  

The shallower model (50 trees, depth 5) produced lower performance across all metrics, with test AUC around 0.9002. Full results for the shallow tree are [here](shallow_model_results.png).

[Here is a visual](baseline_comparison.png) comparison between all of the baseline AUC scores

## Model 2

The PCA-based Random Forest achieved near-perfect performance for most values of k. 

The test AUC scores are as follows:
- k = 3: 0.9999990548  

- k = 2: 0.9999106777  

- k = 1: 0.9431  

Performance stabilized for k=3, with AUC values approaching 1.0. The explained variance plot indicated that variance plateaued around 3–4 principal components. Using k=3, [here is a confusion matrix](final_confusion_matrix.png) showing exact classifications with our final model.

---

# Discussion

The baseline Random Forest model (Model 1) produced consistent and believable results. Training, validation, and test performance were nearly identical, suggesting that the model generalized well and did not exhibit significant overfitting. The improvement from the shallower model to the deeper model indicates that increased model complexity was beneficial for this task.

The PCA-based model (Model 2) achieved extremely high performance, with near-perfect AUC values across most values of k. While this suggests that the data is highly separable, such results are unusual and should be interpreted cautiously.

Several factors may contribute to this behavior:

- Signal-based features may strongly encode earthquake characteristics  
- Filling missing values with 0 may introduce artificial separation between classes  
- Random data splitting may lead to data leakage if temporal or spatial dependencies exist  

Although PCA revealed that most predictive information is contained within a low-dimensional space, this does not guarantee that the model is learning patterns that will generalize to real-world scenarios.

---

# Conclusion

This project demonstrated how distributed machine learning can be applied to large-scale seismic data using Spark. The workflow required careful consideration of preprocessing, feature engineering, and scalability, highlighting the importance of designing pipelines that operate efficiently on large datasets.

The baseline model provided strong and reliable performance, while the PCA-based model achieved near-perfect results. However, these results should be interpreted with caution due to the possibility of data leakage and overly informative features.

With more time and resources, future work would focus on:

- Using time-based splits instead of random splits  
- Conducting more rigorous leakage checks  
- Exploring alternative models such as gradient boosting  
- Evaluating the model’s ability to make predictions further in advance  

Overall, this project highlights both the power and limitations of machine learning in a big data setting. While distributed tools enable large-scale analysis, careful validation is essential to ensure that strong performance reflects meaningful and generalizable insights.

--- 

# Statement of Collaboration
- David Moncivais: Coder/Team Leader: Coder during milestone 2 (created the word graph and described the dataset), fine tuning during milestone 3, and coder and writer for milestone 4.
- Mitchell Farrington: Data Explorer/Writer: Found dataset to work with and did intro analysis for Milestone 1, created the Preprocessing plan (the focus on missingness) for Milestone 2, Planner and evaluator for milestone 3 (based on results, I helped create the plan for how to proceed given the nature of the dataset), and wrote the README, fitting analysis, and conclusion for Milestone 4.
- Marco Curiel: Coder: Created the github, performed exploratory analysis for milestone 2, created the baseline model for Milestone 3, made visuals and corrected code for Milestone 4.
