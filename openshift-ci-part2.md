# Running unit tests in a Pipeline

This article explains how to run a command within the running
container via the Openshift equivalent of `docker exec`.

This article is part of a [series](openshift-ci-part1.md) on Continuous 
Integration using OpenShift

## Basics

Let's start with the skeleton of the Jenkinsfile; this does nothing more
than build the app.

```groovy
node {
  def APPNAME = "demo-app"
  
  stage('build') {
     openshiftBuild(buildConfig: APPNAME, showBuildLogs: 'true')
  }
  
  stage('test') {
    echo "we need to add some Groovy here"
  }
  
  stage('promote') {
    echo "we need to add some Groovy here"
  }
}
```

**Note** that `node` takes no parameter here, on the first line. A parameter
would be used to specify the type of Jenkins node to run on. At this point, 
we just want things to run on the Jenkins-Master.

The 2nd and 3rd stages do nothing at the moment. We'll add code to these later.


## Running a command in the container
In the `stage('test') {}` stage, we would like to exec a command, which we
will do with the **openshiftExec** command. The intention is to be able to
run a test within the same environment as the application itself.
For the purposes of the demo, we will "curl" the homepage of our app which
is not strictly a unit test, but demonstrates that we can execute a shell
command within the container from the Jenkinsfile.

(Reminder: we are trying to exec
the command within the container, NOT in the Jenkins agent)

**openshiftExec** takes a pod name and an array of shell commands as its
arguments. The first challenge is therefore to discover the name of the 
[pod](https://docs.openshift.com/online/architecture/core_concepts/pods_and_services.html).

We can do this with a "selector". The following Jenkinsfile works out the 
name of the pod and calls **openshiftExec**.


```groovy
node {
  stage('build') {
    def APPNAME = "demo-app"
    openshiftBuild(buildConfig: APPNAME, showBuildLogs: 'true')

    // Wait 10 seconds for the WSGI app to start up
    sleep 10
  }
  
  stage('test') {
    openshift.withCluster() {
      openshift.withProject() {
      
          /* Get the Deployment Config (DC) Object */
          def dcObj = openshift.selector('dc', APPNAME).object()
          
          /* Use the DC to find the pods */
          def podSelector = openshift.selector('pod', [deployment: "${APPNAME}-${dcObj.status.latestVersion}"])
          
          podSelector.withEach {
              // obtain the pod name and remove "pods/" from the front.
              def podName = it.name()  // probably pods/abcd1234
              podName = podName.replaceFirst("pods/", "")
              echo "Running unit tests against ${podName}"

              // Run a command on the container
              def resp = openshiftExec(pod: podName, command: ["/usr/bin/curl", "http://127.0.0.1:8080/"])

              assert resp.stdout.contains("Hello World!")              
          }
      }
    }
  }
}
```
**Notes**
* withCluster and withProject allow you to specify which cluster and which
  project; they become more useful later.
* You can print out all the details of the podSelector object with 
  `podSelector.describe()`
* The project name is available via `openshift.project()`
* We have to iterate over the podSelector object with `withEach`. Within the
  loop, the current item is `it`
* The pod name appears as "pods/abcd1234", but we need to remove the first 5
  letters from the front. This is done using Groovy's `String.replaceFirst()`
* The `command` parameter to `openshiftExec` must be an array. The short form
  described in 
  [the documentation](https://github.com/openshift/jenkins-plugin#run-openshift-exec)
  (i.e. `command: "curl http://127.0.0.1:8080"`)
  does not work because it does not perform shell interpolation. You should 
  use `command: [cmd, arg1, arg2, arg3]`
* **openshiftExec** returns a response object, which provides parameters such
  as stdout, stderr, error and failure. We use the 'stdout' parameter to
  verify the app is giving us the phrase we expect
  ([see the source code](https://github.com/OpenShiftDemos/os-sample-python/blob/master/wsgi.py)).
* There may be more efficient ways of doing the above in Groovy

In [Part 3](openshift-ci-part3.md), we'll discuss an alternative mechanism
for running the tests in the Jenkins Slave. 
