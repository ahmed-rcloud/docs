# MediciaSAFELY & Cohort Extractor - User Guide

This document is the user guide to use MediciaSAFELY and Cohort Extractor.

## Project Pipeline

To run a research study at MediciaSAFELY, the starting point is use this research template (https://github.com/MediciaAI/research-template) and customize it according to the study. 

At the root of the repository, there is an YAML file called `project.yaml`. This file defines the project pipeline. This should be the first thing to work on.

The project pipeline defines the execution order for the code in a series of actions from the start to the end (e.g. the project is automated from extracting the data to doing the analysis). 

For every `project.yaml` file, there 2 sections:

1. `version`: Should be set to `'3.0'`.
3. `actions`: This section holds one or more actions that form the project pipeline.

This is an example of the `project.yaml` file. Under the `actions` section, there are 4 actions with the following user-defined names:

1. generate_study_population
2. train_model
3. predict_model
4. r_analysis

```yaml
version: '3.0'

actions:
  generate_study_population:
    run: cohortextractor:latest generate_cohort --generate-cohort-by-sql queries_rcloud.sql --database-url=postgres://postgres:1234@192.168.0.16:5432/rcloud
    outputs:
      highly_sensitive:
        cohort: output/input.csv
  train_model:
    run: python:latest python analysis/train_model.py
    outputs:
      moderately_sensitive:
        model: output/regression_model.joblib
  predict_model:
    run: python:latest python analysis/model_predict.py
    needs: [train_model]
    outputs:
      moderately_sensitive:
        stats: output/stats.txt
  r_analysis:
    run: r:latest Rscript analysis/test_r.R
    outputs:
      moderately_sensitive:
        file1: output/file1.txt
        file2: output/file2.txt
```

Each action must have the following 2 keys:

1. `run`: The command to be executed followed by its arguments.
2. `outputs`: The outputs of the action.

Optionally, an action can include a `needs` key which specifies the list of dependencies (i.e. a list of actions contained within square brackets and separated by commas are required for this action to successfully run). For example, the `predict_model` action depends on the `train_model` action.

### `run`

The value passed to the `run` key has 3 pieces:

1. Docker Image: The Docker image that is used to run this action (e.g. `cohortextractor`, `python`, or `r`).
2. Docker Image Version: The version of the Docker image. To use the latest version, just append `:latest` to the Docker image name. 
3. The command to be executed inside the Docker image.

If you are going to create actions that use the data extracted from the database, then an action in the pipeline must use the `cohortextractor` image. This generates the data that is used by the other actions. 

Below is the value passed to the `run` of this action (which is `generate_study_population` in this example) is as follows:

```YAML
cohortextractor:latest generate_cohort --generate-cohort-by-sql queries_rcloud.sql --database-url=postgres://postgres:1234@192.168.0.16:5432/rcloud
```

The elements of this value are:

1. Docker image and version: `cohortextractor:latest`
2. The command: Out of the commands supported by the `cohortextractor` image, the `generate_cohort` command is used to get the data from the database. The `generate_cohort` command have 2 mandatory arguments:
   1. `--generate-cohort-by-sql`: A user-defined SQL file to filter to the database.
   2. `--database-url`: The database URL. For format of the database URL is `[DATABASE_ENGINE]://[USER_NAME]:[PASSWORD]@[DOMAIN_NAME]:[PORT_NUMBER]/[DATABASE_NAME]`

Note that the database engine used by MediciaSAFELY is PostgreSQL.

This is an example of the contents of the SQL file passed to the `--generate-cohort-by-sql` argument.

```sql
SELECT * FROM PATDEMOGRAPHICS_V WHERE gender=1;
```

The user can use functions and routines in the SQL code. If the code has multiple `SELECT` statements, only the output of the last `SELECT` statement is saved in the output file of the action.

### `outputs`

Each action must include an `outputs` key with at least one output, classified as either `highly_sensitive` or `moderately_sensitive`.

For example, these are the outputs of the `r_analysis` action which indicates that is will save 2 output files and mark them as *moderately sensitive*. Note that all the output files must be saved in the `output` folder. 

```yaml
outputs:
  moderately_sensitive:
    file1: output/file1.txt
    file2: output/file2.txt
```

Inside the Python or R code, the user is responsible to create this folder and save the files into it. For example, this `R` code creates the `output` folder and saves the 2 files listed above inside it.

```R
dir.create("output")

fileConn<-file("output/file1.txt")
writeLines(c("Hello","World"), fileConn)
close(fileConn)

fileConn<-file("output/file1.txt")
writeLines(c("Hello","World"), fileConn)
close(fileConn)
```

## Use MediciaSAFELY Python Library Locally

There is a Python library called `mediciasafely` which helps the users to test the MediciaSAFELY platform locally on their machines. The source code of this library is available at this GitHub repository: https://github.com/MediciaAI/mediciasafely-cli. The library is published at this link: https://pypi.org/project/mediciasafely.

Use the following command for installing the `mediciasafely` library with `pip`:

```
pip3 install mediciasafely
```

Once installed, the `mediciasafely` Python library will be installed and a command-line tool called `mediciasafely` will be created to run commands. Use one of these terminal commands check if it is working properly:

* `mediciasafely`
* `mediciasafely -h`
* `mediciasafely --help`

The following output should be displayed.

```bash
(base) C:\Users\agad>mediciasafely
usage: mediciasafely [-h] [--version] COMMAND ...

optional arguments:
  -h, --help  show this help message and exit
  --version   show program's version number and exit

available commands:

  COMMAND
    help      Show this help message and exit
    run       Run project.yaml actions locally
    codelists
              Commands for interacting with https://codelists.opensafely.org/
    pull      Command for updating the docker images used to run MediciaSAFELY studies locally
    upgrade   Upgrade the mediciasafely cli tool.
    check     Check the mediciasafely project for correctness
    jupyter   Run a jupyter lab notebook using the MediciaSAFELY environment
```

Next is to have a research study available locally. If you do not have one already, just use this template https://github.com/MediciaAI/research-template. Once it is available locally, just move the current directory to its directory. 

Now, it is time to run actions locally. Remember that the actions are listed in the `project.yaml` file. To run an action, use this command:

```
mediciasafely run <action-name>
```

Because each action uses a Docker image to run, we have to make sure that the Docker image used by this action is accessible in our machine (i.e. the user has permission). The images are automatically fetched from `ghcr.io/mediciaai` if not existing locally. Make sure that you have the permission to access the images.

For example, this command executes the `train_model` action in the example above which uses the `ghcr.io/mediciaai/python:latest` image. This command should be entered in the root directory of the research study (i.e. where the `project.yaml` file exists).

```
mediciasafely run train_model
```

> To test the cohort extractor locally without having the `ghcr.io/mediciaai/cohortextractor:latest` image, check the [Testing Cohort Extractor Locally](#Testing-Cohort-Extractor-Locally) section.

You can also pull the Docker image manually using the next command. Make sure you have permission to pull this image.

```shell
docker pull ghcr.io/mediciaai/python
```

These are the logs of the command above. The research study is completed successfully and the output file `output/regression_model.joblib` is saved successfully. You can also check the logs in the `metadata/train_model.log` file. Note that all of these paths are relative to the research study folder.

```shell
(base) C:\Users\agad\research-template>mediciasafely run train_model

Running actions: train_model

jobrunner.run loop started
train_model: Preparing
train_model: Copying in code from C:\Users\agad\research-template
train_model: Started
train_model: View live logs using: docker logs -f job-research-template-train-model-vcwzzk7plp5cw55n
train_model: Running
train_model: Finished, checking status and extracting outputs
train_model: Logs written to: C:\Users\agad\research-template\metadata\train_model.log
train_model: Extracting output file: output/regression_model.joblib
train_model: Completed successfully
train_model: Cleaning up container and volume

=> train_model
   Completed successfully

   log file: metadata/train_model.log
   outputs:
     output/regression_model.joblib  - moderately_sensitive
```

Remember to prepare the environment needed so that the action is completed successfully. For example, the `generate_study_population` action makes a query to the PostgreSQL `rcloud` database. To get it working properly, the PostgreSQL database server must be running on `192.168.0.16:5432` and there is a database user with the name `postgres` and password `1234`.

To run all the actions in the pipeline, use `run_all`.

```
mediciasafely run run_all
```

To force the actions to run even if their output files exist, use the `--force-run-dependencies` (or `-f` for short) option.

```
mediciasafely run run_model --force-run-dependencies
mediciasafely run run_model -f 
```

## Testing Cohort Extractor Locally

The cohort extractor is responsible for extracting the data from the database which is a very important feature. Because it might be tricky to debug the `cohortextractor` Docker image, it is better to install the `cohortextractor` GitHub project (https://github.com/MediciaAI/cohort-extractor) locally and make the tests.

### Install the `cohortextractor` Project

Clone the `cohortextractor` project from GitHub.

```
gh repo clone MediciaAI/cohort-extractor
```

Change the directory to the cloned project. Then, install it.

```
python3 setup.py install
```

Once installed, a command-line tool called `cohortextractor` is created to run commands. Use one of these terminal commands check if it is working properly:

* `cohortextractor`
* `cohortextractor -h`
* `cohortextractor --help`

This is the output of the `cohortextractor` command. The `cohortextractor` command only supports a single argument called `generate_cohort` which is used to generate a cohort from the database.

```shell
(base) D:\Medicia>cohortextractor
usage: cohortextractor [-h] [--version] [--verbose] {generate_cohort} ...

Generate cohorts and run models in openSAFELY framework.

positional arguments:
  {generate_cohort}  sub-command help
    generate_cohort  Generate cohort

optional arguments:
  -h, --help         show this help message and exit
  --version          Display version
  --verbose          Show extra logging info
```

### Using `cohortextractor`

The steps to use the `cohortextractor` project are:

1. Clone the research study: https://github.com/MediciaAI/research-template.
2. Open the command prompt/terminal and change the directory to the root of the research study directory.
2. Prepare the SQL file with the queries to filter the database and fetch the data.
3. Enter the `cohortextractor generate_cohort` command with the appropriate arguments.

Let's have more details to some of these steps.

#### Clone the Research Study

Use this command to clone the research template.

```
gh repo clone MediciaAI/research-template
```

#### Create the SQL File

The `cohortextractor` terminal command has a single argument called `generate_cohort` which is used to generate a cohort from the database.

For MediciaSAFELY to allow the users to apply user-defined filters to filter the data from the database, the `cohortextractor generate_cohort` command supports an argument called `--generate-cohort-by-sql`. This argument accepts either an SQL file (`.sql`) or a string with the SQL code. Because MediciaSAFELY uses PostgreSQL database engine, then the PostgreSQL commands are supported.

Based on the types of filters the user wants to apply, the SQL code can be simple like a single `SELECT` command or complex that has routines/functions. This is an example which selects all the data in the `PATDEMOGRAPHICS_V` table.

```sql
SELECT * FROM PATDEMOGRAPHICS_V;
```

Because this experiment is running locally, then the user is asked to change the table table according to the existing tables.

The SQL code can be stored into an SQL file or passed to the `generate_cohort` command as a string.

#### Run the `cohortextractor generate_cohort` Command

After preparing the SQL code, next is to run the `generate_cohort` command. This is an example. Note that this command is identical to the command used in the `project.yaml` file. Only one exception is excluding the Docker image version (i.e. we use `cohortextractor` instead of `cohortextractor:latest`).

```shell
cohortextractor generate_cohort --generate-cohort-by-sql queries_rcloud.sql --database-url=postgres://postgres:1234@192.168.0.16:5432/rcloud
```

Assuming that the SQL file name is `queries_rcloud.sql`, then this name is passed to the `--generate-cohort-by-sql` command.

The database URL is passed to the `--database-url` argument. For more details about the format of the URL, check the [Project Pipeline](#Project-Pipeline) section of this document.

Similar to the previous command, we can pass the SQL code as a string. This is an example.

```shell
cohortextractor generate_cohort --generate-cohort-by-sql "SELECT * FROM PATDEMOGRAPHICS_V;" --database-url=postgres://postgres:1234@192.168.0.16:5432/rcloud
```

If everything is OK, then this command should end up with the queried data saved in the `input.csv` file in the `output` directory of the research study.

The previous command selects all the records from the `PATDEMOGRAPHICS_V` table. The user can also control the sample size fetched by using the `--population-size` argument. For example, the next command limits the number to only 100 records.

```shell
cohortextractor generate_cohort --generate-cohort-by-sql "SELECT * FROM PATDEMOGRAPHICS_V;" --population-size 100 --database-url=postgres://postgres:1234@192.168.0.16:5432/rcloud
```

