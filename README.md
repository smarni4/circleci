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

