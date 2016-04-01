Cloudbees Jenkins Platform Openshift Templates
==============================================

These are the templates and instructions to install Cloudbees Jenkins Platform on Openshift V3 platform. The templates work for the Origin open-source version and for the Red Hat Enterprise version (Dedicated and On-Premise).

Basic knowledge of the Openshift CLI and the Openshift platform is required.
* [Openshift platform documentation](https://docs.openshift.org/latest/welcome/index.html)
* [Getting started with Openshift CLI](https://docs.openshift.org/latest/cli_reference/get_started_cli.html)

### Topology
The base topology supported by these templates is as follows:
* 1 Cloudbees Jenkins Operations Center (CJOC)
* 1 Elastic Search
* *unlimited* Cloudbees Jenkins Enterprise (CJE)
* Dynamic Pod agents configured at CJE level



### TODOs

* Non HTTP(S) endpoints
	* CloudBees Jenkins Platform needs non HTTP(S) endpoints for 
		* JNLP (50000/TCP) for Jenkins CLI and JNLP agents located outside of the environment (MacOS/Windows agents...)
		* GIT-SSH (2222/SSH) for CloudBees Git Validated Merge feature


# Installation & Usage

* Login to Openshift with CLI
```
	$ oc login --server=<server_url>
    Username: admin
	Password:
	Login successful.

	You have access to the following projects and can switch between them with 'oc project <projectname>':

	  * default (current)
	  * openshift
	  * openshift-infra

	Using project "default".
```
* Create a new project to deploy to
```
    $ oc new-project cjp --display-name="Jenkins Cloud Platform (CJP)" --description="Jenkins Cloud Platform (CJP) with: Jenkins, Jenkins Operations Center and Elastic Search. Slaves are provisioned automatically from Jenkins Enterprise"
	Now using project "cjp" on server "https://openshift.beesshop.org:8443".
	...
```

## Install templates

On this example, templates are going to be installed on the newly created project, but could be installed on the _default_ project to make it available to any other project on the platform.

There are two template files:

* _cjp-joc-elastic-search-base-images.json_ contains JOC components, Elastic Search components and ImageStreams for CJOC, Elastic Search and CJE.
* _cjp-jenkins.json_ contais CJE components, depends on the previous template for the ImageStream and allows the creation of multiple CJE instances.

Being on the root folder of the repository, execute the following commands to import the templates.

```
    $ oc create -f cjp-joc-elastic-search-base-images.json
    template "cjp-ops-center-and-base-images" created
```

```   
    $ oc create -f cjp-jenkins.json
    template "cjp-jenkins-enterprise" created
```

## Create CJOC, Elastic Search application

Add to the project a CJOC application, together with Elastic Search and base ImageStreams. This can be done by clicking "Add to Project" on the console and selecting _cjp-ops-center-and-base-images_ or by executing the following command:

```   
    $ oc new-app cjp-ops-center-and-base-images
	--> Deploying template cjp-ops-center-and-base-images for "cjp-ops-center-and-base-images"
	     With parameters:
	      PUBLIC_URL=
	      CJOC_STORAGE=20Gi
	      ELASTIC_SEARCH_STORAGE=20Gi
	--> Creating resources ...
	    service "jenkins-ops-center" created
	    route "joc" created
	    imagestream "jenkins-ops-center" created
	    deploymentconfig "jenkins-ops-center" created
	    persistentvolumeclaim "joc-claim" created
	    service "elasticsearch" created
	    imagestream "elasticsearch" created
	    deploymentconfig "elasticsearch" created
	    persistentvolumeclaim "elasticsearch-claim" created
	    imagestream "jenkins-enterprise" created
	--> Success
	    Run 'oc status' to view your app.
```

The applications services will be created and deployed. Optionally PUBLIC_URL, CJOC_STORAGE and ELASTIC_SEARCH_STORAGE can be set with the -h command. If the console interface is used, a form with the parameters will be shown.

Once the application is deployed and running you can access it from the URL defined for the service. This URL can be found on the overview page of the console.

_**Notice** - Creating more tan one CJOC application on the same project is not supported._
_**Tip** - Elastic Search comes with the standard [elasticsearch](https://hub.docker.com/_/elasticsearch/) Docker Image configuration, which may be enough for most cases. Please refer to [image readme file](https://hub.docker.com/_/elasticsearch/) for instructions o how to customize it._

### Configure CJP to use Elastic Search service
	- Login into CJP -> Manage Jenkins -> Configure analytics
	- Change Elastic Search Configuration to "Remote Elastic Search"
	- Change Elastic Search URL to -> http://elasticsearch.cjp.svc.cluster.local:9200
	- Test Connection and Save.

## Create CJE application

Add to the project CJE application. This can be done by clicking "Add to Project" on the console and selecting _cjp-ops-center-and-base-images_ or by executing the following command:

```   
    $ oc new-app cjp-jenkins-enterprise
	--> Deploying template cjp-jenkins-enterprise for "cjp-jenkins-enterprise"
	     With parameters:
	      PUBLIC_URL=
	      JENKINS_SERVICE=jenkins
	      JENKINS_STORAGE=20Gi
	--> Creating resources ...
	    service "jenkins" created
	    route "jenkins" created
	    error: imagestreams "jenkins-enterprise" already exists
	    deploymentconfig "jenkins" created
	    persistentvolumeclaim "jenkins" created
``` 

The applications services will be created and deployed. Optionally PUBLIC_URL, JENKINS_SERVICE and JENKINS_STORAGE can be set with the -h command. If the console interface is used, a form with the parameters will be shown.

Once the application is deployed and running you can access it from the URL defined for the service. This URL can be found on the overview page of the console.

_**Notice** - Creating more tan one CJE is supported, but a unique JENKINS_SERVICE parameter must be provided._

### Add CJE to CJOC

 Follow the standard instructions - [Installing a Client Master](https://documentation.cloudbees.com/docs/cjoc-user-guide/_installing_a_client_master.html)

### Configure slaves
	- Login to CJE 
	- Install Kubernetes plugin
	- Go to Manage Jenkins and change the following parameters
		- Executors 0
		- Add New Cloud -> Kubernetes
			- Name: Openshift
			- URL: https://openshift.default.svc.cluster.local
			- NameSpace: cjp
			- Jenkins URL: http://<jenkins_service_name>.cjp.svc.cluster.local:8080/
			- Add Template -> Kubernetes POD Template
				- Name: Jenkins Slave
				- Docker Image: cloudbees/jnlp-slave-with-java-build-tools

_**Tip** - Depending on the configuration of your Openshift environment, it may be needed to provide additional credentials to the default user on the new project, to be able to instantiate agents from CJE. It can be done by executing the following command_

``` 
		oc policy add-role-to-user edit system:serviceaccount:cjp:default
``` 
















