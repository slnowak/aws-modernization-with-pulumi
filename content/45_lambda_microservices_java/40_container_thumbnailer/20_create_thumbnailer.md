+++
title = "3.3 Thumbnailer Container"
chapter = false
weight = 20
+++

Now that you have a project configured to use AWS, we can create the thumbnailer. We'll use the AWS Lambda Containers for this.

## Step 1 &mdash; Create a Docker Image

Create a new directory within your Pulumi project called `app`

```bash
mkdir app
```

Next, create a `Dockerfile` within the `app` directory and use it to build ffmpeg into the image:

```dockerfile
FROM amazon/aws-lambda-nodejs:12
ARG FUNCTION_DIR="/var/task"

# Install tar and xz
RUN yum install tar xz unzip -y

# Install awscli
RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" -s
RUN unzip -q awscliv2.zip
RUN ./aws/install

# Install ffmpeg
RUN curl https://johnvansickle.com/ffmpeg/releases/ffmpeg-release-amd64-static.tar.xz -o ffmpeg.tar.xz -s
RUN tar -xf ffmpeg.tar.xz
RUN mv ffmpeg-5.0.1-amd64-static/ffmpeg /usr/bin

# Create function directory
RUN mkdir -p ${FUNCTION_DIR}

# Copy handler function and package.json
COPY index.js ${FUNCTION_DIR}

# Set the CMD to your handler (could also be done as a parameter override outside of the Dockerfile)
CMD [ "index.handler" ]
```

You'll notice you're specifying an `index.handler` as the command, but we haven't defined it yet. 

## Step 2 &mdash; Define your lambda script

Inside the `app/` directory, let's create `index.js` which becomes your Lambda entrypoint.

```javascript
'use strict';
const { execSync } = require('child_process');

function run(command) {
    console.log(command);
    const result = execSync(command, {stdio: 'inherit'});
    if (result != null) {
        console.log(result.toString());
    }
}

exports.handler = async (event) => {
    console.log("Video handler called");

    if (!event.Records) {
        return;
    }

    for (const record of event.Records) {
        const fileName = record.s3.object.key;
        const bucketName = record.s3.bucket.name;
        const thumbnailFile = fileName.substring(0, fileName.indexOf("_")) + ".jpg";
        const framePos = fileName.substring(fileName.indexOf("_")+1, fileName.indexOf(".")).replace("-", ":");
        
        run(`aws s3 cp s3://${bucketName}/${fileName} /tmp/${fileName}`);
        run(`ffmpeg -v error -i /tmp/${fileName} -ss ${framePos} -vframes 1 -f image2 -an -y /tmp/${thumbnailFile}`);
        run(`aws s3 cp /tmp/${thumbnailFile} s3://${bucketName}/${thumbnailFile}`);

        console.log(`*** New thumbnail: file ${fileName} was saved at ${record.eventTime}.`);
    }    
};
```

{{% notice info %}}
The `app` directory should now look like this:
{{% /notice %}}
```
.
├── Dockerfile
└── index.js
```


## Step 3 &mdash; Build Docker Image & Push to ECR

Back inside your Pulumi program (`src/main/java/myproject/App.java`) we now need to build this Docker image and create a place to push it.

Add the following to `App.java`:

```java
var repo = new Repository("thumbnailer");
var image = new Image("thumbnailer", ImageArgs.builder()
        .repositoryUrl(repo.repositoryUrl())
        .path("./app")
        .build());
```

{{% notice info %}}
The `App.java` file should now look like this:
{{% /notice %}}
```java
package myproject;

import com.pulumi.Pulumi;
import com.pulumi.aws.ecr.Repository;
import com.pulumi.awsx.ecr.Image;
import com.pulumi.awsx.ecr.ImageArgs;

public class App1 {

    public static void main(String[] args) {
        Pulumi.run(ctx -> {
            var repo = new Repository("thumbnailer");
            var image = new Image("thumbnailer", ImageArgs.builder()
                    .repositoryUrl(repo.repositoryUrl())
                    .path("./app")
                    .build());
        });
    }
}
```

This Pulumi code takes care of creating an ECR repository and building the local container image and pushing it.

## Step 4 &mdash; Run the Pulumi Program

Run your Pulumi program and inspect the resources it'll create:

```
Previewing update (dev)

View Live: https://app.pulumi.com/slnowak/iac-workshop-thumbnailer/dev/previews/c58c0be2-a3da-468a-9e46-6803a177b63f

     Type                   Name                          Plan       
 +   pulumi:pulumi:Stack    iac-workshop-thumbnailer-dev  create     
 +   ├─ aws:ecr:Repository  thumbnailer                   create     
 +   └─ awsx:ecr:Image      thumbnailer                   create     
 
Resources:
    + 3 to create
```

Hit yes when you're ready, you should see some output while the image builds in the background (this may take a few minutes):

```
Updating (dev)

View Live: https://app.pulumi.com/slnowak/iac-workshop-thumbnailer/dev/updates/8

     Type                   Name                          Status       Info
 +   pulumi:pulumi:Stack    iac-workshop-thumbnailer-dev  creating.    dockerBuild: {"context":"./app"}
 +   ├─ aws:ecr:Repository  thumbnailer                   created      
 +   └─ awsx:ecr:Image      thumbnailer                   created      a6bec7d40f2d: Pushed
```



