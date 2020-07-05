# COSC2759 System Deployment and Operation - Assignment 1

**Student Name: Thach Ngoc Nguyen**

**Student Number: s3651311**

The goal of this assignement is for ACME corp to be able to commit the files into their own GitHub repository and set up a *CircleCI Pipeline*, and have it work with no modification to the files necessary.

In the root directory, a CircleCi configuration file is provided, in which defines a pipeline that automates build, test and package tasks.

The `package.json` is ready to be installed via `npm`, with all the neccessary dependencies and scripts clarified. 

---

## Running unit tests

In the `src/package.json` file, a `test-unit` script is prepared to initiate the unit test with *Mocha*. The command is written with the `--reporter mocha-junit-reporter` flag to output `unit_test_results.xml`. The alternate location of `unit_test_results.xml` is specified with `--reporter-options mochaFile=...`

```
"test-unit": "cross-env NODE_ENV=unit nyc mocha test/unit --reporter mocha-junit-reporter --reporter-options mochaFile=./test-output/unit_test_results.xml",

```

The result of the test can be found at `src/test-output`.

Artifacts can be collected from the *CircleCI Pipeline*.

In the *CircleCI Pipeline*, the test is automated in `build` job:

```
- run:
    name: Run unit tests
    command: |
    cd src
    npm run test-unit
```

---

## Running linting

In the `src/package.json` file, a `test-lint` script is prepared to initiate linting with *ESLint*. The output format of the command is specified with the `--format` flag. The `--output-file` flag indicates which file the report is written to.

```
"test-lint": "eslint ./ --format junit --output-file ./test-output/eslint_result.xml",
```

The result of linting can be found at `src/test-output`.

Artifacts can be collected from the *CircleCI Pipeline*.

In the *CircleCI Pipeline*, linting is automated in `build` job:

```
- run:
    name: Run linting
    command: |
        cd src
        mkdir -p test-output
        npm run test-lint
```

---

## Generating code coverage report

In the `src/package.json` file, a `test-coverage` script is prepared to generate code coverage reports with *istanbul(nyc)* after the test is finalized. There are `.lcov` and `.json` types generated with the `--reporter` tag:

```
"test-coverage": "cross-env NODE_ENV=unit nyc report --reporter=lcov --reporter=json",
```

The reports can be found at `src/coverage`.

Artifacts can be collected from the *CircleCI Pipeline*.

In the **CircleCI Pipeline**, generating code coverage report is automated in `build` job:

```
- run:
    name: Generate code coverage reports
    command: |
        cd src
        npm run test-coverage
```

The code coverage report is also validated on CodeCov.io:

```
- run:
    name: Validate code coverage
    command: |
        bash <(curl -s https://codecov.io/bash)
```

---

## Multiple failure scenarios

There are 2 example failure scenarios that are implemented on branch f/multiple-failure-scenarios: 
    - One that fails the unit test
    - One that fails the linting

A screenshot of the *CircleCI pipeline* of this branch is provided in the `Screenshots` folder in the submission. To check the commits, do:

```
git checkout f/multiple-failure-scenarios

git log --oneline
```

---

## Generating an artefact that can be deployed

This artefact, which is a built Docker image (.tar), is only generated on master branch in `pack` job.

For this job, `setup_remote_docker` is used to create a remote environment for the primary container to run any docker-related commands.

The image is built using `docker build` with `Dockerfile` located at `./src`.

Then image is saved as `.tar` file with `docker save` and stored as an artefact.

Artifacts can be collected from the *CircleCI Pipeline* by running `pack` job:

```
pack:
    docker:
      - image: circleci/node:lts
    environment:
      NODE_ENV: production
      IMAGE_NAME: sdo-a1/building-on-ci
    steps:
      - checkout
      - setup_remote_docker

      - run:
          name: Build Docker image
          command: docker build -t $IMAGE_NAME:latest ./src
      
      - run:
          name: Archive Docker image
          command: docker save -o image.tar $IMAGE_NAME

      - store_artifacts:
          path: ./image.tar
```

To keep this only run on master branch, `filters` is declared in the `workflows` for `pack` job:

```
- pack:
    requires:
    - build
    filters:
    branches:
        only:
        - master
```

---

## Running integration tests

In the `src/package.json` file, a `test-integration` script is prepared to initiate the integration test with *Mocha*. Similiar to the unit tests, the command is written with the `--reporter mocha-junit-reporter` flag to output `integration_test_results.xml`. The alternate location of `integration_test_results.xml` is specified with `--reporter-options mochaFile=...`

```
"test-integration": "cross-env DB_USERNAME=postgres DB_PASSWORD=password DB_NAME=servian DB_HOSTNAME=localhost NODE_ENV=integration nyc mocha test/integration --reporter mocha-junit-reporter --reporter-options mochaFile=./test-output/integration_test_results.xml",
```

The result of the test can be found at `src/test-output`.

In the *CircleCI Pipeline*, the test is automated in `integration-tests` job. 
This job involves two images. The primary container is for running *npm* script. The second image is for standing up a *psql* database and running tests against it. The second image also has some environment variables configured, such as username, password and database name.

Artifacts can be collected from the *CircleCI Pipeline*. Below is `integration-tests` automated job:

```
integration-tests:
    docker:
      - image: circleci/node:lts
      - image: circleci/postgres:9.6-alpine
        environment:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: password
          POSTGRES_DB: servian
    steps:
      - checkout
      - install_deps

      - run:
          name: Run integration tests
          command: |
            cd src
            mkdir -p test-output
            npm run test-integration

      - store_test_results:
          path: ./src/test-output
      - store_artifacts:
          path: ./src/test-output
```

---

### Running end to end tests

In the `src/package.json` file, a `test-e2e` script is prepared to initiate the E2E test which is implemented using *QaWolf*. The command `npw qawolf test` run tests located at `.qawolf/tests` with the some of the environment variables specifed for the database:

```
"test-e2e": "cross-env NODE_ENV=e2e DB_USERNAME=postgres DB_PASSWORD=password DB_NAME=servian npx qawolf test ./tests"
```

In the *CircleCI Pipeline*, the test is automated in `e2e-tests` job. 
This job involves browser testing and a running app in the background. Therefore `-browsers` is included in the Docker primary image name. The second image is similiar to the one in integration testing setup, which is for standing up a *pqsl* database.

```
docker:
    - image: circleci/node:lts-browsers
    - image: circleci/postgres:9.6-alpine
    environment:
        POSTGRES_USER: postgres
        POSTGRES_PASSWORD: password
        POSTGRES_DB: servian
```

For this job, a few other environment variables are set up for *browser testing* to login and run the tests in *headless mode*:

```
environment:
    QAW_HEADLESS: true
    LOGIN_USERNAME: postgres
    LOGIN_PASSWORD: password
```

To keep the app running in the background, `background: true` is included when starting the app:

```
- run: 
    name: Start the app in the background
    command: |
        cd src
        npm run start    
    background: true
```

To generate *QaWolf artifacts*, a environment variable `QAW_ARTIFACT_PATH` is added with the path in running e2e tests step:

```
- run:
    name: Run e2e tests
    command: |
        cd src
        npm run test-e2e
    environment:
        QAW_ARTIFACT_PATH: /tmp/artifacts
```

Artifacts can be collected from the *CircleCI Pipeline*.
