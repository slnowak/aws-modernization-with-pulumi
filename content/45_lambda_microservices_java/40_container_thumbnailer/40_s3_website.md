+++
title = "3.5 Make a thumbnail website"
chapter = false
weight = 50
+++

The final step is to make a website to display our images. We can use S3's static content host for this.

## Step 1 &mdash; Add a bucket policy
First, we need to adjust our bucket so that its content is accessible.

Add the following to your `App.java` file:

```java
var bucketHttpAccess = new BucketPolicy("allowHttpAccess", BucketPolicyArgs.builder()
        .bucket(bucket.getId())
        .policy(Output.format("""
                {
                  "Version": "2012-10-17",
                  "Statement": [
                      {
                          "Effect": "Allow",
                          "Principal": "*",
                          "Action": "s3:GetObject",
                          "Resource": "%s/*"
                      }
                  ]
                }""", bucket.arn()))
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
import com.pulumi.aws.lambda.Permission;
import com.pulumi.aws.lambda.PermissionArgs;
import com.pulumi.aws.s3.*;
import com.pulumi.aws.s3.inputs.BucketNotificationLambdaFunctionArgs;
import com.pulumi.awsx.ecr.Image;
import com.pulumi.awsx.ecr.ImageArgs;
import com.pulumi.core.Output;
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

            var bucketHttpAccess = new BucketPolicy("allowHttpAccess", BucketPolicyArgs.builder()
                    .bucket(bucket.getId())
                    .policy(Output.format("""
                            {
                              "Version": "2012-10-17",
                              "Statement": [
                                  {
                                      "Effect": "Allow",
                                      "Principal": "*",
                                      "Action": "s3:GetObject",
                                      "Resource": "%s/*"
                                  }
                              ]
                            }""", bucket.arn()))
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
                    .build()
            );

            var lambdaFullAccess = new RolePolicyAttachment("lambdaFullAccess", RolePolicyAttachmentArgs.builder()
                    .role(role.name())
                    .policyArn(ManagedPolicy.LambdaFullAccess.getValue())
                    .build());

            var lambdaBasicExecutionRole = new RolePolicyAttachment("basicExecutionRole", RolePolicyAttachmentArgs.builder()
                    .role(role.name())
                    .policyArn(ManagedPolicy.AWSLambdaBasicExecutionRole.getValue())
                    .build());

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

## Step 2 &mdash; Make the S3 Bucket a Static Website

Now we need to update our bucket definition - we're going to set the index document so that it points to the 'cat.jpg' thumbnail we've created before.

Update your bucket resource with the following in your `App.java` file:

```java
var bucket = new Bucket("thumbnailer", BucketArgs.builder()
        .forceDestroy(true)
        .website(BucketWebsiteArgs.builder().indexDocument("cat.jpg").build())
        .build());
```

Also, make sure you export the website url:

```java
ctx.export("url", bucket.websiteEndpoint())
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
import com.pulumi.aws.s3.*;
import com.pulumi.aws.s3.inputs.BucketNotificationLambdaFunctionArgs;
import com.pulumi.aws.s3.inputs.BucketWebsiteArgs;
import com.pulumi.awsx.ecr.Image;
import com.pulumi.awsx.ecr.ImageArgs;
import com.pulumi.core.Output;
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
                    .website(BucketWebsiteArgs.builder().indexDocument("cat.jpg").build())
                    .build());

            var bucketHttpAccess = new BucketPolicy("allowHttpAccess", BucketPolicyArgs.builder()
                    .bucket(bucket.getId())
                    .policy(Output.format("""
                            {
                              "Version": "2012-10-17",
                              "Statement": [
                                  {
                                      "Effect": "Allow",
                                      "Principal": "*",
                                      "Action": "s3:GetObject",
                                      "Resource": "%s/*"
                                  }
                              ]
                            }""", bucket.arn()))
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
                    .build()
            );

            var lambdaFullAccess = new RolePolicyAttachment("lambdaFullAccess", RolePolicyAttachmentArgs.builder()
                    .role(role.name())
                    .policyArn(ManagedPolicy.LambdaFullAccess.getValue())
                    .build());

            var lambdaBasicExecutionRole = new RolePolicyAttachment("basicExecutionRole", RolePolicyAttachmentArgs.builder()
                    .role(role.name())
                    .policyArn(ManagedPolicy.AWSLambdaBasicExecutionRole.getValue())
                    .build());

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
            ctx.export("url", bucket.websiteEndpoint());
        });
    }
}
```

## Step 3 &mdash; Provision your website

At this stage we're ready to provision our infrastructure again. Run `pulumi up` and observe the resources we're going to provision:

```
Do you want to perform this update? yes
Updating (dev)

View Live: https://app.pulumi.com/slnowak/iac-workshop-thumbnailer/dev/updates/78

     Type                             Name                          Status      Info
     pulumi:pulumi:Stack              iac-workshop-thumbnailer-dev              1 message
 ~   ├─ aws:s3:Bucket                 thumbnailer                   updated     [diff: +website]
 +   ├─ aws:iam:RolePolicyAttachment  basicExecutionRole            created     
 +   ├─ aws:iam:RolePolicyAttachment  lambdaFullAccess              created     
     ├─ awsx:ecr:Image                thumbnailer                               1 warning
 +   └─ aws:s3:BucketPolicy           allowHttpAccess               created     
 
Outputs:
    bucketName: "thumbnailer-54807dd"
  + url       : "thumbnailer-54807dd.s3-website-eu-west-1.amazonaws.com"

Resources:
    + 3 created
    ~ 1 updated
    4 changes. 9 unchanged

Duration: 16s
```

Hit `yes` and provision your infrastructure. You can now visit the chosen URL to see a lovely image of a cat.

## Step 4 &mdash; Destroy Everything

Finally, destroy the resources and the stack itself:

```
pulumi destroy
pulumi stack rm
```







