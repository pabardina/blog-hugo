+++
date = "2020-01-02T10:20:00+02:00"
draft = true
tags = [ "airflow", "docker", "sops", "make", "kubernetes", "helm"]
title = "Airflow part 1: local environment with Docker"
slug = "airflow-local-env-docker"
+++

The goal of this few articles is not to explained what is Airflow, but only how to use it different environments with our team. In this article, I will focus on local environment with Docker and some other tools we like to use.
Then in part 2, I will explain how we use it in our integration / production environment.

Git repository with code described in this article: [https://github.com/pabardina/airflow-example](https://github.com/pabardina/airflow-example)


# Airflow at work
* Quick reminder
Airflow is a platform to schedule and monitor workflows. It helps us to automate scripts, it is python-based and has a lot of features (interact with cloud providers, FTP, HTTP, SQL, Hadoop...). It is easy to monitor your scripts with clean UI and notifications features like e-mail or Slack message.

* pourquoi on l utilise et ce qu on avait avant (step functions + ecs + ...)
In our teams, we have differents kind of workflows. From simple python script to complex ETL used for our data warehouse.
Before Airflow, we had been using AWS Step Functions with AWS ECS. It worked pretty well, but our developers could not easily be autonomus on managing theirs workflows. We used Terraform to deploy AWS Step functions' json. == SO ?

Then, we have discovered Airflow, it has simplified the way we deploy and monitor our scripts. Thus, our time to market has increased because of it ease. For information, we use KubernetesExecutor and are big fan.

### Presentation de nos environment et des problematiques

At work we use 3 main environments:

* Local : Simple Airflow instance launch with Docker.
* Integration: Airflow runs on a small Kubernetes cluster. We deploy git branches on this environment with a Slack bot and Circleci.
* Production: Airflow runs on a Kubernetes cluster. New release are deployed by Circleci.

 Git repository structure:

```
$ tree
.
├── Dockerfile
├── Makefile
├── dags
│   └── example_bash_operator.py
├── docker-compose.yml
├── plugins
└── settings
    ├── integration
    │   └── secret.yaml
    ├── local
    │   ├── import-secret.py
    │   └── secret.yaml
    └── production
        └── secret.yaml
```

* airflow variables / connection

If you are aware of Airflow, you know there are variables and connections. In our integration and production environments, we are using the excellent [helm chart](https://github.com/helm/charts/tree/master/stable/airflow) and it manages very well these data. Thus, to avoid having multiple kind of secrets files, I made a script to import them in our local environment.

Example of our local secret file before encryption:

```
$ cat settings/local/secret.yaml
airflow:
    connections:
    -   id: aws_default_connection
        type: s3
    -   id: postgres
        type: postgres
        host: my_endpoint
        login: my_user
        password: awesome_password
        port: 5432
    variables: '{ "aws_default_region": " eu-west-1", "environment": "local" }'
```

Simple.........

Dirty solution to import in local environment this file's data:

```
$ cat settings/local/import-secret.py
#!/bin/python3

import json
import subprocess
import yaml


secrets = yaml.full_load(open('settings/.airflow-secret.yaml'))['airflow']

json_variables = json.loads(secrets['variables'])
with open('/tmp/variables.json', 'a') as tmp_file:
    json.dump(json_variables, tmp_file, sort_keys=True, indent=4)

subprocess.Popen("/entrypoint.sh airflow variables -i /tmp/variables.json; rm -f /tmp/variables.json",
                 shell=True).wait()

for conn in secrets['connections']:
    params = f"--conn_id {conn['id']} --conn_type {conn['type']} "
    if conn.get('uri'):
        params += f"--conn_uri {conn['uri']} "

    if conn.get('host'):
        params += f"--conn_host {conn['host']} "

    if conn.get('login'):
        params += f"--conn_login {conn['login']} "

    if conn.get('password'):
        params += f"--conn_password {conn['password']} "

    if conn.get('schema'):
        params += f"--conn_schema {conn['schema']} "

    if conn.get('port'):
        params += f"--conn_port {conn['port']} "

    if conn.get('extra'):
        params += f"--conn_extra {conn['extra']}"

    subprocess.Popen(f"/entrypoint.sh airflow connections -a {params}", shell=True).wait()
```

I am not a fan of what I coded, but right now it works like a charm. If you want to improve it, please be my guest.

* sops

I have already talked about [sops](https://www.bardina.net/sops-aws-kms-multi-account/) and the way we use it. 

So yes, I encrypt these files with sops. You are going to see how in the Makefile.

* Dockerfile

We use the excellent docker [image](https://github.com/puckel/docker-airflow). We need to install a few more packages:

```
$ cat Dockerfile
FROM puckel/docker-airflow

USER root

RUN set -xe \
  && pip install papermill flake8 \
	  'apache-airflow[kubernetes]' \
	  awscli \
	  sql_magic \
  && python3 -m ipykernel install

USER airflow

RUN mkdir -p ~/.aws
```

* docker-compose 

```
$ cat docker-compose.yml
version: '3.5'
services:
    postgres:
        image: postgres:9.6
        environment:
            - POSTGRES_USER=airflow
            - POSTGRES_PASSWORD=airflow
            - POSTGRES_DB=airflow
    webserver:
        build: .
        restart: always
        depends_on:
            - postgres
        environment:
            - LOAD_EX=n
            - EXECUTOR=Local
            - AIRFLOW_HOME=/usr/local/airflow
            - AIRFLOW_PATH=/usr/local/airflow/dags
            - FERNET_KEY=OGcEH00sDw8fgOZ3S21K1K0Xr8t1TPR5G_XdVt-9HHc=
        volumes:
            - ./dags:/usr/local/airflow/dags
            - ./plugins:/usr/local/airflow/plugins
            - ./settings/local/:/usr/local/airflow/settings
        ports:
            - "1234:8080"
        command: webserver
        healthcheck:
            test: ["CMD-SHELL", "[ -f /usr/local/airflow/airflow-webserver.pid ]"]
            interval: 30s
            timeout: 30s
            retries: 3
```

Not much to say. We mount our settings/local directory into a settings directory in the webserver container to import variables / connections.



* make

A Makefile is a really good tool to automate a lot of commands. We use it to start / clean our stack.

```
$ cat Makefile
.airflow-secret:
    # Ugly but necessary otherwise postgres is not ready
	@sleep 10

    # Decrypt the airflow secret in a temporary file which is going to be mount into the container
	@sops -d settings/local/secret.yaml > settings/local/.airflow-secret.yaml
	@docker-compose exec webserver bash -c "python3 settings/import-secret.py"
	@rm -f settings/local/.airflow-secret.yaml
	@touch $@

apply-secret:
	@sops -d settings/local/secret.yaml > settings/local/.airflow-secret.yaml
	@docker-compose exec webserver bash -c "python3 settings/import-secret.py"
	@rm -f settings/local/.airflow-secret.yaml

edit-secret:
	@sops settings/local/secret.yaml

up:
	$(info Make: Starting containers.)
	@docker-compose up --build -d

start: up .airflow-secret

stop:
	$(info Make: Stopping environment containers.)
	@docker-compose stop

restart: stop start

bash:
	$(info Make: Bash in webserver container)
	@docker-compose exec webserver bash

logs:
	@docker-compose logs -f

test:
	$(info Make: Unit test dags files)
	@docker-compose exec webserver flake8 --max-line-length 120 --max-complexity 12 --statistics .

clean:
	@docker-compose down -v
	@rm -f .airflow-secret
	@rm -f settings/.airflow-secret.yaml
```

With this Makefile, our developers don't have to use another command like docker, docker-compose or sops.

* `make start` for deploy the stack
* `make edit-secret` for edit variables or connections
* `make apply-secret` for apply new variables or connections
* `make clean` 

That's it !


In part 2, I will focus on Airflow running in Kubernetes and how we deploy it with Helm. 


## Part 2
* kubernetes
** deployer avec eks + terraform
* helm : files
** command + file
* circleci 
** example circleci
* slack bot
** example + screen

## Part 3
* coreos/prometheus-operator
* prometheus-exporter..
* alertmanager