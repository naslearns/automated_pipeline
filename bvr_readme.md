```
Create Environment
```
```
conda create -n wineq python=3.8 -y
conda activate wineq
```
```
Create requirements.txt
```
```
dvc
dvc[gdrive]
scikit-learn
```
```
pip install -r requirements.txt
```
```
Create template.py
```
```
import os

dirs = [
    os.path.join("data", "raw"),
    os.path.join("data", "processed"),
    "notebooks",
    "saved_models",
    "src",
    "data_given"
]

for dir_ in dirs :
    os.makedirs(dir_, exist_ok=True)
    with open(os.path.join(dir_, ".gitkeep"), "w") as f :
        pass

files = [
    "dvc.yaml",
    "params.yaml",
    ".gitignore",
    os.path.join("src", "__init__.py")
]


for file_ in files :
    with open(file_, "w") as f :
        pass
```
```
python template.py
```
```
Observe project directory structure has been created 
```
```
Download data file (winequality.csv) from 
```
```
https://drive.google.com/drive/folders/18zqQiCJVgF7uzXgfbIJ-04zgz1ItNfF5
```
```
copy winequality.csv to data_given folder 
```
```
git init
dvc init
dvc add data_given/winequality.csv
git add .
git commit -m "csv commit"
git remote add origin https://github.com/naslearns/automated_pipeline.git 
git push -u origin main
```
```Update params.yaml file```
```base :
  project : winequality-project
  random_state : 42
  target_col : TARGET

data_source : 
  s3_source : data_given/winequality.csv

load_data :
  raw_dataset_csv : data/raw/winequality.csv

split_data :
  train_path : data/processed/train_winequality.csv
  test_path : data/processed/test_winequality.csv
  test_size : 0.2

estimators :
  ElasticNet :
    params :
      # alpha : 0.88
      # l1_ratio : 0.11
      # alpha : 0.9
      # l1_ratio : 0.4
      alpha : 0.2
      l1_ratio : 0.2

model_dir : saved_models

reports :
  params : report/params.json
  scores : report/scores.json

webapp_model_dir : prediction_service/model/model.joblib

mlflow_config :
  artifacts_dir : artifacts
  experiment_name : ElasticNet regression
  run_name : mlops
  registered_model_name : ElasticNetWineModel
  remote_server_uri : http://0.0.0.0:1234
```
```
git add .
git commit -m "commited1"
git push -u origin main
```

```Update requirements.txt```

```
dvc
dvc[gdrive]
scikit-learn
pandas
pytest
tox
flake8
flask
gunicorn
mlflow
```
```pip install -r requirements.txt
```

```create create load_data.py and get_data.py```

```get_data.py```
```
import os
import yaml
import pandas as pd
import argparse

def read_params(config_path) :
    with open(config_path) as yaml_file :
        config = yaml.safe_load(yaml_file)
    return config

def get_data(config_path) :
    config = read_params(config_path)

    print(config)

    data_path = config["data_source"]["s3_source"]

    df = pd.read_csv(data_path, sep=",", encoding="utf-8")

    print(df)

    return df


if __name__ == "__main__" :
    args = argparse.ArgumentParser()
    args.add_argument("--config", default="params.yaml")
    parsed_args = args.parse_args()
    data = get_data(config_path=parsed_args.config)
```

```Create load_data.py```

```import os
from get_data import read_params, get_data
import argparse

def load_and_save(config_path) :
    config = read_params(config_path)
    df = get_data(config_path)

    new_cols = [col.replace(" ", "_") for col in df.columns]
    print(new_cols)

    raw_data_path = config["load_data"]["raw_dataset_csv"]

    df.to_csv(raw_data_path, sep=",", index=False, header=new_cols)


if __name__ == "__main__" :
    args = argparse.ArgumentParser()
    args.add_argument("--config", default="params.yaml")
    parsed_args = args.parse_args()
    load_and_save(config_path=parsed_args.config)
```
```Add stage to dvc.yaml file```
```stages :
  load_data :
    cmd : python src/load_data.py --config=params.yaml
    deps :
      - src/get_data.py
      - src/load_data.py
      - data_given/winequality.csv
    outs :
      - data/raw/winequality.csv
```
```
dvc repro
```
```Create split_data.py file```
```
import os
import argparse 
import pandas as pd 
from sklearn.model_selection import train_test_split
from get_data import read_params


def split_and_saved_data(config_path) :
    config = read_params(config_path)
    test_data_path = config["split_data"]["test_path"]
    train_data_path = config["split_data"]["train_path"]
    raw_data_path = config["load_data"]["raw_dataset_csv"]
    split_ratio = config["split_data"]["test_size"]
    random_state = config["base"]["random_state"]

    df = pd.read_csv(raw_data_path, sep=",")

    train, test = train_test_split(df, test_size=split_ratio, random_state=random_state)

    train.to_csv(train_data_path, sep=",", index=False, encoding="utf-8")
    test.to_csv(test_data_path, sep=",", index=False, encoding="utf-8")


if __name__ == "__main__" :
    args = argparse.ArgumentParser()
    args.add_argument("--config", default="params.yaml")
    parsed_args = args.parse_args()
    split_and_saved_data(config_path=parsed_args.config)
```

```Update dvc.yaml```

```
stages :
  load_data :
    cmd : python src/load_data.py --config=params.yaml
    deps :
      - src/get_data.py
      - src/load_data.py
      - data_given/winequality.csv
    outs :
      - data/raw/winequality.csv

  split_data :
    cmd : python src/split_data.py --config=params.yaml
    deps :
      - src/split_data.py
      - data/raw/winequality.csv
    outs :
      - data/processed/train_winequality.csv
      - data/processed/test_winequality.csv
```
dvc repro



