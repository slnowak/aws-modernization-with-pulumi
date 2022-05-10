+++
title = "3.4 Define Lambda Container Function"
chapter = false
weight = 20
+++

In the previous step, we built and pushed a Lambda container to an ECR repository. Now let's define a lambda function which runs this container.

## Step 1 &mdash; Declare an S3 Bucket and IAM Role
We need to create an S3 bucket. We also need to define an IAM Role for our lambda function - granting it necessary permissions and s3 bucket access.

Add the following to your `App.java` file:

```java
var bucket = new Bucket("thumbnailer", BucketArgs.builder()
        .forceDestroy(true)
        .build());

var role = new Role("thumbnailerRole", RoleArgs.builder()
        .assumeRolePolicy("""
                {
                  "Version": "2012-10-17",
                  "Statement": [
                    {
                      "Effect": "Allow",
                      "Principal": {
                        "Service": "lambda.amazonaws.com"
                      },
                      "Action": "sts:AssumeRole"
                    }
                  ]
                }""")
        .managedPolicyArns(ManagedPolicy.AWSLambdaBasicExecutionRole.getValue(), ManagedPolicy.LambdaFullAccess.getValue())
        .build()
);

var s3AccessPolicy = new Policy("lambdaS3Access", PolicyArgs.builder()
        .policy(bucket.arn().applyValue(buketArn -> """
                {
                  "Version": "2012-10-17",
                  "Statement": [
                    {
                      "Effect": "Allow",
                      "Action": "s3:*",
                      "Resource": ["%s", "%s/*"]
                    }
                  ]
                }""".formatted(buketArn, buketArn)))
        .build());

var s3Access = new RolePolicyAttachment("lambdaS3Access", RolePolicyAttachmentArgs.builder()
        .role(role.name())
        .policyArn(s3AccessPolicy.arn())
        .build());
```

{{% notice info %}}
The `App.java` file should now have the following contents:
{{% /notice %}}
```java
package myproject;

import com.pulumi.Pulumi;
import com.pulumi.aws.ecr.Repository;
import com.pulumi.aws.iam.*;
import com.pulumi.aws.iam.enums.ManagedPolicy;
import com.pulumi.aws.s3.Bucket;
import com.pulumi.aws.s3.BucketArgs;
import com.pulumi.awsx.ecr.Image;
import com.pulumi.awsx.ecr.ImageArgs;

public class App {

    public static void main(String[] args) {
        Pulumi.run(ctx -> {
            var repo = new Repository("thumbnailer");
            var image = new Image("thumbnailer", ImageArgs.builder()
                    .repositoryUrl(repo.repositoryUrl())
                    .path("./app")
                    .build());

            var bucket = new Bucket("thumbnailer", BucketArgs.builder()
                    .forceDestroy(true)
                    .build());

            var role = new Role("thumbnailerRole", RoleArgs.builder()
                    .assumeRolePolicy("""
                            {
                              "Version": "2012-10-17",
                              "Statement": [
                                {
                                  "Effect": "Allow",
                                  "Principal": {
                                    "Service": "lambda.amazonaws.com"
                                  },
                                  "Action": "sts:AssumeRole"
                                }
                              ]
                            }""")
                    .managedPolicyArns(ManagedPolicy.AWSLambdaBasicExecutionRole.getValue(), ManagedPolicy.LambdaFullAccess.getValue())
                    .build()
            );

            var s3AccessPolicy = new Policy("lambdaS3Access", PolicyArgs.builder()
                    .policy(bucket.arn().applyValue(buketArn -> """
                            {
                              "Version": "2012-10-17",
                              "Statement": [
                                {
                                  "Effect": "Allow",
                                  "Action": "s3:*",
                                  "Resource": ["%s", "%s/*"]
                                }
                              ]
                            }""".formatted(buketArn, buketArn)))
                    .build());

            var s3Access = new RolePolicyAttachment("lambdaS3Access", RolePolicyAttachmentArgs.builder()
                    .role(role.name())
                    .policyArn(s3AccessPolicy.arn())
                    .build());
        });
    }
}
```

## Step 2 &mdash; Add the Lambda Function

We've created the IAM role we need, let's now define our Lambda function.

Add the following to your `App.java` file:

```java
var thumbnailer = new Function("thumbnailer", FunctionArgs.builder()
        .packageType("Image")
        .imageUri(image.imageUri())
        .role(role.arn())
        .timeout(900)
        .build());
```

{{% notice info %}}
The `App.java` file should now have the following contents:
{{% /notice %}}
```java
package myproject;

import com.pulumi.Pulumi;
import com.pulumi.aws.ecr.Repository;
import com.pulumi.aws.iam.*;
import com.pulumi.aws.iam.enums.ManagedPolicy;
import com.pulumi.aws.lambda.Function;
import com.pulumi.aws.lambda.FunctionArgs;
import com.pulumi.aws.s3.Bucket;
import com.pulumi.aws.s3.BucketArgs;
import com.pulumi.awsx.ecr.Image;
import com.pulumi.awsx.ecr.ImageArgs;

public class App {

    public static void main(String[] args) {
        Pulumi.run(ctx -> {
            var repo = new Repository("thumbnailer");
            var image = new Image("thumbnailer", ImageArgs.builder()
                    .repositoryUrl(repo.repositoryUrl())
                    .path("./app")
                    .build());

            var bucket = new Bucket("thumbnailer", BucketArgs.builder()
                    .forceDestroy(true)
                    .build());

            var role = new Role("thumbnailerRole", RoleArgs.builder()
                    .assumeRolePolicy("""
                            {
                              "Version": "2012-10-17",
                              "Statement": [
                                {
                                  "Effect": "Allow",
                                  "Principal": {
                                    "Service": "lambda.amazonaws.com"
                                  },
                                  "Action": "sts:AssumeRole"
                                }
                              ]
                            }""")
                    .managedPolicyArns(ManagedPolicy.AWSLambdaBasicExecutionRole.getValue(), ManagedPolicy.LambdaFullAccess.getValue())
                    .build()
            );

            var s3AccessPolicy = new Policy("lambdaS3Access", PolicyArgs.builder()
                    .policy(bucket.arn().applyValue(buketArn -> """
                            {
                              "Version": "2012-10-17",
                              "Statement": [
                                {
                                  "Effect": "Allow",
                                  "Action": "s3:*",
                                  "Resource": ["%s", "%s/*"]
                                }
                              ]
                            }""".formatted(buketArn, buketArn)))
                    .build());

            var s3Access = new RolePolicyAttachment("lambdaS3Access", RolePolicyAttachmentArgs.builder()
                    .role(role.name())
                    .policyArn(s3AccessPolicy.arn())
                    .build());

            var thumbnailer = new Function("thumbnailer", FunctionArgs.builder()
                    .packageType("Image")
                    .imageUri(image.imageUri())
                    .role(role.arn())
                    .timeout(900)
                    .build());

            ctx.export("bucketName", bucket.getId());
        });
    }
}
```

Notice we're referencing the image we built earlier and passing it to the function.

## Step 3 &mdash; Add the Lambda Event

We want our Lambda function to trigger when we upload files to it. We can do this inline within our Pulumi program.

Before we instruct AWS to trigger our lambda when we upload files to a bucket, we actually need to modify our bucket's policy a bit, so that it is allowed to execute a function we've created before. 
To do this, add the following snippet to your `App.java` file:
```java
var bucketExecutionAllowance = new Permission("bucketExecutionAllowance", PermissionArgs.builder()
        .function(thumbnailer.arn())
        .action("lambda:InvokeFunction")
        .principal("s3.amazonaws.com")
        .sourceArn(bucket.arn())
        .build());
```
Notice, that we've referenced both the lambda function and the bucket itself.

Now, let's actually define the event notification trigger:
Add the following to your `App.java` file:

```java
// When a new video is uploaded, run the FFMPEG task on the video file.
// Use the time index specified in the filename (e.g. cat_00-01.mp4 uses timestamp 00:01)
var runFfmpeg = new BucketNotification("onNewVideo",
                    BucketNotificationArgs.builder()
                            .bucket(bucket.getId())
                            .lambdaFunctions(
                                    BucketNotificationLambdaFunctionArgs.builder()
                                            .events("s3:ObjectCreated:*")
                                            .filterSuffix(".mp4")
                                            .lambdaFunctionArn(thumbnailer.arn())
                                            .build()
                            )
                            .build(),

                    CustomResourceOptions.builder()
                            .dependsOn(bucketExecutionAllowance)
                            .build()
            );
```
{{% notice info %}}
The `App.java` file should now have the following contents:
{{% /notice %}}
```java
package myproject;

import com.pulumi.Pulumi;
import com.pulumi.aws.ecr.Repository;
import com.pulumi.aws.iam.*;
import com.pulumi.aws.iam.enums.ManagedPolicy;
import com.pulumi.aws.lambda.Function;
import com.pulumi.aws.lambda.FunctionArgs;
import com.pulumi.aws.lambda.Permission;
import com.pulumi.aws.lambda.PermissionArgs;
import com.pulumi.aws.s3.Bucket;
import com.pulumi.aws.s3.BucketArgs;
import com.pulumi.aws.s3.BucketNotification;
import com.pulumi.aws.s3.BucketNotificationArgs;
import com.pulumi.aws.s3.inputs.BucketNotificationLambdaFunctionArgs;
import com.pulumi.awsx.ecr.Image;
import com.pulumi.awsx.ecr.ImageArgs;
import com.pulumi.resources.CustomResourceOptions;

public class App {

    public static void main(String[] args) {
        Pulumi.run(ctx -> {
            var repo = new Repository("thumbnailer");
            var image = new Image("thumbnailer", ImageArgs.builder()
                    .repositoryUrl(repo.repositoryUrl())
                    .path("./app")
                    .build());

            var bucket = new Bucket("thumbnailer", BucketArgs.builder()
                    .forceDestroy(true)
                    .build());

            var role = new Role("thumbnailerRole", RoleArgs.builder()
                    .assumeRolePolicy("""
                            {
                              "Version": "2012-10-17",
                              "Statement": [
                                {
                                  "Effect": "Allow",
                                  "Principal": {
                                    "Service": "lambda.amazonaws.com"
                                  },
                                  "Action": "sts:AssumeRole"
                                }
                              ]
                            }""")
                    .managedPolicyArns(ManagedPolicy.AWSLambdaBasicExecutionRole.getValue(), ManagedPolicy.LambdaFullAccess.getValue())
                    .build()
            );

            var s3AccessPolicy = new Policy("lambdaS3Access", PolicyArgs.builder()
                    .policy(bucket.arn().applyValue(buketArn -> """
                            {
                              "Version": "2012-10-17",
                              "Statement": [
                                {
                                  "Effect": "Allow",
                                  "Action": "s3:*",
                                  "Resource": ["%s", "%s/*"]
                                }
                              ]
                            }""".formatted(buketArn, buketArn)))
                    .build());

            var s3Access = new RolePolicyAttachment("lambdaS3Access", RolePolicyAttachmentArgs.builder()
                    .role(role.name())
                    .policyArn(s3AccessPolicy.arn())
                    .build());

            var thumbnailer = new Function("thumbnailer", FunctionArgs.builder()
                    .packageType("Image")
                    .imageUri(image.imageUri())
                    .role(role.arn())
                    .timeout(900)
                    .build());

            var bucketExecutionAllowance = new Permission("bucketExecutionAllowance", PermissionArgs.builder()
                    .function(thumbnailer.arn())
                    .action("lambda:InvokeFunction")
                    .principal("s3.amazonaws.com")
                    .sourceArn(bucket.arn())
                    .build());

            var runFfmpeg = new BucketNotification("onNewVideo",
                    BucketNotificationArgs.builder()
                            .bucket(bucket.getId())
                            .lambdaFunctions(
                                    BucketNotificationLambdaFunctionArgs.builder()
                                            .events("s3:ObjectCreated:*")
                                            .filterSuffix(".mp4")
                                            .lambdaFunctionArn(thumbnailer.arn())
                                            .build()
                            )
                            .build(),

                    CustomResourceOptions.builder()
                            .dependsOn(bucketExecutionAllowance)
                            .build()
            );
        });
    }
}
```

We've just added an event notification trigger. It waits for videos to be uploaded to our S3 bucket with the suffix `.mp4` and if that happens, it triggers the thumbnailer lambda container.

## Step 4 &mdash; Export the Bucket name

We need to know where to upload our videos to, so let's export our bucket name from our Pulumi program.

```java
ctx.export("bucketName", bucket.getId());
```

{{% notice info %}}
The `App.java` file should now have the following contents:
{{% /notice %}}
```java
package myproject;

import com.pulumi.Pulumi;
import com.pulumi.aws.ecr.Repository;
import com.pulumi.aws.iam.*;
import com.pulumi.aws.iam.enums.ManagedPolicy;
import com.pulumi.aws.lambda.Function;
import com.pulumi.aws.lambda.FunctionArgs;
import com.pulumi.aws.lambda.Permission;
import com.pulumi.aws.lambda.PermissionArgs;
import com.pulumi.aws.s3.Bucket;
import com.pulumi.aws.s3.BucketArgs;
import com.pulumi.aws.s3.BucketNotification;
import com.pulumi.aws.s3.BucketNotificationArgs;
import com.pulumi.aws.s3.inputs.BucketNotificationLambdaFunctionArgs;
import com.pulumi.awsx.ecr.Image;
import com.pulumi.awsx.ecr.ImageArgs;
import com.pulumi.resources.CustomResourceOptions;

public class App {

    public static void main(String[] args) {
        Pulumi.run(ctx -> {
            var repo = new Repository("thumbnailer");
            var image = new Image("thumbnailer", ImageArgs.builder()
                    .repositoryUrl(repo.repositoryUrl())
                    .path("./app")
                    .build());

            var bucket = new Bucket("thumbnailer", BucketArgs.builder()
                    .forceDestroy(true)
                    .build());

            var role = new Role("thumbnailerRole", RoleArgs.builder()
                    .assumeRolePolicy("""
                            {
                              "Version": "2012-10-17",
                              "Statement": [
                                {
                                  "Effect": "Allow",
                                  "Principal": {
                                    "Service": "lambda.amazonaws.com"
                                  },
                                  "Action": "sts:AssumeRole"
                                }
                              ]
                            }""")
                    .managedPolicyArns(ManagedPolicy.AWSLambdaBasicExecutionRole.getValue(), ManagedPolicy.LambdaFullAccess.getValue())
                    .build()
            );

            var s3AccessPolicy = new Policy("lambdaS3Access", PolicyArgs.builder()
                    .policy(bucket.arn().applyValue(buketArn -> """
                            {
                              "Version": "2012-10-17",
                              "Statement": [
                                {
                                  "Effect": "Allow",
                                  "Action": "s3:*",
                                  "Resource": ["%s", "%s/*"]
                                }
                              ]
                            }""".formatted(buketArn, buketArn)))
                    .build());

            var s3Access = new RolePolicyAttachment("lambdaS3Access", RolePolicyAttachmentArgs.builder()
                    .role(role.name())
                    .policyArn(s3AccessPolicy.arn())
                    .build());

            var thumbnailer = new Function("thumbnailer", FunctionArgs.builder()
                    .packageType("Image")
                    .imageUri(image.imageUri())
                    .role(role.arn())
                    .timeout(900)
                    .build());

            var bucketExecutionAllowance = new Permission("bucketExecutionAllowance", PermissionArgs.builder()
                    .function(thumbnailer.arn())
                    .action("lambda:InvokeFunction")
                    .principal("s3.amazonaws.com")
                    .sourceArn(bucket.arn())
                    .build());

            var runFfmpeg = new BucketNotification("onNewVideo",
                    BucketNotificationArgs.builder()
                            .bucket(bucket.getId())
                            .lambdaFunctions(
                                    BucketNotificationLambdaFunctionArgs.builder()
                                            .events("s3:ObjectCreated:*")
                                            .filterSuffix(".mp4")
                                            .lambdaFunctionArn(thumbnailer.arn())
                                            .build()
                            )
                            .build(),

                    CustomResourceOptions.builder()
                            .dependsOn(bucketExecutionAllowance)
                            .build()
            );

            ctx.export("bucketName", bucket.getId());
        });
    }
}
```

## Step 4 &mdash; Provision your function

At this stage we're ready to provision our infrastructure again. Run `pulumi up` and observe the resources we're going to provision:

```
Do you want to perform this update? yes
Updating (dev)

View Live: https://app.pulumi.com/slnowak/iac-workshop-thumbnailer/dev/updates/77

     Type                             Name                          Status      Info
     pulumi:pulumi:Stack              iac-workshop-thumbnailer-dev              1 message
 +   ├─ aws:s3:Bucket                 thumbnailer                   created     
 +   ├─ aws:iam:Role                  thumbnailerRole               created     
     ├─ awsx:ecr:Image                thumbnailer                               1 warning
 +   ├─ aws:iam:Policy                lambdaS3Access                created     
 +   ├─ aws:iam:RolePolicyAttachment  lambdaS3Access                created     
 +   ├─ aws:lambda:Function           thumbnailer                   created     
 +   ├─ aws:lambda:Permission         bucketExecutionAllowance      created     
 +   └─ aws:s3:BucketNotification     onNewVideo                    created    
```

Hit `yes` and provision your infrastructure. You should see your bucket name displayed as an Output:

```
Outputs:
  + bucketName: "thumbnailer-54807dd"

Resources:
    + 7 created
    3 unchanged

Duration: 57s
```

## Step 5 &mdash; Upload an example video

{{% notice info %}}
If you need an example video, try the one in our examples repo:
https://github.com/pulumi/examples/tree/master/aws-ts-lambda-thumbnailer/sample
{{% /notice %}}

Let's trigger our function by uploading an `.mp4` video to the bucket:

```
aws s3 cp ./sample/cat.mp4 s3://$(pulumi stack output bucketName)/cat_00-01.mp4                                                                                                                    
```

You should see the file get uploaded to s3:

```
upload: sample/cat.mp4 to s3://thumbnailer-f91a64e/cat_00-01.mp4
```

You can view the logs from your function to see if the functions were triggered:


```
2022-05-19T11:55:49.414+02:00	START RequestId: 0dfc2145-67a6-424b-a894-9434871d9d11 Version: $LATEST

2022-05-19T11:55:49.430+02:00	2022-05-19T09:55:49.428Z 0dfc2145-67a6-424b-a894-9434871d9d11 INFO Video handler called

2022-05-19T11:55:49.430+02:00	2022-05-19T09:55:49.430Z 0dfc2145-67a6-424b-a894-9434871d9d11 INFO aws s3 cp s3://thumbnailer-54807dd/cat_00-01.mp4 /tmp/cat_00-01.mp4

2022-05-19T11:56:03.929+02:00	Completed 256.0 KiB/666.5 KiB (1.2 MiB/s) with 1 file(s) remaining Completed 512.0 KiB/666.5 KiB (2.1 MiB/s) with 1 file(s) remaining Completed 666.5 KiB/666.5 KiB (2.7 MiB/s) with 1 file(s) remaining download: s3://thumbnailer-54807dd/cat_00-01.mp4 to ../../tmp/cat_00-01.mp4

2022-05-19T11:56:06.349+02:00	2022-05-19T09:56:06.349Z 0dfc2145-67a6-424b-a894-9434871d9d11 INFO ffmpeg -v error -i /tmp/cat_00-01.mp4 -ss 00:01 -vframes 1 -f image2 -an -y /tmp/cat.jpg

2022-05-19T11:56:12.039+02:00	2022-05-19T09:56:12.039Z 0dfc2145-67a6-424b-a894-9434871d9d11 INFO aws s3 cp /tmp/cat.jpg s3://thumbnailer-54807dd/cat.jpg

2022-05-19T11:56:22.248+02:00	Completed 86.6 KiB/86.6 KiB (246.7 KiB/s) with 1 file(s) remaining upload: ../../tmp/cat.jpg to s3://thumbnailer-54807dd/cat.jpg

2022-05-19T11:56:24.809+02:00	2022-05-19T09:56:24.809Z 0dfc2145-67a6-424b-a894-9434871d9d11 INFO *** New thumbnail: file cat_00-01.mp4 was saved at 2022-05-19T09:55:47.265Z.

2022-05-19T11:56:24.831+02:00	END RequestId: 0dfc2145-67a6-424b-a894-9434871d9d11

2022-05-19T11:56:24.831+02:00	REPORT RequestId: 0dfc2145-67a6-424b-a894-9434871d9d11 Duration: 35415.30 ms Billed Duration: 35853 ms Memory Size: 128 MB Max Memory Used: 128 MB Init Duration: 436.77 ms
```

You can see that the thumbnailer generated a `cat.jpg` for our video! Let's download it:

```
aws s3 cp s3://$(pulumi stack output bucketName)/cat.jpg .
```
