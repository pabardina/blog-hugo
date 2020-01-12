+++
date = "2020-01-12T10:20:00+02:00"
draft = false
tags = [ "airflow", "docker", "sops", "make", "kubernetes", "helm"]
title = "Airflow part 1: local environment with Docker"
slug = "airflow-local-env-docker"
+++

Git repository with code described in this article: [https://github.com/pabardina/airflow-example](https://github.com/pabardina/airflow-example)


# Airflow

Airflow is a platform to schedule and monitor workflows. It helps us to automate scripts, it is python-based and has a lot of features (interact with cloud providers, FTP, HTTP, SQL, Hadoop...). It is easy to monitor your scripts with clean UI and notifications features like e-mail or Slack message.

## Airflow at work 

In our teams, we have different kind of workflows. From simple python scripts to complex ETL used for our data warehouse.
Before Airflow, we had been using AWS Step Functions with AWS ECS. It worked pretty well, but our developers could not easily manage their workflows autonomously. We used Terraform to deploy AWS Step functions which was not used by anybody except myself.

Then, we discovered Airflow, it has simplified the way we deploy and monitor our scripts. Thus, our time to market has increased because of its ease. For information, we use KubernetesExecutor and are big fans.

### How do we use it ?

At work we use 3 main environments:

* Local : Airflow instance runs in Docker.
* Integration: Airflow runs in a small Kubernetes cluster. We deploy git branches on this environment with a Slack bot and Circleci.
* Production: Airflow runs in a Kubernetes cluster. New releases are deployed by Circleci.

#### Our git repository structure

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

#### Airflow variables and connections

If you are aware of Airflow, you know there are variables and connections. In our integration and production environments, we are using the excellent [helm chart](https://github.com/helm/charts/tree/master/stable/airflow). It handles very well the import of Airflow' secrets. So the goal was to find a way to use the same file structure in all enviroments. But when we develop locally we use Docker, so we did not have a way to easily manage our Airflow' secrets. Thus, to avoid having multiple kinds of sensitive files, I made a script to have the same feature as the helm chart. It imports the secrets to our local Airflow.

An example of our local secret file before encryption:

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

Simple yaml with two parts.

Dirty solution to import secrets in local environment:

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

#### Sops

I have already talked about [sops](https://www.bardina.net/sops-aws-kms-multi-account/) and the way we use it. 

Thus, I encrypt these sensitive files with sops. You are going to see how in the Makefile.

#### Dockerfile

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
```

#### Docker Compose 

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

Not much to say. We mount our `settings/local` directory into a `settings` directory in the webserver container to import variables / connections. It will be used by the home made script which imports secrets into Airflow.

#### Make

A Makefile is a really good tool to automate a lot of commands. We use it to start our stack, manage secrets and clean everything to do with the project.

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

* `make start` to deploy the stack
* `make edit-secret` to edit secret
* `make apply-secret` to apply secret
* `make clean` 

That's it !

Git repository with code described in this article: [https://github.com/pabardina/airflow-example](https://github.com/pabardina/airflow-example)
