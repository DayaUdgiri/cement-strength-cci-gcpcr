## Cement Strength Prediction

#### Problem Statement:
    To build a regression model to predict the concrete compressive strength based on 
    the different features in the training data. 
       
#### Data Description
    
    Given is the variable name, variable type, the measurement unit and a brief description. 
    The concrete compressive strength is the regression problem. The order of this listing 
    corresponds to the order of numerals along the rows of the database. 
           
#### Data Validation
    
    In This step, we perform different sets of validation on the given set of training files.
    
    Name Validation: We validate the name of the files based on the given name in the schema file. We have 
    created a regex patterg as per the name given in the schema fileto use for validation. After validating 
    the pattern in the name, we check for the length of the date in the file name as well as the length of time 
    in the file name. If all the values are as per requirements, we move such files to "Good_Data_Folder" else
    we move such files to "Bad_Data_Folder."
    
    Number of Columns: We validate the number of columns present in the files, and if it doesn't match with the
    value given in the schema file, then the file id moves to "Bad_Data_Folder."
    
    Name of Columns: The name of the columns is validated and should be the same as given in the schema file. 
    If not, then the file is moved to "Bad_Data_Folder".
    
    The datatype of columns: The datatype of columns is given in the schema file. This is validated when we insert
    the files into Database. If the datatype is wrong, then the file is moved to "Bad_Data_Folder."
    
    Null values in columns: If any of the columns in a file have all the values as NULL or missing, we discard such
    a file and move it to "Bad_Data_Folder".
    
#### Data Insertion in Database
     
     Database Creation and Connection: Create a database with the given name passed. If the database is already created,
     open the connection to the database.
     
     Table creation in the database: Table with name - "Good_Data", is created in the database for inserting the files 
     in the "Good_Data_Folder" based on given column names and datatype in the schema file. If the table is already
     present, then the new table is not created and new files are inserted in the already present table as we want 
     training to be done on new as well as old training files.
     
     Insertion of file in the table: All the files in the "Good_Data_Folder" are inserted in the above-created table. If
     any file has invalid data type in any of the columns, the file is not loaded in the table and is moved to 
     "Bad_Data_Folder".
     
#### Model Training
    
     Data Export from DB: The data in a stored database is exported as a CSV file to be used for model training.
     
     Data Preprocessing: 
        Check for null values in the columns. If present, impute the null values using the KNN imputer.
        
        Check if any column has zero standard deviation, remove such columns as they don't give any information during 
        model training.
        
     Clustering: KMeans algorithm is used to create clusters in the preprocessed data. The optimum number of clusters 
     is selected

     Model Selection:
        After clusters are created, we find the best model for each cluster. 
        We are using two algorithms, "Random forest Regressor" and “Linear Regression”. For each cluster, 
        both the algorithms are passed with the best parameters derived from GridSearch. 
        We calculate the Rsquared scores for both models and select the model with the best score. 
        Similarly, the model is selected for each cluster. All the models for every cluster are saved for use in prediction. 
  
## Create a file "Dockerfile" with below content

```
FROM python:3.7
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
ENTRYPOINT [ "python" ]
CMD [ "main.py" ]
```

## to create requirements.txt

```buildoutcfg
pip freeze>requirements.txt
```

## In our Code Directory --> create a file ".circleci\config.yml" with following content
```
version: 2.1
orbs:
  gcp-gcr: circleci/gcp-gcr@0.6.1
  cloudrun: circleci/gcp-cloud-run@1.0.0
jobs:
  build_test:
    docker:
      - image: circleci/python:3.6.2-stretch-browsers
    steps:
      - checkout
      - restore_cache:
          key: deps1-{{ .Branch }}-{{ checksum "requirements.txt" }}
      - run:
          name: Install Python deps in a venv
          command: |
            echo 'export PATH=~$PATH:~/.local/bin' >> $BASH_ENV && source $BASH_ENV
            python3 -m venv venv
            . venv/bin/activate
            pip install --upgrade pip
            pip install -r requirements.txt
      - save_cache:
          key: deps1-{{ .Branch }}-{{ checksum "requirements.txt" }}
          paths:
            - "venv"
      - run:
          command: |
            . venv/bin/activate
            python -m pytest -v tests/test_script.py
      - store_artifacts:
          path: test-reports/
          destination: tr1
      - store_test_results:
          path: test-reports/
  build_push_image_cloud_run_mangaged:
    docker:
      - image: circleci/python:3.6.2-stretch-browsers
    steps:
      - checkout
      - setup_remote_docker:
          docker_layer_caching: false
      - run:
          name: Build app binary and Docker image
          command: |
            echo 'export PATH=~$PATH:~/.local/bin' >> $BASH_ENV
            echo ${GCP_PROJECT_KEY} | base64 --decode --ignore-garbage > $HOME/gcloud-service-key.json
            echo 'export GOOGLE_CLOUD_KEYS=$(cat $HOME/gcloud-service-key.json)' >> $BASH_ENV
            echo 'export TAG=${CIRCLE_SHA1}' >> $BASH_ENV
            echo 'export IMAGE_NAME=$CIRCLE_PROJECT_REPONAME' >> $BASH_ENV && source $BASH_ENV
            python3 -m venv venv
            . venv/bin/activate
            pip install --upgrade pip
            pip install -r requirements.txt
            docker build -t us.gcr.io/$GOOGLE_PROJECT_ID/$IMAGE_NAME -t us.gcr.io/$GOOGLE_PROJECT_ID/$IMAGE_NAME:$TAG .
      - gcp-gcr/gcr-auth:
          gcloud-service-key: GOOGLE_CLOUD_KEYS
          google-project-id: GOOGLE_PROJECT_ID
          google-compute-zone: GOOGLE_COMPUTE_ZONE
      - gcp-gcr/push-image:
          google-project-id: GOOGLE_PROJECT_ID
          registry-url: "us.gcr.io"
          image: $IMAGE_NAME
      - cloudrun/deploy:
          platform: "managed"
          image: "us.gcr.io/$GOOGLE_PROJECT_ID/$IMAGE_NAME"
          service-name: $IMAGE_NAME
          region: $GOOGLE_COMPUTE_ZONE
          unauthenticated: true
workflows:
  build_test_deploy:
    jobs:
      - build_test
      - build_push_image_cloud_run_mangaged:
          requires:
            - build_test
          filters:
            branches:
              only:
                - main
```


## initialize git repo
```
git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin <github_url>
git push -u origin main
```

##Create a new Project in GCP

#### Capture GOOGLE_PROJECT_ID
#### Create a Service account for this project and generate the key in .jason format
#### Convert the key into base64

## Create an account at CircleCI and setup the project using this repo from github

<a href="https://circleci.com/login/">Circle CI</a>


## Select project setting in CircleCI and add below environment variable

```
GOOGLE_PROJECT_ID: The Project ID for your Google Cloud project. This value can be retrieved from the project card in the Google Cloud Dashboard.
GCP_PROJECT_KEY: The base64 encoded result from the previous section.
GOOGLE_COMPUTE_ZONE: The value of the region to target your deployment.
```


## To update the modification from local to github repo

```
git add .
git commit -m "message"
git push -u origin main
```

## Above push will start the deployment in CircleCI
