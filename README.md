# vny

Velocity New York Tutorial

# Pre-requisites

## 1) GitHub account

Please [sign up for a GitHub account](https://github.com/join) if you don't have one already!

## 2) Docker Hub account

Please [sign up for a new Docker Hub account](https://hub.docker.com/). Since we'll be using shared Jenkins instances, please use a throwaway account for this tutorial.

## 3) DC/OS cluster

You will need a DC/OS cluster! We have provisioned 15 of these for the tutorial, listed below. Access details will be provided in the slides.

Please use the number that has been assigned to you!

| Cluster | DC/OS Hostname | Public Agent Hostname|
| --- | --- | --- |
| 1 | https://dcos01.vny.mesosphere.com | https://public.dcos01.vny.mesosphere.com |
| 2 | https://dcos02.vny.mesosphere.com | https://public.dcos02.vny.mesosphere.com |
| 3 | https://dcos03.vny.mesosphere.com | https://public.dcos03.vny.mesosphere.com |
| 4 | https://dcos04.vny.mesosphere.com | https://public.dcos04.vny.mesosphere.com |
| 5 | https://dcos05.vny.mesosphere.com | https://public.dcos05.vny.mesosphere.com |
| 6 | https://dcos06.vny.mesosphere.com | https://public.dcos06.vny.mesosphere.com |
| 7 | https://dcos07.vny.mesosphere.com | https://public.dcos07.vny.mesosphere.com |
| 8 | https://dcos08.vny.mesosphere.com | https://public.dcos08.vny.mesosphere.com |
| 9 | https://dcos09.vny.mesosphere.com | https://public.dcos09.vny.mesosphere.com |
| 10 | https://dcos10.vny.mesosphere.com | https://public.dcos10.vny.mesosphere.com |
| 11 | https://dcos11.vny.mesosphere.com | https://public.dcos11.vny.mesosphere.com |
| 12 | https://dcos12.vny.mesosphere.com | https://public.dcos12.vny.mesosphere.com |
| 13 | https://dcos13.vny.mesosphere.com | https://public.dcos13.vny.mesosphere.com |
| 14 | https://dcos14.vny.mesosphere.com | https://public.dcos14.vny.mesosphere.com |
| 15 | https://dcos15.vny.mesosphere.com | https://public.dcos15.vny.mesosphere.com |

### Ground Rules

Since these clusters are shared resources for the tutorial, please be courteous:
+ clean up applications after yourself
+ don’t delete other people’s applications
+ don't install any other application from the DC/OS Universe since this will consume resources that will prevent other attendees from running the tutorial steps
+ ...and so on

# Exercises

## Exercise 0: DC/OS UI

Before we start,  let’s take a quick look at the DC/OS UI by visiting the DC/OS hostname provided to you above.

See the [DC/OS GUI docs](https://docs.mesosphere.com/1.8/usage/webinterface/) for a quick overview of the main components of the UI. We'll be using the UI to observe tasks spinning up on our cluster as our pipeline works.

## Exercise 1: DC/OS CLI

Start by installing the DC/OS CLI. This is an extensible, Python-based command line tool that lets you interact with a DC/OS cluster.

+ Follow the instructions here to set it up: https://dcos.io/docs/1.8/usage/cli/install/
+ Use the same cluster hostname you used to access the UI (e.g. https://dcos01.vny.mesosphere.com)

Next, try running some example queries:

+ List all running applications with: `dcos marathon app list`
+ List running tasks with: `dcos task`
+ Show connected nodes with: `dcos node`

## Exercise 2:  Set up a GitHub repository

Let's create a GitHub repository to use for the rest of this tutorial. This will be where we store the source for our very simple Nginx website, along with the configuration files necessary to create a pipeline.

1. [Create a new GitHub repository](https://github.com/new). We will use https://github.com/mesosphere/vny for the rest of this tutorial, you should replace this with the URL of the repo you've just created.

    + For simplicity, set the repository visibility to "Public" and initialise it with a README.

1. Check it out locally, e.g.:

    git clone https://github.com/mesosphere/vny.git && cd vny

## Exercise 3: Set up a Docker Hub repository

Next, let's create a custom Docker Hub repository for your pipeline. This will be where we store built Docker images prior to deploying them.

1. [Create a new repository](https://hub.docker.com/add/repository/). In our examples, we'll be using mesosphere/vny. Make sure to replace this with your own URL (since you won't be able to push to it otherwise :)).

    + Set the repository visibility to "Public". This will let our agent nodes pull from it without needing explicit authorization.

## Exercise 4: Add your Docker Hub credentials to Jenkins

Next, we'll add these Docker Hub credentials to Jenkins, so that you're able to push to this repository from your Pipeline.

1. Access Jenkins on the DC/OS cluster to which you've been assigned by visiting the Services page, clicking on Jenkins and then clicking on Open Service.

1. Go to the Credentials page.

1. Click on "(global)" underneath the "Stores scoped to Jenkins" section.

1. Click "Add Credentials" and enter your Docker Hub username and password. Give it a meaningful ID. We will use `dockerhub-mesosphere` in the examples below. Be sure to rename it to yours!

## Exercise 5: Create an application and Dockerise it

Now we're going to create a very simple Dockerised Nginx website.

1. The very first thing we'll do is create a simple index.html in your git repository. Create the file and paste the following into it:

```
<html> Hello World! </html>
```

1. Let's create our [Dockerfile](https://docs.docker.com/engine/reference/builder/) now. The Dockerfile tells Docker how to package and run our application. Create a file called `Dockerfile` in the root of your repository. This tells Docker to extend the `nginx` image and copy the `index.html` file we just created into the right path within the container image:

```
FROM nginx
COPY index.html /usr/share/nginx/html/index.html
```

1. (Optional) If you like, you can build and run this Docker container locally to check that it works. Run the following commands and then visit http://localhost:8080 in your browser:

```
docker build -t mesosphere/vny .
docker run -p 8080:80 mesosphere/vny
```

1. Let's add both of these files to the git index and push them up to Docker Hub:

```
git add index.html
git add Dockerfile
git commit -m "Add index.html and Dockerfile"
git push origin master
```

## Exercise 6: Set up new Jenkins pipeline job

Let's set up some continuous integration for this application! What we're going to do is set up a Jenkins build using the new [Pipeline](https://jenkins.io/doc/pipeline/) functionality that's part of Jenkins 2.0. This build will build the container and push it to Docker Hub for us.

1. A core part of Pipeline is allowing you to script your builds and check these in with your code. The very first thing we'll do is create a `Jenkinsfile` in the root of your repository. Paste the following into it. Make sure to replace the `mesosphere/vny` with your Docker Hub repository, and `dockerhub-mesosphere` with the name of your Docker Hub credentials:

```
def gitCommit() {
    sh "git rev-parse HEAD > GIT_COMMIT"
    def gitCommit = readFile('GIT_COMMIT').trim()
    sh "rm -f GIT_COMMIT"
    return gitCommit
}

node {
    // Checkout source code from Git
    stage 'Checkout'
    checkout scm

    // Build Docker image
    stage 'Build'
    sh "docker build -t mesosphere/vny:${gitCommit()} ."

    // Log in and push image to GitLab
    stage 'Publish'
    withCredentials(
        [[
            $class: 'UsernamePasswordMultiBinding',
            credentialsId: 'dockerhub-mesosphere',
            passwordVariable: 'DOCKERHUB_PASSWORD',
            usernameVariable: 'DOCKERHUB_USERNAME'
        ]]
    ) {
        sh "docker login -u ${env.DOCKERHUB_USERNAME} -p ${env.DOCKERHUB_PASSWORD} -e demo@mesosphere.com"
        sh "docker push mesosphere/vny:${gitCommit()}"
    }
}
```

1. Add this to your git index and push it up to GitHub:
```
git add Jenkinsfile
git commit -m "Add Jenkinsfile"
git push origin master
```

1. Next, we'll create the pipeline job in Jenkins itself. Navigate to the Jenkins UI and click on New Item. We'll create a new "Pipeline" job. Be sure to pick a more descriptive name, e.g. `nginx-mesosphere`:

![Create New Item](/img/new-item.png)

1. For this simple example, we'll just select "Poll SCM" and set the schedule to `* * * * *`. This asks us to poll every minute, which might be inefficient for large installations:

![Poll SCM](/img/poll-scm.png)

1. Next, change the Pipeline definition to use "Pipeline script from SCM" and configure the repository you created earlier (e.g. `https://github.com/mesosphere/vny.git`):

![Pipeline Configuration](/img/pipeline.png)

1. Hit save. You should see the Pipeline trigger within a minute. After it triggers, a build agent will be spun up dynamically on the DC/OS cluster and the build will run there.

1. Once the build has completed successfully, visit Docker Hub to verify that a Docker image has been pushed and tagged with the latest git commit SHA.

## Exercise 7: Add Marathon deploy step

Now that we have a working Docker build and push pipeline, let's add a deploy step to it! This will deploy our application to [Marathon](https://docs.mesosphere.com/1.8/usage/service-guides/marathon/) that will be available to our authenticated users.

1. First, let's create a new `marathon.json` file. This tells Marathon how to run our application. The following JSON blob specifies some basic properties of the application (such as how many resources to give it), what Docker image to use, as well as some other metadata. Again, replace the Docker image with your own, and change the `id `, as well as the `DCOS_SERVICE_NAME` to something unique for your application:

```json
{
  "id": "/nginx-mesosphere",
  "cpus": 1,
  "mem": 128,
  "instances": 1,
  "container": {
    "docker": {
      "image": "mesosphere/vny:latest",
      "portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp",
          "name": "http"
        }
      ],
      "network": "BRIDGE"
    }
  },
  "healthChecks": [
    {
      "protocol": "HTTP",
      "path": "/",
      "portIndex": 0,
      "gracePeriodSeconds": 5,
      "intervalSeconds": 10,
      "timeoutSeconds": 10,
      "maxConsecutiveFailures": 3
    }
  ],
  "labels": {
    "DCOS_SERVICE_PORT_INDEX": "0",
    "DCOS_SERVICE_SCHEME": "http",
    "DCOS_SERVICE_NAME": "nginx-mesosphere"
  }
}
```


1. Next let's add a snippet to the Jenkinsfile that makes use of the [Marathon plugin](https://github.com/jenkinsci/marathon-plugin) to kick off a new deployment as a step in our pipeline. Add this after the last stage in your Jenkinsfile but before the very last closing brace. In this blob, replace `appId` with a unique application ID and `mesosphere/vny` with your own Docker Hub repository name:

```
    // Deploy
    stage 'Deploy'

    marathon(
        url: 'http://marathon.mesos:8080',
        forceUpdate: false,
        credentialsId: 'dcos-token',
        filename: 'marathon.json',
        appId: 'nginx-mesosphere',
        docker: "mesosphere/vny:${gitCommit()}".toString()
    )
```

1. Add both files to your git index and push them up to GitHub:

```
git add marathon.json
git add Jenkinsfile
git commit -m "Add marathon.json and deploy step to Jenkinsfile"
git push origin master
```

1. This will trigger a new Pipeline build. once this completes, head over to the Services UI to see your application deploying and then running.

## Exercise 8: Demo pipeline

1. We can observe the pipeline working by committing a small change to the `index.html`. Change the contents of `index.html` to anything you like (e.g. "Hello Pipeline!"), then commit and push the change:

```
git add index.html
git commit -m "Update index.html"
git push origin master
```

1. This will trigger a new build. If you check out the services UI, you'll see that the application is re-deployed by Marathon automatically and your new index page is now being served!

## Exercise 9: Expose application using Marathon-lb

Often you'll want to expose your application to the public. We make use of Marathon-lb, a wrapper around HAProxy, an HTTP load balancer for this. This is already pre-installed for us and serving requests at the "Publice Agent Hostname" for your cluster up above. In this exercise, we'll add labels to our Marathon application that tells Marathon-lb to expose it to the internet.

1. Edit the `marathon.json` file in your repository and add the following label before the three `DCOS_` labels:

```
    "HAPROXY_GROUP": "external",
```

1. With the file still open, we need to update our `portMapping` to add a `servicePort` (you can read more about [Marathon ports here](https://mesosphere.github.io/marathon/docs/ports.html)). Change the portMapping array to include a `servicePort` as below. A value of 0 means Marathon will choose one randomly for us:

```
"portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp",
          "name": "http",
          "servicePort": 0
        }
      ],
```

1. Add this to your git index and push it up to GitHub:

```
git add marathon.json
git commit -m "Expose through Marathon-lb"
git push origin master
```

1. Marathon should re-deploy your application. Go find it in the Services UI and check out the "Configuration" tab for your application to see the `servicePort` that it has been assigned.

1. Now visit the public agent appended by the `servicePort` (e.g. http://public.dcos01.vny.mesosphere.com:10000/) to view your application available on the internet! 

## Exercise 10: Scaling & Deploys

Now that we've got our application running, let's scale it up and observe how we can do rolling deploys using Marathon without downtime.

1. First, let's scale the application up to 3 instances by updating the line in the marathon.json:

```
"instances": 3,
```

1.  Commit this, and wait for the Pipeline to run. Once the Pipeline completes, Marathon will trigger a new deployment.

```
git add marathon.json
git commit -m "Scale up"
git push origin master
```

1. Once these 3 instances are running and healthy, Marathon-lb will route to each randomly. If you kill one of these and try to access the application via Marathon-lb, it'll continue routing only to the healthy instances.

# Help!

If you're stuck, check the following:

+ Jenkins console logs can help diagnose why your build may not have worked. Did you replace all the right strings in the examples above?

+ Try rebuilding by hitting the "Build Now" button in Jenkins. Occasionally build contention and environmental issues may result in your build failing incorrectly.

+ Are you using the right credentials? Have you saved your credentials in Jenkins correctly?

+ Wait a little. Build executors on Mesos can sometimes take a few minutes to spin up, particularly if there is resource contention.

# Next Steps

+ Check out [the DC/OS documentation](https://dcos.io/docs/).

+ Get help by asking [the DC/OS Community](https://dcos.io/community/).