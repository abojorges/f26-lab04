# Deployment Evidence

Fill this in as you go. Paste real output, not descriptions of output. A TA reads this
file with you at recitation.

## 1. Deployed URL and instance id

<!-- The ServiceUrl and InstanceId outputs. Paste both here every time
describe-stacks prints them, for the healthy deploy and for scenario 2. Both
change on every recreate, and you will need them for curls and sessions. -->

**Milestone 1 (healthy deploy, `params-healthy.json`):**

```
$ aws cloudformation describe-stacks --stack-name lab04-service \
    --query "Stacks[0].Outputs[].[OutputKey,OutputValue]" --output table
-------------------------------------------------------------------------
|                            DescribeStacks                             |
+------------+----------------------------------------------------------+
|  InstanceId|  i-02f2f2706370f7d57                                     |
|  ServiceUrl|  http://ec2-54-84-182-230.compute-1.amazonaws.com:8080   |
+------------+----------------------------------------------------------+
```

**Milestone 2, broken deploy (`params-scenario2.json`):**

```
-------------------------------------------------------------------------
|                            DescribeStacks                             |
+------------+----------------------------------------------------------+
|  InstanceId|  i-0ec2db2b630d5e0cd                                     |
|  ServiceUrl|  http://ec2-54-227-63-155.compute-1.amazonaws.com:8080   |
+------------+----------------------------------------------------------+
```

**Milestone 2, healthy redeploy (`params-healthy.json`):**

```
-------------------------------------------------------------------------
|                            DescribeStacks                             |
+------------+----------------------------------------------------------+
|  InstanceId|  i-063338c5bdcb666c3                                     |
|  ServiceUrl|  http://ec2-54-221-87-226.compute-1.amazonaws.com:8080   |
+------------+----------------------------------------------------------+
```

## 2. External health check

Run the check from your own machine, not from the instance. Paste the command and the
response.

```
$ curl http://ec2-54-84-182-230.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 3. What the template created

Three or four sentences, your own words. What compute, what network access, and what
glue made the service start.

For compute, the template creates one `t3.micro` EC2 instance running Amazon Linux 2023,
with the Learner Lab's `LabInstanceProfile` (so SSM sessions work) and the `vockey` key
pair (for SSH fallback). For network access, it creates a security group that allows
inbound TCP on the service port (8080) and on SSH (22) from anywhere, and blocks all
other inbound traffic. The glue is the instance's UserData script, which runs on first
boot: it installs and starts Docker, schedules an automatic shutdown after 4 hours, and
runs the `ghcr.io/cmu-17-214/lab04-service` container with host port 8080 forwarded to
container port 8080 and the `PORT` environment variable telling the app which port to
listen on. The stack outputs the public URL and the instance id.

## 4. Scenario 2 diagnosis

**The failing curl** (command and output), run from my machine more than 3 minutes
after CREATE_COMPLETE, so well past the warmup. It kept failing on every retry:

```
$ curl http://ec2-54-227-63-155.compute-1.amazonaws.com:8080/api/health
curl: (28) Failed to connect to ec2-54-227-63-155.compute-1.amazonaws.com port 8080 after 75029 ms: Couldn't connect to server
```

The instance itself was fine: `running`, status checks `ok`, SSM agent `Online`, and
port 22 on the same host accepted connections. Only the service port failed.

**The log line that told you what was wrong** (from an SSM session on
`i-0ec2db2b630d5e0cd`):

```
$ aws ssm start-session --target i-0ec2db2b630d5e0cd \
    --document-name AWS-StartInteractiveCommand \
    --parameters '{"command":["sudo docker ps; echo ---; sudo docker logs lab04-service; echo ---; sudo ss -ltnp | grep -E \"8080|9090\""]}'
Starting session with SessionId: user5423213=Antonio_Bojorges-q9v8ayivb79dv7kyqayou9iiqe
CONTAINER ID   IMAGE                                     COMMAND                  CREATED         STATUS         PORTS                                       NAMES
d7f3daa3290f   ghcr.io/cmu-17-214/lab04-service:latest   "/__cacert_entrypoin…"   5 minutes ago   Up 5 minutes   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   lab04-service
---
lab04-service listening on 9090
---
LISTEN 0      4096         0.0.0.0:8080       0.0.0.0:*    users:(("docker-proxy",pid=15614,fd=5))
LISTEN 0      4096            [::]:8080          [::]:*    users:(("docker-proxy",pid=15670,fd=5))
Cannot perform start session: EOF
```

**What was wrong, and the fix you applied:**

The app inside the container was listening on port 9090, but the Docker port mapping
sent traffic to container port 8080, where nothing was listening. `docker ps` shows the
mapping `8080->8080`, and `docker logs` shows `listening on 9090` (the local warm-up and
healthy deploys print `listening on 8080`). The cause was the stack parameter
`PortOverride=9090` in `params-scenario2.json`. The template's UserData passes it to the
container as `-e PORT=9090`, but the `-p ${ServicePort}:${ServicePort}` mapping and the
security group still use `ServicePort=8080`. I fixed it through the infrastructure,
without touching the running container: I deleted the broken stack and recreated it with
`params-healthy.json`, where `PortOverride` is empty. The container then gets `PORT=8080`
and its log reads `lab04-service listening on 8080`.

**The healthy curl after the fix:**

```
$ curl http://ec2-54-221-87-226.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 5. Teardown proof

Paste the delete output, or describe the console evidence that the resources are gone.

The last stack (the healthy redeploy from milestone 2) was deleted right after its
healthy curl. The teardown commands below were then run again for milestone 3. They
succeed without error when there is nothing left to delete. The checks after them show
that everything is gone.

```
$ aws cloudformation delete-stack --stack-name lab04-service
(exit 0)
$ aws cloudformation wait stack-delete-complete --stack-name lab04-service
(exit 0)

$ aws cloudformation describe-stacks --stack-name lab04-service
An error occurred (ValidationError) when calling the DescribeStacks operation: Stack with id lab04-service does not exist
```

Stack history: all three stacks from this lab (milestone 1, the milestone 2 broken
deploy, and the milestone 2 healthy redeploy) ended in `DELETE_COMPLETE`. Times are UTC.

```
$ aws cloudformation list-stacks \
    --query "StackSummaries[?StackName=='lab04-service'].[StackStatus,CreationTime,DeletionTime]" \
    --output table
---------------------------------------------------------------------------------------------
|                                        ListStacks                                         |
+-----------------+------------------------------------+------------------------------------+
|  DELETE_COMPLETE|  2026-09-18T03:08:58.863000+00:00  |  2026-09-18T03:11:00.722000+00:00  |
|  DELETE_COMPLETE|  2026-09-18T03:01:01.330000+00:00  |  2026-09-18T03:07:56.036000+00:00  |
|  DELETE_COMPLETE|  2026-09-18T02:48:55.367000+00:00  |  2026-09-18T02:50:46.029000+00:00  |
+-----------------+------------------------------------+------------------------------------+
```

Every instance the stacks created is terminated, no instance of any kind is left
running or stopped, and the stack's security group is gone:

```
$ aws ec2 describe-instances --filters Name=tag:Name,Values=lab04-service \
    --query "Reservations[].Instances[].[InstanceId,State.Name]" --output table
---------------------------------------
|          DescribeInstances          |
+----------------------+--------------+
|  i-0ec2db2b630d5e0cd |  terminated  |
|  i-02f2f2706370f7d57 |  terminated  |
|  i-063338c5bdcb666c3 |  terminated  |
+----------------------+--------------+

$ aws ec2 describe-instances --filters Name=instance-state-name,Values=pending,running,stopping,stopped \
    --query "Reservations[].Instances[].InstanceId" --output text
(no output: no live instances)

$ aws ec2 describe-security-groups --filters Name=tag:Name,Values=lab04-service \
    --query "SecurityGroups[].GroupId" --output text
(no output: no lab04 security groups)
```
