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

**The failing curl** (command and output):

```

```

**The log line that told you what was wrong:**

```

```

**What was wrong, and the fix you applied:**

<!-- One or two sentences. Say what you changed and where you changed it. -->

**The healthy curl after the fix:**

```

```

## 5. Teardown proof

Paste the delete output, or describe the console evidence that the resources are gone.

```

```
