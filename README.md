# Seminario MLOps

## First Hands On - Get the code

Fork the code from:
```
https://github.com/anllogui/masteria3_22
```

* Get the code:
```
git clone https://github.com/anllogui/masteria3_22.git
```

* Modify Readme.md
* Execute following commands:
```
git status # show files changed
git diff # show file changes
git add . # add changes to the index
git status # show files added in green
git commit # commit changes to local repository
git status # shows no changes
git push # push changes to repository
```
## Code organization 
There are 3 main folders:
- *training*: code and data needed for training
- *exploitation*: code to deploy de model
- *mlflow_server*: folder to store mlflow data 

We will start with the training folder:
- model: notebook for training the model
- tests: data testing

## Second Hands On - Configure environment

### Install depencendies

- Install conda: https://conda.io/docs/user-guide/install/index.html#


### Create environment

In the training folder type:
```
conda env create -f environment.yml
conda activate ci_training
```

## Third Hands On - Unit Tests

* Test for invalid valid data:
```
pytest tests/tests_data.py --datafile="wrong_data"
```

* Test for valid data:
```
pytest tests/tests_data.py --datafile="SalaryData1"
```

## Fourth Hands On - MLFlow

### Train the Model

Execute Notebook
- Run Jupyter
```
jupyter notebook
```
- Go to "nb/simple_regression.ipynb".
- Execute Notebook

#### Mlflow Local 
Review results:
- Exexute MLFlow ui:
```
cd nb
mlflow ui
```
- go to: http://127.0.0.1:5000

## Fifth Hands On

### Expose the model

The service is developed in "masteria3_22/flaskr/linreg.py".

To run the service:

Create environment:

```
conda deactiate 
conda env create -f environment.yml
conda activate ci_exploitiation
```

**Mac/Linux**:
```
cd ..
export FLASK_APP=flaskr
export FLASK_ENV=development
pip install -e .
flask run
```

**Windows**:
```
cd ..
set "FLASK_APP=flaskr"
set "FLASK_ENV=development"
pip install -e .
flask run
```

To test the service:
```
curl -i -H "Content-Type: application/json" -X POST -d '{"yearsOfExperience":8}' http://localhost:5000/
```

### Third Hands On

#### Automatize

Automatize Model Training and Versioning
```
papermill simple_regression.ipynb output.ipynb -p data_ver 1 -p model_ver 1
```
#### Continuous integration

##### Jenkins
Install Jenkins for ubuntu:
https://www.jenkins.io/doc/book/installing/linux/#debianubuntu

After installing Jenkins:
```
sudo service jenkins start
```
To access to jenkins: http://localhost:8080


Install Anaconda for all users and give permissions:
```
sudo addgroup anaconda
sudo chgrp -R anaconda /opt/anaconda3
sudo adduser jenkins anaconda
sudo chmod 777 -R /opt/anaconda3
```
#### Automated Training in Jenkins

```
#!/bin/bash
echo "---- SETING ENVS ---- "

export PATH=$PATH:/Users/anllogui/miniconda3/bin
PYENV_HOME=$WORKSPACE/venv/
export MLFLOW_TRACKING_URI="http://127.0.0.1:5000"

cd training

echo "---- GETING PROPERTIES ----"

file="./train.properties"

if [ -f "$file" ]
then
  echo "$file found."

  while IFS='=' read -r key value
  do
    key=$(echo $key | tr '.' '_')
    eval ${key}=\${value}
  done < "$file"

  echo "Model Version = " ${model_version}
  echo "Data Version  = " ${data_version}
else
  echo "$file not found."
fi

echo "---- CLEANING ENVIRONMENT ----"
if [ -d $PYENV_HOME ]; then
    echo "- Project exists: cleanning.."
    rm -Rf $PYENV_HOME 
fi
source /Users/anllogui/miniconda3/etc/profile.d/conda.sh
echo "*** creating env ***"
echo $PYENV_HOME
mamba env create -f environment.yml --prefix $PYENV_HOME
conda activate ci_training
cd nb
papermill Simple_Regression.ipynb output.ipynb -p data_ver ${data_version} -p model_ver ${model_version}

#ls -la ../models

#curl -v -u admin:admin -X POST 'http://localhost:8081/service/rest/v1/components?repository=maven-releases' -F "maven2.groupId=models" -F "maven2.artifactId=simple_regresion" -F "maven2.version=${data_version}.${model_version}" -F "maven2.asset1=../models/linear_regression_model_v${model_version}.pkl" -F "maven2.asset1.extension=pkl"

```

#### Nexus
(This step is optional)
Install Nexus for Ubuntu:
https://medium.com/@everton.araujo1985/install-sonatype-nexus-3-on-ubuntu-20-04-lts-562f8ba20b98


#### Start MLFlow as a server
Create Mlflow Project:
mkdir mlflow_server
conda create -n mlflow_server python=3
conda activate mlflow_server
pip install mlflow
```
mlflow server --backend-store-uri sqlite:///mydb.sqlite --default-artifact-root /Users/anllogui/dev/mlflow_server/artifacts
```


#### Train Shell
```
#!/bin/bash
echo "---- SETING ENVS ---- "

export PATH=$PATH:/home/anllogui/anaconda3/bin
PYENV_HOME=$WORKSPACE/venv/
export MLFLOW_TRACKING_URI="http://127.0.0.1:5000"

cd training

echo "---- GETING PROPERTIES ----"

file="./train.properties"

if [ -f "$file" ]
then
  echo "$file found."

  while IFS='=' read -r key value
  do
    key=$(echo $key | tr '.' '_')
    eval ${key}=\${value}
  done < "$file"

  echo "Model Version = " ${model_version}
  echo "Data Version  = " ${data_version}
else
  echo "$file not found."
fi

echo "---- CLEANING ENVIRONMENT ----"
if [ -d $PYENV_HOME ]; then
    echo "- Project exists: cleanning.."
    rm -Rf $PYENV_HOME 
fi
source /home/anllogui/anaconda3/etc/profile.d/conda.sh
echo "*** creating env ***"
echo $PYENV_HOME
conda env create -f environment.yml --prefix $PYENV_HOME
conda activate pythonCI
cd nb
papermill Simple_Regression.ipynb output.ipynb -p data_ver ${data_version} -p model_ver ${model_version}

ls -la ../models

curl -v -u admin:admin -X POST 'http://localhost:8081/service/rest/v1/components?repository=maven-releases' -F "maven2.groupId=models" -F "maven2.artifactId=simple_regresion" -F "maven2.version=${data_version}.${model_version}" -F "maven2.asset1=../models/linear_regression_model_v${model_version}.pkl" -F "maven2.asset1.extension=pkl"

```

## Docker 

- Build and run training:
```
cd training
docker build -t training .; docker run --name=training -v ~/../output:/output --rm --network host training
```
- Build and run execution:
```
cd exploitation
docker build -t model-exploitation .; docker run -p 127.0.0.1:5000:5000 model-exploitation
````
### Docker compose
mlflow server:
```
docker-compose run --service-ports mlflow_server
```
training:
```
docker-compose run --service-ports training
```
all:
```
docker-compose up --build
```
- Delete old images:
```
docker system prune -a
```

- Connect to a container:
```
docker exec -it <container name> /bin/bash
```
