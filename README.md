This is a MLOps project based on the [Kaggle Used Car Auction Prices dataset](https://www.kaggle.com/datasets/tunguz/used-car-auction-prices). The goal is to master the full lifetime cycle of ML model development. Project is fully created on the Yandex Cloud and covers:

- Creation of a model (python, sklearn, xgboost)
- Its tracking while training and validating (Yandex Cloud, s3-bucket, MLflow)
- Orchestrating of retraining model and switching outdated model (Prefect)
- Deployment of the best version as a web-service (docker, Flask) 
- Online and batch monitoring in production (Evidently, Prometeus, Graphana)
- Unit and Integration tests
- Using best practices with pytest, pylint, black, isort and pre-commit hooks

This guide contains all the instructions for reproducing the project, and the following links lead to a description of the main stages of creation.

### Problem description
The dataset contains historical information about characteristics of used cars put up for various auctions and its selling prices. I want to create a web service will allows us to predict the initial price of a car on the basis of its type and information about recent transactions. 
Of course each car has its own characteristics and unique condition, nevertheless, service will allows a potential seller to evaluate his auto in the current market or projected price can be used by an auction as a starting point for further bidding.

This guide contains all the instructions for reproducing the project, and the following links lead to a description of the main stages of creation.

### Stage-0 Creation of cloud environment and preparation of original dataset
The dataset used for this project contains data from December'14 till July'15. Lets imitate the working of a web-service and imagine that the current data is 2014-5-30.
So for the training purposes i want to use data from December'14 till March'15 and April'15 will be the test. Load data function getting the current data and periods parameter equals number of months for training+validation&test.
I divided  the initial kaggle's data into files by month and upload these files in Yandex Cloud Object Storage (aws s3-bucket analog).
At the next stages of monitoring and orchestraring model on the latest data one can simply invoke the training scrip with parametes of current date and the desired number of periods and it will automaticly retrain the model considering the previous month as a test.

### Stage-1 Baseline models and pipelines
I used Ridge, Random Forest, XGBoost Regressions as a baseline. There are two steps of data preparation: dropping NA of important columns (mske, model, trim - that specifying what the car actually was) and training Column transformer, which imputes missing values. 

Unfortunetly training of Random Forest and XGBoost regressions usually take about 2 hours so I turned off these models for reviewing puproses 

(but you can turn on all of them, for that:
- go to the folder **orchestration_manager**
- locate the section flow **def main()** and uncomment rows in the **model** variable
- model will be train for all models and select the best one of them on the certain period
)

### Stage-2 Searching the best parameters with MLFlow tracking service
I had been using MLFlow to tracking of training process.
I used Hyperopt library in order to train each of the models. So it is returns the set of parameters that gives me the best model prediction on the train+valid part.
I had been training each model on a hyperopt grid for 20 times and takes the best ones on valid dataset.
On the next step I created predictions on the test dataset (final month of desired period) and promoted the best model based on _Mean absolute percentile error (MAPE)_ metrics.
Hyperopt tried to minimize MAPE while training as soon as i think that it is quite suitable metric for cars market.

After that, the best models are combined with preprocessor into a pipeline so i can save just a solid model into s3 bucket. And use just _model.predict()_ without loading any preprocessors of features.

Also MLFlow helped me to create mechanism of switching of between models if newly trained model will be better than current production model.

### Stage-3 Initial orchestrating with Prefect 2
I covered previously constructed model with into @flows and @tasks in order to Prefect Orion agent will be able automaticly launch retrain process on the end of each month after getting report from Evidently service and if model drift will be located.

### Stage-4 Deployment final model with Flask as a web-service 
I took an advantage of Flask to create web service that gets production model with help of MLFlow service and use it to predict price by request.

### Stage-5 Monitoring
Evidently service helps me to create online monitoring of prediction service.
On the end of each month Prefect launched creation of statistical report based on received data. So the prediction service can check is there a drift of production model. If so manager-service will invoke retrain process with the latest data.

### Stage-6 Tests & Best practices
There are few unit tests and integration test of deployment of prediction service.
I had beed using pylint, isort and black to make the code more estetical. Pre-commit hooks for manage the process.

### Deployment for reviewing
You will need a VM (I used Yandex Cloud for that) with :
- Ubuntu server
- public IP address of the server
- bucket storage (s3 or analog)
- AWS credentials for access to bucket
- cloud endpoint url to access to bucket
- install pip
- install pipenv
- install docker & docker-compose
- install make
 
### [Full deployment instructions](https://github.com/K0nkere/kkr-mlops-project/issues/9#issue-1369072636)
(better to see for detailed walkthrough)

### Fast Run

Clone the repo from github
Create your own bucket with name <your_bucket_name> in the Cloud Service UI or with CLI command if you havent one
```
aws --endpoint-url=https://storage.yandexcloud.net/ s3 mb s3://<your_bucket_name>
```
(Yandex Cloud Object Storage example)

Edit the **my.env** file in the _project folder_ and place your values
```
PUBLIC_SERVER_IP=<your_public_ip>                               #insert
MLFLOW_S3_ENDPOINT_URL=<endpoint_irl>                           #like https://storage.yandexcloud.net
AWS_DEFAULT_REGION=<defult_region>                              #like ru-central1
AWS_ACCESS_KEY_ID=<your_key_id>                                 #insert yours
AWS_SECRET_ACCESS_KEY=<your_secret_key>                         #insert yours
BACKEND_URI=sqlite:////mlflow/database/mlops-project.db         #leave it as it is
ARTIFACT_ROOT=s3://<your_bucket_name>/mlflow-artifacts/         #insert your bucket_name
```
Open _project folder_ and run few terminals

Terminal 1
`make setup` > `cd ..` > `docker ps` > check ports are free > `make preparation`

Terminal 2
from project/orchestration_manager
`pipenv shell` > `cd ..` >`bash run-manager.sh` 

Terminal 3
from project/orchestration_manager
`pipenv shell` > `cd ..` > `python send-data.py 2015-5-30 100`

Use the data from 2015-2 to 2015-7
