# Job Runner User Guide

This document is a guide to help users know the steps to run the Job Runner locally just to have an idea about how things work. 

## Setup Ubuntu OS

An Ubuntu OS is needed to run the MediciaSAFELY's Job Runner. This section gives instructions to have a running OS with the requirements needed.

### Install Ubuntu

Ubuntu OS can be downloaded from this link: https://ubuntu.com/download/desktop. Either install Ubuntu on a physical or virtual machine. 

Because you may need to install some libraries and build multiple Docker images (which may take much storage), it is recommended to increase the disk storage assigned to the virtual machine. Something above 60 GB is good but this might change based on whether you download many files, build many Docker images, etc.  

Try to assign this storage to the VM at the time of creating it because the virtual machine may crash if the storage is expanded later.

If a VM is used, it is better to keep a backup of it each period of time it case it something crashed. 

Once the OS starts, it is time start installing the components needed. This includes:

- Docker
- Python 3.9 and above.
- PostgreSQL

### Install Docker

To install Docker on Ubuntu, follow the instructions in this page: https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository

At the time of writing this document, these are the commands:

```shell
sudo apt-get update

sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Verify the Docker installation by printing its version. 

```
sudo docker version
```

After following the link above and getting Docker installed successfully, then you will have to use `sudo` for each Docker command. The reason is that the Docker demon (`dockerd`) uses a Unix socket which is owned by the user `root`. Other users can access the socket by using `sudo`. 

For simplicity, there is a way to use the `docker` commands without using `sudo`. This is by creating a Unix group called `docker` and add the normal (non-root) users to it. Check this link to create the `docker` group: https://docs.docker.com/engine/install/linux-postinstall

At the time of writing this document, these are the steps to create the group and add the users.

1. Create the `docker` group: `sudo groupadd docker`
2. Add the user: `sudo usermod -aG docker $USER`
3. Restart the virtual machine.
4. Activate the changes to the groups: `newgrp docker `
5. Verify that the `docker` group works: `docker run hello-world`

Note that `docker` in the last command refers to the Unix group, not the `docker` command. Now, these 2 commands have an identical behavior:

```
docker # This is the Unix group.
sudo docker # This is the docker command.
```

If the `docker` group is created successfully, then you can run Docker commands by just using `docker` instead of `sudo docker`.

```
docker version
```

If not already installed, install Docker Compose by following the instructions in this page: https://docs.docker.com/compose/install

#### Edit `.bashrc` File

Add the following 2 lines in the **.bashrc** (`/home/<USERNAME>/.bashrc`) file. The 2 variables `DOCKER_BUILDKIT` and `COMPOSE_DOCKER_CLI_BUILD` will be available as environment variables each time a new shell terminal is opened. 

```bash
export DOCKER_BUILDKIT=1
export COMPOSE_DOCKER_CLI_BUILD=1
```

These are the uses of these 2 environment variables:

1. The `DOCKER_BUILDKIT` activates BuildKit tool which is needed by some images to be built.
2. The `COMPOSE_DOCKER_CLI_BUILD` is set to 1 to use the native Docker CLI. If set to 0, then Docker Compose will use a Python client to communicate with the Docker Demon.

Enter the `source ~/.bashrc` command after adding these 2 lines to refresh without logging off and logging in again.

### Install Python 3.9

Right now, the latest LTS Ubuntu 22.04 does not come with Python 3.9 or higher installed. It only have Python 3.8.

For now, Python 3.8 works well with the Job Runner but Python 3.9 or higher is recommended. This is because we might use some of the features supported in Python 3.9 or higher later such as the dictionary merge operator `|`.

> One way to install Python 3.9 or higher in Ubuntu is to use the DeadSnakes PPA. But this is an unofficial way. Search the web about that topic. **Be careful to not delete system files**. Just verify that things are working properly after both Python and PIP get them installed.

## Clone Job Runner GitHub Repository

After preparing the Ubuntu OS, now it is time to clone the Job Runner repository from GitHub.

```
gh repo clone MediciaAI/job-runner
```

Let's assume that the project is cloned into this directorty:

```
/home/ahmedgad/MediciaAI/job-runner
```

The Job Runner is considered as a Python library. In Python, if we have a script that wants to call the library, then we have 2 options:

1. Run the script in the same directory of the library. The documentation uses this option.
2. Install the library and then use it from a script that exists anywhere (not mandatory at the same directory of the library). Use this command `python3 setup.py install`.

To make things simple, we will change the current directory of the terminal to the directory of the cloned Job Runner project. 

```
ahmedgad@ubuntu:~/MediciaAI/job-runner$ 
```

To check if the Job Runner code is readable by Python, just enter the `python3` command and then import the jobrunner. By just entering `jobrunner`, its path will be printed which is identical to the previous directory.

```shell
ahmedgad@ubuntu:~/MediciaAI/job-runner (copy)$ python3
Python 3.10.4 (main, Apr  8 2022, 17:35:13) [GCC 9.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import jobrunner
>>> jobrunner
<module 'jobrunner' from '/home/ahmedgad/MediciaAI/job-runner/jobrunner/__init__.py'>
>>> 
```

In addition to the Python code, the repository has the needed configuration files. One of the important configuration files is [dotenv-sample](https://github.com/MediciaAI/job-runner/blob/main/dotenv-sample). This file has some environment variables that specifies some parameters about the Job Runner. 

In the next section, the Job Runner will be tested locally.

### Run Job Runner Locally

The first thing we should care about is the `.env` file.

#### .env File

As mentioned above, the [dotenv-sample](https://github.com/MediciaAI/job-runner/blob/main/dotenv-sample) file saves some environment variables that are used to configure the Job Runner. But this file is not used at its current form. We need to copy this file into another file called `.env`. After creating the `.env` file, it is time to have a closer look at its variables. 

When running the Job Runner locally, it is very important to pay attention to the directories assigned to some variables. Examples of these variables include:

* `WORKDIR`: The directory in the Job Runner Docker image where the jobs' files are saved.
* `WORKSPACES`: The directory at which workspaces are saved. Its value should be identical to the previous `WORKDIR` variable's value.
* `HIGH_PRIVACY_STORAGE_BASE`: The directory to save the high privacy files.
* `MEDIUM_PRIVACY_STORAGE_BASE`: The directory to save the medium privacy files.

For simplicity, we can set all of these directories to be absolute directories inside the Job Runner project. This is an example. 

```python
WORKDIR="/home/ahmedgad/MediciaAI/job-runner/workdir"

WORKSPACES="/home/ahmedgad/MediciaAI/job-runner/workdir/"

HIGH_PRIVACY_STORAGE_BASE="/home/ahmedgad/MediciaAI/job-runner/workdir/high_privacy"

MEDIUM_PRIVACY_STORAGE_BASE="/home/ahmedgad/MediciaAI/job-runner/workdir/medium_privacy"
```

Once the Job Runner starts, the folder `workdir` will be created. Later, the `high_privacy` and `medium_privacy` folder will be created to save the output files from the jobs.

To export the variables defined in the `.env` file in the current terminal, just enter the following commands:

```shell
ahmedgad@ubuntu:~/MediciaAI/job-runner$ set -a
ahmedgad@ubuntu:~/MediciaAI/job-runner$ source .env
ahmedgad@ubuntu:~/MediciaAI/job-runner$ set +a
```

#### config.py Files

As mentioned in the previous section, the `.env` file has some environment variables that configures the Job Runner. These variables are accessed from the Python code. For example, these 2 scripts read these variables to configure the Job Runner: 

1. `/job-runner/jobrunner/config.py` 
2. `/job-runner/jobrunner/hatch/config.py`

The scripts read the values of the environment variables and assign them to Python variables that can then be accessed anywhere from the Job Runner source code. If any variable does not have a value, then the scripts can:

1. Assign a default value.
2. Raise an exception. 

These are some examples from the `/job-runner/jobrunner/config.py` script:

* `WORKDIR`: The workspace directory. It is defined in an environment variable called `WORKDIR` in the `.env` file. If this environment variable is not defined, then a default directory saved in the `default_work_dir` variable is used.
* `HIGH_PRIVACY_WORKSPACES_DIR`: The directory where the high privacy files are saved. It is saved in the environment variable `HIGH_PRIVACY_STORAGE_BASE` in the `.env` file. If no variable is defined, then the default directory is `WORKDIR/"high_privacy"`.

Assuming that there is no `.env` file, then no environment variables will exist and the default values defined in the `config.py` script will be used.

#### Run the `jobrunner` Module

Now, it is time to run the Job Runner locally. To do that, please make sure that you have a research study GitHub repository available. For example, this one https://github.com/mediciaai/research-template.

There are 2 steps to run jobs locally using the Job Runner:

1. Add the jobs.
2. Run the jobs.

##### Add the Jobs

We can add the jobs by calling the `/job-runner/jobrunner/cli/add_job.py` module from the terminal. Two parameters are passed to the command:

1. The GitHub repository of the research study. For example, https://github.com/mediciaai/research-template.
2. The name of the action to run (as defined in the [project.yaml](https://github.com/MediciaAI/research-template/blob/main/project.yaml) file). For example, `train_model`.

```shell
ahmedgad@ubuntu:~/MediciaAI/job-runner$ python3 -m jobrunner.cli.add_job https://github.com/mediciaai/research-template train_model
```

By entering this command, then a job request will be submitted to the Job Runner asking to run the `train_model` action. These are the log messages generated after entering the previous terminal command. The message `Created 1 new jobs`, at the middle of the log, indicates that 1 job request is created successfully.

```bash
ahmedgad@ubuntu:~/MediciaAI/job-runner$ python3 -m jobrunner.cli.add_job https://github.com/mediciaai/research-template train_model
Submitting JobRequest:

  {'branch': 'HEAD',
   'cancelled_actions': [],
   'commit': 'c2c2058aaab28815ed0e95068a86da779dfe36fe',
   'database_name': 'dummy',
   'force_run_dependencies': False,
   'force_run_failed': False,
   'id': '1ca804518b',
   'original': {'cancelled_actions': [],
                'force_run_dependencies': False,
                'identifier': '1ca804518b',
                'requested_actions': ['train_model'],
                'sha': 'c2c2058aaab28815ed0e95068a86da779dfe36fe',
                'workspace': {'branch': 'HEAD',
                              'db': 'dummy',
                              'name': 'test',
                              'repo': 'https://github.com/mediciaai/research-template'}},
   'repo_url': 'https://github.com/mediciaai/research-template',
   'requested_actions': ['train_model'],
   'workspace': 'test'}

2022-07-06 04:28:07.704Z Handling new JobRequest:
JobRequest(id='1ca804518b', repo_url='https://github.com/mediciaai/research-template', commit='c2c2058aaab28815ed0e95068a86da779dfe36fe', requested_actions=['train_model'], cancelled_actions=[], workspace='test', database_name='dummy', force_run_dependencies=False, force_run_failed=False, branch='HEAD', original={'identifier': '1ca804518b', 'sha': 'c2c2058aaab28815ed0e95068a86da779dfe36fe', 'workspace': {'name': 'test', 'repo': 'https://github.com/mediciaai/research-template', 'branch': 'HEAD', 'db': 'dummy'}, 'requested_actions': ['train_model'], 'force_run_dependencies': False, 'cancelled_actions': []}) 
2022-07-06 04:28:08.036Z Created 1 new jobs 
Created 1 new jobs:

  {'action': 'train_model',
   'action_commit': None,
   'action_repo_url': None,
   'cancelled': 0,
   'commit': 'c2c2058aaab28815ed0e95068a86da779dfe36fe',
   'completed_at': None,
   'created_at': '2022-07-06T04:28:08Z',
   'database_name': 'dummy',
   'id': 'mihgefhenob3d2nh',
   'image_id': None,
   'job_request_id': '1ca804518b',
   'output_spec': {'moderately_sensitive': {'model': 'output/regression_model.joblib'}},
   'outputs': None,
   'repo_url': 'https://github.com/mediciaai/research-template',
   'requires_outputs_from': [],
   'run_command': 'python:latest python analysis/train_model.py',
   'started_at': None,
   'state': 'pending',
   'status_code': None,
   'status_message': None,
   'unmatched_outputs': None,
   'updated_at': '2022-07-06T04:28:08Z',
   'wait_for_job_ids': [],
   'workspace': 'test'}
```

##### Run the Jobs

Now, it is time to run the job request submitted to the Job Runner. This is done by using the `/job-runner/jobrunner/run.py` module. This is an example.

```
python3 -m jobrunner.run
```

These are some of the log messages printed once the job is executing. Note that it is expected to get an error once the Job Runner communicates with the Job Server. It will be discussed later.

```shell
ahmedgad@ubuntu:~/MediciaAI/job-runner$ python3 -m jobrunner.run
2022-07-06 02:51:24.260Z jobrunner.run loop started 
2022-07-06 02:51:24.263Z Preparing project=research-template action=train_model id=5enl3urcsgkzq5oq
2022-07-06 02:51:24.776Z Copying in code from https://github.com/mediciaai/research-template@c2c2058aaab28815ed0e95068a86da779dfe36fe project=research-template action=train_model id=5enl3urcsgkzq5oq
2022-07-06 02:51:25.354Z Started project=research-template action=train_model id=5enl3urcsgkzq5oq
2022-07-06 02:51:25.354Z View live logs using: docker logs -f job-research-template-train-model-5enl3urcsgkzq5oq project=research-template action=train_model id=5enl3urcsgkzq5oq
2022-07-06 02:51:25.356Z Running project=research-template action=train_model id=5enl3urcsgkzq5oq
2022-07-06 02:51:26.383Z Finished, checking status and extracting outputs project=research-template action=train_model id=5enl3urcsgkzq5oq
2022-07-06 02:51:26.531Z Logs written to: /home/ahmedgad/MediciaAI/job-runner/workdir/high_privacy/workspaces/test/metadata/train_model.log project=research-template action=train_model id=5enl3urcsgkzq5oq
2022-07-06 02:51:26.531Z Extracting output file: output/regression_model.joblib project=research-template action=train_model id=5enl3urcsgkzq5oq
2022-07-06 02:51:26.659Z User is None project=research-template action=train_model id=5enl3urcsgkzq5oq
2022-07-06 02:51:26.659Z Workspace is test project=research-template action=train_model id=5enl3urcsgkzq5oq
2022-07-06 02:51:26.659Z Output is {'output/regression_model.joblib': 'moderately_sensitive'} project=research-template action=train_model id=5enl3urcsgkzq5oq
2022-07-06 02:51:26.660Z 

Files to be released: [('output/regression_model.joblib', 'moderately_sensitive')]

 project=research-template action=train_model id=5enl3urcsgkzq5oq
2022-07-06 02:51:26.660Z Current file to be released: ('output/regression_model.joblib', 'moderately_sensitive') project=research-template action=train_model id=5enl3urcsgkzq5oq
2022-07-06 02:51:26.660Z High privacy workspace directory: /home/ahmedgad/MediciaAI/job-runner/workdir/high_privacy/workspaces/test project=research-template action=train_model id=5enl3urcsgkzq5oq
2022-07-06 02:51:26.660Z High privacy file directory: /home/ahmedgad/MediciaAI/job-runner/workdir/high_privacy/workspaces/test/output/regression_model.joblib project=research-template action=train_model id=5enl3urcsgkzq5oq
2022-07-06 02:51:26.660Z Path of the file to be released: /home/ahmedgad/MediciaAI/job-runner/workdir/test/high_privacy/workspaces/test/output/regression_model.joblib project=research-template action=train_model id=5enl3urcsgkzq5oq
2022-07-06 02:51:26.661Z SHA256 of the file to be released: c76e04ed09ee45e1f0efd1ea5a4ceef6f2a46361745ad8fbc44a4e62478432f1 project=research-template action=train_model id=5enl3urcsgkzq5oq
```

Once the Job Runner prints this log message, this means the output file out of the `train_model` action is created successfully in this directory: `/home/ahmedgad/MediciaAI/job-runner/workdir/high_privacy/workspaces/test/output/regression_model.joblib`.

```shell
2022-07-06 02:51:26.660Z High privacy file directory: /home/ahmedgad/MediciaAI/job-runner/workdir/high_privacy/workspaces/test/output/regression_model.joblib project=research-template action=train_model id=5enl3urcsgkzq5oq
```

There is also a log file with the log messages printed while the action is running. For the `train_model` action, the log file is saved in this directory: `/home/ahmedgad/MediciaAI/job-runner/workdir/high_privacy/workspaces/test/metadata/train_model.log`

This is the content of this log file. Its state is `succeeded` which means this action is executed successfully.

```
state: succeeded
commit: c2c2058aaab28815ed0e95068a86da779dfe36fe
docker_image_id: sha256:dea598cca4c0048069008f83734624a7d05b6e84e77b3b0d6ec8d3b0091a1494
job_id: npesghoit7cbhxch
created_at: 2022-07-06T03:22:48Z
completed_at: 2022-07-06T03:22:52Z

Completed successfully

outputs:
  output/regression_model.joblib  - moderately_sensitive
```

After the Job Runner creates the output file, then it starts to communicate with the Job Server to create a new file release and upload the file. Because this local environment is not configured to communicate with the Job Server, then the connection will be refused. As a result, the Job Runner will raise an exception.

These are the log messages of the kind of exception raised. 

```shell
Traceback (most recent call last):
  File "/usr/lib/python3.10/runpy.py", line 196, in _run_module_as_main
    return _run_code(code, main_globals, None,
  File "/usr/lib/python3.10/runpy.py", line 86, in _run_code
    exec(code, run_globals)
  File "/home/ahmedgad/MediciaAI/job-runner/jobrunner/run.py", line 596, in <module>
    main()
  File "/home/ahmedgad/MediciaAI/job-runner/jobrunner/run.py", line 63, in main
    active_jobs = handle_jobs(api)
  File "/home/ahmedgad/MediciaAI/job-runner/jobrunner/run.py", line 92, in handle_jobs
    handle_running_job(job)
  File "/home/ahmedgad/MediciaAI/job-runner/jobrunner/run.py", line 173, in handle_running_job
    mark_job_as_completed(job)
  File "/home/ahmedgad/MediciaAI/job-runner/jobrunner/run.py", line 507, in mark_job_as_completed
    response = workspace_release(workspace=workspace, release=release, user=job_metadata["run_by_user"])
  File "/home/ahmedgad/MediciaAI/job-runner/jobrunner/hatch/app.py", line 33, in workspace_release
    response = models.create_release(workspace, workspace_dir, release, user)
  File "/home/ahmedgad/MediciaAI/job-runner/jobrunner/hatch/models.py", line 81, in create_release
    response = api_client.create_release(workspace, release, user)
  File "/home/ahmedgad/MediciaAI/job-runner/jobrunner/hatch/api_client.py", line 37, in create_release
    response = client.post(
  File "/home/ahmedgad/.local/lib/python3.10/site-packages/httpx/_client.py", line 1130, in post
    return self.request(
  File "/home/ahmedgad/.local/lib/python3.10/site-packages/httpx/_client.py", line 802, in request
    request = self.build_request(
  File "/home/ahmedgad/.local/lib/python3.10/site-packages/httpx/_client.py", line 345, in build_request
    headers = self._merge_headers(headers)
  File "/home/ahmedgad/.local/lib/python3.10/site-packages/httpx/_client.py", line 412, in _merge_headers
    merged_headers.update(headers)
  File "/home/ahmedgad/.local/lib/python3.10/site-packages/httpx/_models.py", line 198, in update
    headers = Headers(headers)
  File "/home/ahmedgad/.local/lib/python3.10/site-packages/httpx/_models.py", line 69, in __init__
    self._list = [
  File "/home/ahmedgad/.local/lib/python3.10/site-packages/httpx/_models.py", line 73, in <listcomp>
    normalize_header_value(v, encoding),
  File "/home/ahmedgad/.local/lib/python3.10/site-packages/httpx/_utils.py", line 54, in normalize_header_value
    return value.encode(encoding or "ascii")
AttributeError: 'NoneType' object has no attribute 'encode'
```

#### Prepare the Environment Needed by the Jobs

The reason why we were able to run the `train_model` action successfully is that we already prepared the environment needed to get this action run. This action only needs Python 3 which is already installed locally.

For other actions, there are other requirements to get them run. For example, this repository (https://github.com/mediciaai/research-template) has an action named `generate_study_population` which needs a PostgreSQL database to run at this URL `postgres://postgres:1234@192.168.0.16:5432/rcloud`. So, if you do not have PostgreSQL installed, then this action will fail.

If the environment needed by all the actions is prepared, then you can add all the actions by using the `run_all` command.

```shell
ahmedgad@ubuntu:~/MediciaAI/job-runner$ python3 -m jobrunner.cli.add_job https://github.com/mediciaai/research-template run_all
```


