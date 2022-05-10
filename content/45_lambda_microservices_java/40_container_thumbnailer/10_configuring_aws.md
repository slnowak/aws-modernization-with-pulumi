+++
title = "1.2 Configuring AWS"
chapter = false
weight = 10
+++

Now that you have a basic project, let's configure AWS support for it.

## Step 1 &mdash; Add AWS dependencies
In this particular example, we're going to use both [AWS Classic](https://www.pulumi.com/registry/packages/aws/) and [AWSX](https://www.pulumi.com/registry/packages/awsx/) providers.
To use these packages, we need to include them as dependencies.

You can do it by editing your `pom.xml` file. Find the `dependencies` section and add the following entries:
```xml
<dependency>
    <groupId>com.pulumi</groupId>
    <artifactId>aws</artifactId>
    <version>5.4.0</version>
</dependency>
<dependency>
    <groupId>com.pulumi</groupId>
    <artifactId>awsx</artifactId>
    <version>1.0.0-beta.2</version>
</dependency>
```

Your `pom.xml` file should look like the following:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.pulumi</groupId>
    <artifactId>iac-workshop</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <encoding>UTF-8</encoding>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <maven.compiler.release>17</maven.compiler.release>
        <mainClass>myproject.App</mainClass>
        <mainArgs/>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.pulumi</groupId>
            <artifactId>pulumi</artifactId>
            <version>0.1.0</version>
        </dependency>
        <dependency>
            <groupId>com.pulumi</groupId>
            <artifactId>aws</artifactId>
            <version>5.4.0</version>
        </dependency>
        <dependency>
            <groupId>com.pulumi</groupId>
            <artifactId>awsx</artifactId>
            <version>1.0.0-beta.2</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>3.2.2</version>
                <configuration>
                    <archive>
                        <manifest>
                            <addClasspath>true</addClasspath>
                            <mainClass>${mainClass}</mainClass>
                        </manifest>
                    </archive>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>3.3.0</version>
                <configuration>
                    <archive>
                        <manifest>
                            <addClasspath>true</addClasspath>
                            <mainClass>${mainClass}</mainClass>
                        </manifest>
                    </archive>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                </configuration>
                <executions>
                    <execution>
                        <id>make-my-jar-with-dependencies</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>exec-maven-plugin</artifactId>
                <version>3.0.0</version>
                <configuration>
                    <mainClass>${mainClass}</mainClass>
                    <commandlineArgs>${mainArgs}</commandlineArgs>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-wrapper-plugin</artifactId>
                <version>3.1.0</version>
                <configuration>
                    <mavenVersion>3.8.5</mavenVersion>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>

```

## Step 2 &mdash; Configure an AWS Region

Configure the AWS region you would like to deploy to:

```bash
pulumi config set aws:region us-west-2
```

Feel free to choose any AWS region that supports the services used in these labs ([see this table](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html#concepts-available-regions) for a list of available regions).

## (Optional) Step 4 &mdash; Configure an AWS Profile

If you're using an alternative AWS profile, you can tell Pulumi which to use in one of two ways:

* Using an environment variable: `export AWS_PROFILE=<profile name>`
* Using configuration: `pulumi config set aws:profile <profile name>`