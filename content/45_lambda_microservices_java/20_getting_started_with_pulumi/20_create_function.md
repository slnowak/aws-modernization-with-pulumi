+++
title = "1.3 Provisioning a Lambda Function"
chapter = false
weight = 20
+++

Now that you have a project configured to use AWS, you'll create some basic infrastructure in it. Let's create a simple lambda function.

## Step 1 &mdash; Create a Lambda Execution Role

Add a role to your Pulumi project so that your Lambda function can execute.

```java
var lambdaAssumeRolePolicy = """
        {
            "Version": "2012-10-17",
            "Statement": [{
                "Effect": "Allow",
                "Principal": {
                    "Service": "lambda.amazonaws.com"
                },
                "Action": "sts:AssumeRole"
            }]
        }""";

var lambdaRole = new Role("lambda-role", RoleArgs.builder()
        .assumeRolePolicy(lambdaAssumeRolePolicy)
        .managedPolicyArns("arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole")
        .build());
```


## Step 2 &mdash; Declare a New Lambda Function

We've defined an execution role, let's define a Lambda function that uses this execution role.

Add the following to your `App.java` file:

```
var lambdaFunction = new Function("hello-world-lambda", FunctionArgs.builder()
        .role(lambdaRole.arn())
        .handler("index.handler")
        .runtime(Runtime.NodeJS12dX)
        .code(new AssetArchive(Map.of("index.js", new StringAsset("exports.handler = (e, c, cb) => cb(null, {statusCode: 200, body: 'Hello, world!'});"))))
        .build());
```

{{% notice info %}}
The `App.java` file should now have the following contents:
{{% /notice %}}
```java
package myproject;

import com.pulumi.Pulumi;
import com.pulumi.asset.AssetArchive;
import com.pulumi.asset.StringAsset;
import com.pulumi.aws.iam.Role;
import com.pulumi.aws.iam.RoleArgs;
import com.pulumi.aws.lambda.Function;
import com.pulumi.aws.lambda.FunctionArgs;
import com.pulumi.aws.lambda.enums.Runtime;

import java.util.Map;

public class App1 {

    public static void main(String[] args) {
        Pulumi.run(ctx -> {
            var lambdaAssumeRolePolicy = """
                    {
                        "Version": "2012-10-17",
                        "Statement": [{
                            "Effect": "Allow",
                            "Principal": {
                                "Service": "lambda.amazonaws.com"
                            },
                            "Action": "sts:AssumeRole"
                        }]
                    }""";

            var lambdaRole = new Role("lambda-role", RoleArgs.builder()
                    .assumeRolePolicy(lambdaAssumeRolePolicy)
                    .managedPolicyArns("arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole")
                    .build());

            var lambdaFunction = new Function("hello-world-lambda", FunctionArgs.builder()
                    .role(lambdaRole.arn())
                    .handler("index.handler")
                    .runtime(Runtime.NodeJS12dX)
                    .code(new AssetArchive(Map.of("index.js", new StringAsset("exports.handler = (e, c, cb) => cb(null, {statusCode: 200, body: 'Hello, world!'});"))))
                    .build());
        });
    }
}
```

## Step 3 &mdash; Preview Your Changes

Now preview your changes:

```
pulumi up
```

This command evaluates your program, determines the resource updates to make, and shows you an outline of these changes:

```
Previewing update (dev)

     Type                    Name                     Plan       
 +   pulumi:pulumi:Stack     iac-workshop-lambda-dev  create     
 +   ├─ aws:iam:Role         lambda-role              create     
 +   └─ aws:lambda:Function  hello-world-lambda       create     
 
Resources:
    + 3 to create
```

This is a summary view. Select `details` to view the full set of properties:

```
Do you want to perform this update? details
+ pulumi:pulumi:Stack: (create)
    [urn=urn:pulumi:dev::iac-workshop-lambda::pulumi:pulumi:Stack::iac-workshop-lambda-dev]
    + aws:iam/role:Role: (create)
        [urn=urn:pulumi:dev::iac-workshop-lambda::aws:iam/role:Role::lambda-role]
        [provider=urn:pulumi:dev::iac-workshop-lambda::pulumi:providers:aws::default_5_3_0::04da6b54-80e4-46f7-96ec-b56ff0331ba9]
        assumeRolePolicy   : (json) {
            Statement: [
                [0]: {
                    Action   : "sts:AssumeRole"
                    Effect   : "Allow"
                    Principal: {
                        Service: "lambda.amazonaws.com"
                    }
                }
            ]
            Version  : "2012-10-17"
        }

        forceDetachPolicies: false
        managedPolicyArns  : [
            [0]: "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
        ]
        maxSessionDuration : 3600
        name               : "lambda-role-650a908"
        path               : "/"
    + aws:lambda/function:Function: (create)
        [urn=urn:pulumi:dev::iac-workshop-lambda::aws:lambda/function:Function::hello-world-lambda]
        [provider=urn:pulumi:dev::iac-workshop-lambda::pulumi:providers:aws::default_5_3_0::04da6b54-80e4-46f7-96ec-b56ff0331ba9]
        code                        : archive(assets:b898ebe) {
            "index.js": asset(text:fd60d66) {
                <contents elided>
            }
        }
        handler                     : "index.handler"
        memorySize                  : 128
        name                        : "hello-world-lambda-1cea1ae"
        packageType                 : "Zip"
        publish                     : false
        reservedConcurrentExecutions: -1
        role                        : output<string>
        runtime                     : "nodejs12.x"
        timeout                     : 3

```

The stack resource is a synthetic resource that all resources your program creates are parented to.

## Step 4 &mdash; Deploy Your Changes

Now that we've seen the full set of changes, let's deploy them. Select `yes`:

```
Updating (dev)

     Type                    Name                     Status      
 +   pulumi:pulumi:Stack     iac-workshop-lambda-dev  created     
 +   ├─ aws:iam:Role         lambda-role              created     
 +   └─ aws:lambda:Function  hello-world-lambda       created     
 
Resources:
    + 3 created

Duration: 26s
```

Now our lambda function has been created in our AWS account. Feel free to click the Permalink URL and explore; this will take you to the [Pulumi Console](https://www.pulumi.com/docs/intro/console/), which records your deployment history.

## Step 4 &mdash; Export Your Function Name

To inspect your new Lambda function, you will need its name. Pulumi records a logical name, `my-function`, however the resulting AWS lambda function resource in AWS will be autogenerated.

Programs can export variables which will be shown in the CLI and recorded for each deployment. Export your lambda function's name by adding this line to `App.java`:

```java
ctx.export("functionName", lambdaFunction.name());
```

> The difference between logical and physical names is in part due to "auto-naming" which Pulumi does to ensure side-by-side projects and zero-downtime upgrades work seamlessly. 
>It can be disabled if you wish; [read more about auto-naming here](https://www.pulumi.com/docs/intro/concepts/programming-model/#autonaming).


{{% notice info %}}
The `App.java` file should now have the following contents:
{{% /notice %}}
```java
package myproject;

import com.pulumi.Pulumi;
import com.pulumi.asset.AssetArchive;
import com.pulumi.asset.StringAsset;
import com.pulumi.aws.iam.Role;
import com.pulumi.aws.iam.RoleArgs;
import com.pulumi.aws.lambda.Function;
import com.pulumi.aws.lambda.FunctionArgs;
import com.pulumi.aws.lambda.enums.Runtime;

import java.util.Map;

public class App {

    public static void main(String[] args) {
        Pulumi.run(ctx -> {
            var lambdaAssumeRolePolicy = """
                    {
                        "Version": "2012-10-17",
                        "Statement": [{
                            "Effect": "Allow",
                            "Principal": {
                                "Service": "lambda.amazonaws.com"
                            },
                            "Action": "sts:AssumeRole"
                        }]
                    }""";

            var lambdaRole = new Role("lambda-role", RoleArgs.builder()
                    .assumeRolePolicy(lambdaAssumeRolePolicy)
                    .managedPolicyArns("arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole")
                    .build());

            var lambdaFunction = new Function("hello-world-lambda", FunctionArgs.builder()
                    .role(lambdaRole.arn())
                    .handler("index.handler")
                    .runtime(Runtime.NodeJS12dX)
                    .code(new AssetArchive(Map.of("index.js", new StringAsset("exports.handler = (e, c, cb) => cb(null, {statusCode: 200, body: 'Hello, world!'});"))))
                    .build());

            ctx.export("functionName", lambdaFunction.name());
        });
    }
}
```

Now deploy the changes:

```bash
pulumi up --yes
```

Notice a new `Outputs` section is included in the output containing the lambda function's name:

```
Previewing update (dev)

View Live: https://app.pulumi.com/slnowak/iac-workshop-lambda/dev/previews/077b7010-ea20-44ff-a9fc-a84a310c0198

     Type                 Name                     Plan     
     pulumi:pulumi:Stack  iac-workshop-lambda-dev           
 
Outputs:
  + functionName: "hello-world-lambda-f42ada6"

Resources:
    3 unchanged

Updating (dev)

View Live: https://app.pulumi.com/slnowak/iac-workshop-lambda/dev/updates/5

     Type                 Name                     Status     
     pulumi:pulumi:Stack  iac-workshop-lambda-dev             
 
Outputs:
  + functionName: "hello-world-lambda-f42ada6"

Resources:
    3 unchanged

Duration: 3s
```

## Step 5 &mdash; Trigger your function

We can now trigger out function using the AWS CLI

```
aws lambda invoke --function-name $(pulumi stack output functionName) /tmp/out
```

This should tell us that the function executed successfully. We could go and look at the cloudwatch log group, but there's an easier way we'll be able to see in the next step.

## Step 6 &mdash; Destroy Everything

Finally, destroy the resources and the stack itself:

```
pulumi destroy
pulumi stack rm
```
