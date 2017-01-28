# Develop all-the-things() with a dev-ci-cd pipeline

## Problem

How do we enable developers on a project to be more efficient and produce higher quality code into production environments and not overburden them with a CI/CD process?

Often developers want to develop code and continuously test it and produce an end component, application, module, etc.  In these cases more often than not when the component or application is built and tested in a developer's environment it behaves differently than in a stage or production environment.  How do we allow developers to make changes faster without compromising quality.

## Solution

If a developer can develop, build, test, and continuously integrate a component as it would be in production in their own development environment before it even goes into a pull request or even worse checked in to the code base it help identify issues sooner and closer to the changes and the person making the changes.

Openshift has put an excellent PaaS and CI/CD pipeline combination together that can be be run in a developer’s environment to Build->Test->Integrate->Test->Deliver->Deploy to environments. (ex. Stage and Production)

A developer can take their code and easily move it through a CI/CD Pipeline on Openshift which is a Platform as a Service (PaaS). Openshift combined with Jenkins and Jenkins pipeline to develop code from their local sandbox, git repositories, docker hub, etc.

## Overview

### CI/CD in Openshift

* Webhook triggers for builds
  * Build my application image when my code changes
* ImageChangeTriggers for builds
  * Build my application image when my base image changes
* Post-build test hooks
  * Test my application image before I push it to a registry
* ImageChangeTriggers for deployments
  * Redeploy my application when my image changes (e.g. after a build)

### Jenkins Pipeline

Developers can store their tests and CI/CD pipelines with their source code and this can be automatically loaded into the Jenkins Master container instance.  So as changes are made to code and tests it can also be made to the pipeline itself and tested.  This all can be run from within the developer's desktop to be used as an identical setup of what runs in production

* Groovy syntax for running arbitrarily complex workflows
* Many plugins offer a DSL
* Pipeline is defined in a Jenkinsfile
* Jenkinsfile lives with source code on developers local code or in scm

### Openshift Pipelines

* New Pipeline BuildConfig strategy type
* Jenkins master and slave images
* Synchronization logic between Jenkins and OpenShift
* Web console for viewing Pipelines
* Auto-provisioning of Jenkins as needed

## Components

* Openshift Cluster
* Router, Registry, HA Proxy,  etc
* Pipelines
* Apps
* Jenkins Example
* Master and slave images
git server

## Setup

### Prerequisites

* oc
* oc cluster
* oc-cluster-wrapper

### Install

#### Launch Openshift Cluster

Add the below line to /etc/sysconfig/docker and restart docker
````
INSECURE_REGISTRY='--insecure-registry 172.30.0.0/16'
````
Install oc and oc-cluster-wrapper and set the path

````
$ sudo iptables -F
$ oc cluster up 
$ oc login
$ Username: developer
$ Password: developer
````

#### Deploy Jenkins

Deploy a new project and Jenkins ephemeral or persistent master

````
$ oc new-project 
$ oc new-app jenkins-persistent
````

Otherwise:

````
$ oc new-app jenkins-ephemeral
````

Find route to Jenkins service

````
$ oc get route
````

Loading the Sample App Configuration

If you decide to load the sample app that will appear as a job in Jenkins to test

````
$ oc new-app -f https://raw.githubusercontent.com/openshift/origin/master/examples/jenkins/application-template.json
````

#### Deploy Local Git Server

````
$ oc create -f https://raw.githubusercontent.com/openshift/origin/master/examples/gitserver/gitserver-ephemeral.yaml
$ oc policy add-role-to-user edit -z git
$ GITSERVER=http://$(oc get route git -o template --template '{{.spec.host}}')
$ echo $GITSERVER
$ git config --global credential.$GITSERVER.helper \
'!f() { echo "username=$(oc whoami)"; echo "password=$(oc whoami -t)"; }; f'
````

#### Add your project to start development

##### Clone a public repository

````
$ git clone https://github.com/arilivigni/linch-pin.git
````

##### Add a remote for your git server


````
$ cd linch-pin
$ git remote add openshift $GITSERVER/linch-pin.git
````

##### Add Jenkinsfile to root of repo

````
$ cd linch-pin
$ git remote add openshift $GITSERVER/linch-pin.git
````

##### Add Jenkinsfile to root of repo

ex. Jenkinsfile

````
node('linch-pin-slave') {
  stage ('build') {
    dir('linch-pin') {
        git url: 'http://git:8080/linch-pin.git'
        sh '''
            virtualenv $WORKSPACE/lp-test-venv
            source $WORKSPACE/lp-test-venv/bin/activate
            ./install.sh
        '''
    }
  }
  stage ('test') {
    sh '''
        source $WORKSPACE/lp-test-venv/bin/activate
        pip install nosexcover pylint pycrypto flake8 pep8
        iconv -f utf8 -t ascii $WORKSPACE/linch-pin/requirements.txt >> $WORKSPACE/linch-pin/requirements.txt
        chmod -x $WORKSPACE/linch-pin/tests/*.py
        nosetests --verbosity=3 --with-xunit $WORKSPACE/linch-pin/tests/test_linchpin_creds.py
    '''
    archiveArtifacts '*.xml,**/*.md'
    junit 'nosetests.xml'
  }
}
````

##### Make changes and commit and push the code to the git server

````
$ git push openshift master
````