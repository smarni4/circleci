# CircleCi

## Basic CircleCI config file
```yaml
# CircleCi configuration file

version: 2.1

jobs:
  #job name
  Hello-World:
    docker:   #executor
      # base ubuntu image
      - image: cimg/base:2021.04
        auth:
          username: marni4
          password: $Docker_pass
    steps:
      - run:
          name: Saying Hello
          command: |
            echo "Hello world!"
            echo 'This is the delivery pipeline'

  #Second job: Fetching the code from GitHub repo
  Fetch-Code:
    docker:
      - image: cimg/base:2021.04
        auth:
          username: marni4
          password: $Docker_pass
    steps:
      - checkout
      - run:
          name: Getting the code from the circleci repo
          command: |
            echo "Files in the circleci repo"
            ls -al

  #Third job: Running a node container
  Using-Node:
    docker:
      - image: cimg/node:17.2
        # authentication if you use your own images from your docker hub registry
        auth:
          username: marni4
          password: $Docker_pass
    steps:
      - run:
          name: Running the Node Container
          command: |
            node -v

  #Fourth job: Completes only after the manual approval
  Now-Complete:
    docker:
      - image: alpine:3.15
        auth:
          username: Marni4
          password: $Docker_pass
    resource_class: large
    steps:
      - run:
          name: Approval-Complete
          command: |
            echo "The work is now complete"

# Workflows
workflows:
  Example-Workflow:
    jobs:
      - Hello-World
      - Fetch-Code:
          requires:
            - Hello-World
      - Using-Node:
          requires:
            - Fetch-Code
      # This job waits for the manual approval in the CircleCI app
      - Hold-For-Approval:
          type: approval
          requires:
            - Using-Node
            - Fetch-Code
          # Runs this job only when the commit is performed on master branch
          filters:
            branches:
              only: master
      # This job runs only after successful completion of the Hold-For-Approval job.
      - Now-Complete:
          requires:
            - Hold-For-Approval
```

## Saving the test results
```yaml
version: 2.1

jobs:
  build:
    docker:
      - image: faucet/python3
    resource_class: large
    steps:
      - checkout
      - run:
          name: install packages
          command: |
            python -m pip install --upgrade pip
            pip install pandas numpy awscli pytest
  test:
    docker:
      -  image: faucet/python3
    resource_class: large
    steps:
      - checkout
      - run:
          name: install packages
          command: |
            python -m pip install --upgrade pip
            pip install pandas numpy awscli pytest
            mkdir -p /root/project/test_report
      - run:
          name: test packages
          command: |
            python -m pytest
      - store_test_results:
          path: ./test_report

workflows:
  test-build:
    jobs:
      - build
      - test:
          requires:
            - build
```

# [Caching and Orbs](https://circleci.com/docs/caching/)

## Caching Dependencies:
* Caches are immutable. Once saved, we cannot rewrite into the same variable.
* We can save the packages or libraries for future use to avoid the multiple downloads of the same package.
* Using caching dependencies will reduce the build time of the process.
* Creating cache is as simple as setting up `restore_cache` and `save_cache` steps along with unique keys for each
  version of the cache that we create.
* Caching files between different executors such as docker, machine, linux, or macOS can result in file permissions 
  or path errors.
  We have to be careful when caching files with different executors.
* By default, cache is stored for 15 days.
* Caching is project-specific, and there are a number of strategies to implement caching into workflows.
* To save a cache of a file or directory.
```yaml
  steps:
   - save_cache:
        key: my-cache
        paths:
          - my-file.txt
          - my-project/my-dependencies-directory
```
* To restore the cache from the saved caches.
* In the list, the first key looks for the cache corresponding to the file and saves it in the `v1-npm-deps` string.
* If this file changed in commit, CircleCi looks for the new cache key.
* 
```yaml
steps:
  - restore_cache:
      keys:
        # Checks the cache corresponding to the package-lock.json file. If the file changes after saving the cache,
        # this key will fail
        - v1-npm-deps-{{ checksum "package-lock.json" }}
        # finds the most recent cache from any branch
        - v1-npm-deps-
  - run:
      name: install additional packages
      command: npm install <package_name>
  - save_cache:
      key:
        - v1-npm-deps{{ checksum "package-lock.json" }}
      paths:
        - ~/.cache/npm
```

## Orbs
* Orbs are sharable packages of configuration elements, including jobs, commands, and executors.
* Orbs make writing and customizing CircleCI config simple.
* Orbs can find under the circleci Orbs page.
```yaml
version: 2.1
orbs:
  node: circleci/node@5.2.0
jobs:
  install-node-example:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - node/install:
          # parameters of the install job under circleci/node@5.2.0 Orb
          install-yarn: true
          node-version: '16.13'
      - run: node --version
workflows:
  test_my_app:
    jobs:
      - install-node-example

```
## Using Orbs to Install and Cache dependencies
```yaml
version: 2.1
orbs:
  node: circleci/node@5.0.2
jobs:
  build_and_test:
    docker:
      - image: 'cimg/node:16.16.0'
    steps:
      - checkout
      - node/install_packages
      - run:
          name: Run tests
          command: npm run test-ci
workflows:
  basic_workflow:
    jobs:
      - build_and_test
```
* We can create our own Orbs using the circleci cli.
* Either public or private
* Two commands `circleci orb init <path>` and `circleci orb pack <path_to_@orb.yml>`

## Artifacts
```yaml
version: 2.1

jobs:
  build:
    docker:
      - image: python:3.6.3-jessie

    working_directory: /tmp
    steps:
      - run:
          name: Creating dummy artifacts
          command: |
            echo "My first artifact file" > /tmp/artifact-1;
            mkdir /tmp/artifacts;
            echo "My second artifact file" > /tmp/artifacts/artifact-2;
      - store_artifacts:
          path: /tmp/artifact-1
          destination: artifact-file

      - store_artifacts:
          path: /tmp/artifacts

workflows:
  artifact-loading:
    jobs:
      - build
```

## Secrets Manager
**Environment Variables**
* We can secure secrets using the environment variables section in the circleci dashboard.
* Environment variables are context at job level. We can use them at job-level logics but not at work-flow or pipeline logics.

**Contexts**
* Contexts are groups of environment variables that can be secured and shared across projects.
* After a context is created, we can use the context key in the workflows section of the config.yml file to give any job
  access to the environment variables associated with the context.
```yaml
version: 2.1
orbs:
  docker_orb: circleci/docker@2.5.0

jobs:
  docker_login_check:
    executor: docker_orb/docker
    steps:
      - setup_remote_docker
      - docker_orb/check:
          docker-password: DOCKER_PASSWORD
          docker-username: DOCKER_USER

workflows:
  docker_login_check:
    jobs:
      - docker_login_check:
          context:
            - context_demo 
```

## Testing

* Circleci provides actionable information about testing jobs that help developers produce higher-quality code by 
  identifying and resolving issues.
* Circleci provides insight into why tests failed. so developers can easily the code, learn what went wrong, and resolve.

**Storing test results on circleci**
