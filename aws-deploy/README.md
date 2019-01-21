These instructions will:
  1. create an ECS cluster (using Fargate, not EC2)
  1. deploy this docker image as a scheduled task
  1. configure a cron schedule to run the task

Fargate is fairly expensive but as we're only running the task infrequently and for a very short time,
the cost should be minimal.

## Steps

  1. optionally, define an AWS CLI profile and region to run in
      ```bash
      export AWS_PROFILE=default        # change me if needed
      export AWS_REGION=ap-southeast-2  # change me if needed
      ```

  1. create a new ECS cluster. You only need one cluster, you can run backups for staging and prod on the same cluster.
      ```bash
      Z_CLUSTER_NAME=oeh-fargate-cluster # optionally change me
      aws ecs create-cluster \
        --cluster-name=$Z_CLUSTER_NAME
      ```

  1. create an env var of the `clusterArn` that was returned, we'll need that later
      ```bash
      export Z_CLUSTER_ARN=arn:aws:ecs:ap-southeast-2:123456789123:cluster/oeh-fargate-cluster
      ```

  1. create a log group for the ECS task to write to
      ```bash
      export Z_STAGE=staging # change me to 'prod' if needed
      export Z_LOG_GROUP_NAME=/ecs/mongodb-backup-s3-logs-$Z_STAGE # we use this later too

      aws logs create-log-group \
        --log-group-name=$Z_LOG_GROUP_NAME
      ```

  1. create an IAM policy and user that this container can run as
      ```bash
      export Z_S3_BUCKET=oeh-photopoints-db-backup-$Z_STAGE
      export Z_USER_NAME=mongodb-backup-s3-${Z_STAGE}-user

      ./create-iam-policy-and-user.sh
      ```

  1. grab the access keys from the output of the previous command and export them as env vars
      ```bash
      export Z_AWS_ACCESS_KEY_ID=change-me      # TODO change me
      export Z_AWS_SECRET_ACCESS_KEY=change-me  # TODO change me
      ```

  1. get the ARN for the ECS task execution role. If you don't have this role already, [create it](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html)
      ```bash
      export Z_EXECUTION_ROLE_ARN=$(aws iam list-roles --query "Roles[?RoleName==\`ecsTaskExecutionRole\`].[Arn]" --output=text) && \
      if [ -z "$Z_EXECUTION_ROLE_ARN" ]; then echo "[ERROR] no role ARN found, you need to create one and re-run this command"; \
      else echo "[INFO] role ARN found, carry on"; fi
      ```

  1. create a new task definition/update (create new revision) of existing definition
      ```bash
      export Z_MONGODB_HOST=OEHCluster-shard-0/oehcluster-shard-00-00-no9bo.mongodb.net,oehcluster-shard-00-01-no9bo.mongodb.net,oehcluster-shard-00-02-no9bo.mongodb.net                    # TODO change me
      export Z_MONGODB_DB=oehphotopoints-staging # TODO change me
      export Z_MONGODB_USER=someuser             # TODO change me
      export Z_MONGODB_PASS=somepassword         # TODO change me

      ./create-ecs-task-def.sh
      ```

  1. create an env var from the newly created task ARN, we'll need that later
      ```bash
      export Z_TASK_DEF_ARN=$(aws ecs list-task-definitions --query="taskDefinitionArns[?contains(@, 'mongodb-backup-s3-task') == \`true\`] | [0]" --output=text) && \
      if [ -z "$Z_TASK_DEF_ARN" ]; then echo "[ERROR] no task ARN found, you did the previous command work?"; \
      else echo "[INFO] task ARN found, carry on"; fi
      ```

  1. create a schedule (that we'll use to run the task)
      ```bash
      export Z_RULE_NAME=MongoBackup-$Z_STAGE
      aws events put-rule \
        --schedule-expression="cron(0 20 * * ? *)" \
        --name=$Z_RULE_NAME
      ```

  1. get the ARN of the `ecsEventsRole` IAM role that was created for us when we registered the task. If you don't already have this role, [create it](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/CWE_IAM_role.html)
      ```bash
      export Z_ROLE_ARN=$(aws iam list-roles --query "Roles[?RoleName==\`ecsEventsRole\`].[Arn]" --output=text) && \
      if [ -z "$Z_ROLE_ARN" ]; then echo "[ERROR] no role ARN found, you need to create one and re-run this command"; \
      else echo "[INFO] role ARN found, carry on"; fi
      ```

  1. get the ID of the network security group to deploy into. Use the `GroupId` field of your chosen group
      ```bash
      aws ec2 describe-security-groups --query="SecurityGroups[*].{GroupId: GroupId, GroupName: GroupName}"
      # find the GroupID field and export it as an env var
      export Z_SEC_GROUP=sg-0cffbb75
      ```

  1. get the ID of a network subnet to deploy into. Use the `SubnetId` field
      ```bash
      aws ec2 describe-subnets
      # find the SubnetId field and export it as an env var
      export Z_SUBNET=subnet-86d610de
      ```

  1. tie the task definition to the event trigger.
      ```bash
      # uses all the env vars we exported earlier
      ./create-event-schedule.sh
      ```

  1. if you see output that looks like the following, you're done
      ```bash
      {
          "FailedEntryCount": 0,
          "FailedEntries": []
      }
      ```
