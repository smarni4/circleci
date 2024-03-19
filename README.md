# CircleCi

## Basic CircleCI config file
```yaml
# CircleCi configuration file

version: 2.1

jobs:
  #job name
  Hello-World:
    docker:
      # base ubuntu image
      - image: cimg/base:2021.04
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
    steps:
      - run:
          name: Running the Node Container
          command: |
            node -v

  #Fourth job: Completes only after the manual approval
  Now-Complete:
    docker:
      - image: alpine:3.15
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
      # This job runs only after successful completion of the Hold-For-Approval job.
      - Now-Complete:
          requires:
            - Hold-For-Approval
```

