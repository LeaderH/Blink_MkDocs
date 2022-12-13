# Installation
---
* __python3__  (python3.9)  
    install dependecies, we highly recommand using pipenv or venv to run the tool in a controlled envirionment
* __pipenv__  
	1. `pipenv install --python 3.9`
	2. `pipenv shell`

	- exit virtual environment with `exit`  

* __mongodb__ (optional)
    1. `cp db_config.py.example db_config.py`
    2. `vim db_config.py`
	- specify ***db_config.py*** for your db connection

#### db_config.py.example
```python
##############################
# MongoDB Config
##############################
"""Remote MongoDB"""
# If remote sever address is provided,
# the following local setting is ignored
MONGODB_REMOTE = None
"""Local/ host MongoDB """
MONGODB_HOST = "localhost"
MONGODB_PORT = 27017
"""Database Name"""
MONGODB_DBNAME = "Blink"

##############################
# MongoDB Collections Setting
##############################
COLLECTION_RESULT = "AppInfo"
COLLECTION_RULES = "RuleInfo"
```

#### User without the privilege to install python3.9
  
* __getting standalone python build__  
		get python3.9 from [standalone builds](https://github.com/indygreg/python-build-standalone/releases)
		(cpython3.9...install_only.tar.gz)
* __extracting and aliasing__  
	1. `tar -xvf *install_only.tar.gz`
	2. `alias python3.9="python\bin\python3.9"`
* __pipenv__  
	1. `python3.9 -m pipenv install --python 3.9`
	2. `python3.9 -m pipenv shell`