# turbine-homework
## Classificiation task<br>
1. The custom block I added to the TF training is called FCN block, which combines an FCN layer with a layernorm and relu layer. Layer norm is used to normalize the feature values.<br>
3. for task 3 I chose an ensemble model called catboost (gradient boosted ensemble model), which is an improved version of XGBoost. I chose this model because these types of models most often outperform DL applications on tabular data (kaggle competitions), and can work with less data better than DL ones. It is also really easy to train, compatible with sklearn library and has built in hyperparam search.<br>
4. I wrote a simple "automated" script that searches for eval published by TF and CatBoost, and compares them across every metric key included in sklearn's classification_report class. The script chooses a candidate model based on how many times was model_x better at metric_y, (x, y - 1 ... n). If I had to look at the results manually, I would first identify a metric that is cruical to the business problem at hand and then evaluate based on that. In this case in my opinion correctly classifying the malignant cases is more important than classifying benign ones, since an unnoticed cancer could lead to death, whereas the other way around would be less lethal / could turn out easily that it is false on the next check up. Therefore, it would be suitable to look at recall, sensitivity, and f1-score. Recall, because (tp / tp + fn), where fn means that we classified a malignant case as benign, and the goal would be to minimize these cases. For sensitivity (precision for the benign's point of view), because (tn / tn + fn). F1 is just for balance reasons (from the malignant case's point of view). 
<br>
Usually the preferred way to do training properly would be to split the original dataset into three parts (train, validation, test). But in this case due to the size of the dataset I decided to only use train and test (where I use the test set basically as validation). <br>

## MLOps arhictecture
__First proposal__

![diagrammlops](https://user-images.githubusercontent.com/57996039/220168455-bc7d9ce6-e8b5-4c77-b1d3-aa015df4fcbb.png)
<br>
I chose to split the responsibility of the users into two groups:
- the first one is the training account, that has access to all training related tools, training databases, file storages etc. 
- The second account is used for prediction related task. Since we are planning to do offline batch inference we don't need API services, we only need to be able to run our inference script job somewhere, get access to the eval data and save the results / evaluation somewhere.
Splitting the workflow into multiple accounts usually is not ideal for a startup, since the user group would have to deal with a lot of tasks related to MLOps, but it seemed ideal to make an additional distinction in this case.
<br>
Training account architecture:
On a high level, to run scalable training pipelines we need a database / file storage to load the prepared data from, have a workspace either on cloud or locally where the code development is happening, a scheduler that is able to automatically scale the capacity based on workload and schedule multiple jobs parallel and then some place where we can store model artifacts / training results, additionally we can place a monitoring service somewhere in the architecture so that we can detect covariate shift / other shifts, compare models etc. For this POC I created a simple architecture workflow on AWS with AWS tools.<br>
Since one of the constraints was that for this usecase we need to use large datasets, storing datasets on S3 would not be ideal, since the read time / versioning might introduce additional complexity. Therefore, depending on the usecase we could use PostgresSQL / NoSQL DB (based on dataset size / do we need relationship abstractions) or amazon feature store if we want to use classical ML models with pre-computed features on scale. Versioning might be easy with an additional column to keep track of changes, however if the data would change often it might be better to use a AWS file system (FSx or S3) with DVC. But  for this usecase let's assume that it doesn't change often, and we can easily keep track and store metadata regarding these properties.<br>
For the development workspace, we have 2 options. We either develop with SageMaker studio notebooks, and then push our environment / code as a container to ECR or we develop locally and then push that containerized to ECR.<br>
For the job orchestration / scheduling we can use AWS Batch, where we need to define our compute group (ECS). We push our pipelines as jobs, and then autoscale the compute group horizontially if the workload needs it. I chose AWS Batch as it had autoscale capability, job scheduling, and compute resource management options. It would have been easier to use AWS SageMaker pipelines here, but I will talk about that in the second proposal.<br>
Regarding one of the constraints that the task had, was that we wanted to minimize the training time for each team. The task said that there are 4 teams that wants to run the same training pipeline 2 times a week. One training pipeline lasts run lasts for 100 hours on a single cpu and that one gpu can accelerate it 6-7x. To be concious lets go forward with the worst case scenario where it only speeds it up by 6x, then one training pipeline would last about ~16 - 17 hours. <br>
The biggest VM that can be provisioned has 8 GPU cores, therefore if we assume that we have a perfect parallel / distributed training and the number of cores scales the runtime lineraly, one training pipeline run would last about 100 / (6 * 8) = ~ 2 hour, which would be close to the required 1 hour. Additionally, the max parallel runs are capped at 7, so one training pipeline job would be queud. The proposed time table to run it for every team would be the following:
We provision 7 instances with 8 GPU-s to each team and queue one. Each run would finish around in 2 hours and 3 teams would be done for the week, and then after the first one finishes, we run the job on that instance and then it the second run for team 4 would be finished in 2 hours so only one team would have to wait a total of 4 hours for the 2 pipelines to finish and they would be done in a day. If we could do multi-node parallel jobs, we could speed up this process even more. In that case we would need to provision 4 8 gpu VMs for each team, but not at the same time. The first team schedules their pipelines each on two nodes and each would be finished in 1 hour. Then the rest of the teams scheduled jobs would run on the free instances, and we could finish in 4 hours using 4 VM's only for each team.<br>
After the pipeline is finished, assume that we have a publish step, where we save the results to S3, with the model artifacts (at each configured checkpoint). We can track this with pre/post fixes or with DVC. Additionally, we could monitor the results, and watch drifts and then trigger events based on that but it is not needed in this case. I added one optional step, where we use cloudwatch to compare the current best model performance with the currently trained one, and then save the best model to the sagemaker model registry, that could be used later to serve the model online if needed, or use it for inference, but we can also just pull models with DVC from S3 with the required parameters that is discussed within the teams.<br>

__prediction pipeline__ <br>
The idea I had is that we use batch transform to do offline inference. The user submits the AWS batch transform script with the correct parameters (eg dvc_model_id, dataset_location, checkpoint metadata etc...) and then it gets inferences from large datasets. In this approach we have 2 lambda functions, one is for the offline inference one is for the comparison script. In both cases it runs a py script / container that we would need to get from other sources configured in lambda, and since we also do not need an API in our use case this solution is not the berst and will be updated in the next one.


__Second proposal__


![mlopsdiagram2](https://user-images.githubusercontent.com/57996039/220202657-c1ed1229-56b2-4001-9ce3-5a3447fa0ab8.png)

<br>
This solution is very similar to the first one but more optimized, and is easier to use since it mostly uses native aws sagemaker services. 
<br>
In the training account workflow I changed the AWS Batch service to AWS SageMaker pipelines. It works almost similarly, where the maintainer can configure a compute group, and the service will scale / balance the commited resources to a submitted job based on workload. In my opinion it is a better solution than using another layer (AWS Batch), reduces extra configuring and is native to sagemaker. Another change I made here is that I replaced cloudwatch with sagemaker model monitoring.

<br>
In the prediction pipeline I removed the APIs. Now the end user just submits a batch transform job, sagemaker will initialize the compute instance configured, and will run the inference on the datasets. We can collect the results to an S3 bucket and then with an additional model monitoring we can collect the results from two custom checkpoints with a script, or automate it with a service. 
