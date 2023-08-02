# Jenkins documentation

Here, you will be able to find everything related to concepts, setups, examples and information about how to implement CI for your projects using Jenkins.

# Getting started

TODO

# [Jenkins jobs](https://www.jenkins.io/doc/book/using/working-with-projects/)

What are Jenkins Jobs?
Jenkins Jobs are a given set of tasks that runs sequentially as defined by the user. Any automation implemented in Jenkins is a Jenkins Job. These jobs are a significant part of Jenkins's build process. We can create and build Jenkins jobs to test our application or project.

When working with Jenkins, the term “Jenkins Job” and “Jenkins Project” are synonymous. With a Jenkins Job, you can clone source code from version control like Git, compile the code, and run unit tests based on your requirements. Also, Jenkins allows you to merge code using other code management tools like Superversion, CVS, CVN, perforce, etc.

<font size=4>Types Of Jenkins Jobs</font>

There are different Jenkins Job types available intended for different purposes. Based on the complexity and nature of your project, you can choose the one that best suits your needs. Let us look at the different types of jobs in Jenkins briefly:

| Job Type                                                                      | Description                                                                                                                                                                                                                                                                                      |
| ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Freestyle Project                                                             | This is the central and the most widely used feature in Jenkins. It is an available Jenkins build job offering multiple operations. Using this option, you can build and run pipelines or scripts seamlessly.                                                                                    |
| Maven Project                                                                 | If your work involves managing and building projects containing POM files, you prefer using Maven Project to build jobs in Jenkins. On choosing this option, Jenkins, by default, will pick the POM files, make configurations, and run builds.                                                  |
| Pipeline                                                                      | Freestyle Project is often not a good option to create Jenkins Jobs. Therefore, Pipeline is the best option. Use the option Pipeline for creating Jenkins Jobs, especially when working on long-running activities.                                                                              |
| Multi-configuration Project                                                   | If you are working on a project requiring multiple configurations, you prefer to use the Multi-configuration Project option. This option allows for making multiple configurations for testing in multiple environments.                                                                         |
| GitHub Organization                                                           | If you click on this option, it scans the User's GitHub account for all repositories. And then, it matches markers as defined.                                                                                                                                                                   |
| [Multibranch pipeline](https://www.jenkins.io/doc/book/pipeline/multibranch/) | The Multibranch Pipeline project type enables you to implement different Jenkinsfiles for different branches of the same project. In a Multibranch Pipeline project, Jenkins automatically discovers, manages and executes Pipelines for branches which contain a Jenkinsfile in source control. |

# [Jenkins pipeline](https://www.jenkins.io/doc/book/pipeline/)

What is Jenkins Pipeline?
Jenkins Pipeline (or simply "Pipeline" with a capital "P") is a suite of plugins which supports implementing and integrating continuous delivery pipelines into Jenkins.

A continuous delivery (CD) pipeline is an automated expression of your process for getting software from **version control right** through to your users and customers. Every change to your software (committed in source control) goes through a complex process on its way to being released. This process involves **building the software** in a reliable and repeatable manner, as well as **progressing the built software** (called a "build") through multiple stages of `testing` and `deployment`.

# [Jenkinsfile](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/)

This section builds on the information covered in Getting started with Pipeline and introduces more useful steps, common patterns, and demonstrates some non-trivial Jenkinsfile examples.

Creating a Jenkinsfile, which is checked into source control, provides a number of immediate benefits:

- Code review/iteration on the Pipeline

- Audit trail for the Pipeline

- Single source of truth for the Pipeline, which can be viewed and edited by multiple members of the project.

Pipeline supports two syntaxes, Declarative (introduced in Pipeline 2.5) and Scripted Pipeline. Both of which support building continuous delivery pipelines. Both may be used to define a Pipeline in either the web UI or with a Jenkinsfile, though it’s generally considered a best practice to create a Jenkinsfile and check the file into the source control repository.

## [Declarative Pipeline fundamentals](https://www.jenkins.io/doc/book/pipeline/#declarative-pipeline-fundamentals)

In Declarative Pipeline syntax, the pipeline block defines all the work done throughout your entire Pipeline.

<font size=3>Jenkinsfile (Declarative Pipeline)</font>

```text
pipeline {
    agent any [1]

    stages {
        stage('Build') { [2]
            steps {
                echo 'Building..' [3]
            }
        }
        stage('Test') { [4]
            steps {
                echo 'Testing..' [5]
            }
        }
        stage('Deploy') { [6]
            steps {
                echo 'Deploying....' [7]
            }
        }
    }
}
```

[1] - Execute this Pipeline or any of its stages, on any available agent.</br>
[2] - Defines the "Build" stage.</br>
[3] - Perform some steps related to the "Build" stage.</br>
[4] - Defines the "Test" stage.</br>
[5] - Perform some steps related to the "Test" stage.</br>
[6] - Defines the "Deploy" stage.</br>
[7] - Perform some steps related to the "Deploy" stage.</br>

## [Using environment variables](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/#using-environment-variables)

Jenkins Pipeline exposes environment variables via the global variable env, which is available from anywhere within a Jenkinsfile. Referencing or using these environment variables can be accomplished like accessing any key in a Groovy Map.

| Variable name   | Description                                                                                                                                                                                                                |
| --------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| BUILD_ID        | The current build ID, identical to BUILD_NUMBER for builds created in Jenkins versions 1.597+                                                                                                                              |
| BUILD_NUMBER    | The current build number, such as "153"                                                                                                                                                                                    |
| BUILD_TAG       | String of jenkins-${JOB_NAME}-${BUILD_NUMBER}. Convenient to put into a resource file, a jar file, etc for easier identification                                                                                           |
| BUILD_URL       | The URL where the results of this build can be found (for example http://buildserver/jenkins/job/MyJobName/17/ )                                                                                                           |
| EXECUTOR_NUMBER | The unique number that identifies the current executor (among executors of the same machine) performing this build. This is the number you see in the "build executor status", except that the number starts from 0, not 1 |
| JAVA_HOME       | If your job is configured to use a specific JDK, this variable is set to the JAVA_HOME of the specified JDK. When this variable is set, PATH is also updated to include the bin subdirectory of JAVA_HOME                  |
| JENKINS_URL     | Full URL of Jenkins, such as https://example.com:port/jenkins/ (NOTE: only available if Jenkins URL set in "System Configuration")                                                                                         |
| JOB_NAME        | Name of the project of this build, such as "foo" or "foo/bar".                                                                                                                                                             |
| NODE_NAME       | The name of the node the current build is running on. Set to 'master' for the Jenkins controller.                                                                                                                          |
| WORKSPACE       | The absolute path of the workspace                                                                                                                                                                                         |


<font size=4>Jenkinsfile using environment variables</font>

```text
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo "Running ${env.BUILD_ID} on ${env.JENKINS_URL}"
            }
        }
    }
}
```

<font size=4>Jenkinsfile setting environment variables</font>

```text
pipeline {
    agent any
    environment { [1]
        CC = 'clang'
    }
    stages {
        stage('Example') {
            environment { [2]
                DEBUG_FLAGS = '-g'
            }
            steps {
                sh 'printenv'
            }
        }
    }
}
```

[1] - 	An environment directive used in the top-level pipeline block will apply to all steps within the Pipeline.</br>
[2] - 	An environment directive defined within a stage will only apply the given environment variables to steps within the stage.
</br>

<font size=4>Jenkinsfile setting environment variables dynamically</font>

Environment variables can be set at run time and can be used by **shell scripts (sh)**, **Windows batch scripts (bat)** and **PowerShell scripts (powershell)**. Each script can either `returnStatus` or `returnStdout`. More information on scripts.

Below is an example in a declarative pipeline using sh (shell) with both `returnStatus` and `returnStdout`.

```text
pipeline {
    agent any [1]
    environment {
        // Using returnStdout
        CC = """${sh(
                returnStdout: true,
                script: 'echo "clang"'
            )}""" [2]
        // Using returnStatus
        EXIT_STATUS = """${sh(
                returnStatus: true,
                script: 'exit 1'
            )}"""
    }
    stages {
        stage('Example') {
            environment {
                DEBUG_FLAGS = '-g'
            }
            steps {
                sh 'printenv'
            }
        }
    }
}
```

[1] - 	An agent must be set at the top level of the pipeline. This will fail if agent is set as agent `none`.</br>
[2] - 	When using returnStdout a trailing whitespace will be appended to the returned string. Use .trim() to remove this.
</br>