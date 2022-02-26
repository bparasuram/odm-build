# Topics
1. [Introduction](#introduction)
2. [Pipeline definitions](#pipeline)
3. [Building Task specific images](#images)

## Introduction <a name="introduction"></a>
This section describes the set of Pipeline and Tasks specific to building and deploying ODM projects. It is assumed that OpenShift Pipelines is deployed in the cluster and a namespace has been created to execute the build and deploy tasks.

## Pipeline definitions <a name="pipeline"></a>
1. The following ODM specific Tasks need to be defined in the namespace: task-odm-build.yaml, task-odm-deploy.yaml, task-termination.yaml
2. The following Pipleline needs to be defined in the namespace to build and deploy the ODM project: pipeline-build-pipeline.yaml
3. Define the ssh private key to the git repo containing the source code should be defined. Refer to secret-git-ssh-key.yaml
4. Define the PVC for the workspace. Refer to persistentvolumeclaim-tekton-workspace.yaml
5. Define the basic auth credentials for ODM server where the deployment is targeted. Refer to secret-odm-deploy-secret.yaml

#### To execute the Pipeline
1. Create a PipelineRun for **build-pipeline**
2. Provide the following to the PipelineRun
  - gitUrl -> The git repo where the ODM project has been checked-in.
  - decision_dir -> The directory where the output jar file from the build is to be found.
  - workspaces git-source -> need to map the **tekton-workspace** PVC here.
  - workspaces ssh-dir -> need to map the **git-ssh-key** secret here. 


### Pipeline Tasks involved:
1. **git-clone** -> from Tekton cluster task git-clone. Will need to pass the git specific parameters.
2. **odm-build** -> no parameters required. It executes mvn clean install from the top level project directory to build the ODM project. It is assumed that the source code is checked in and necessary pom.xml files are already defined for the decision service and XOMs and validated by the developer, ensuring that the build works correctly outside the CI/CD Pipeline to beging with. If successful, it creates a jar file containing the ODM executable along with the XOM definition jars into the Decision Service directory. 
3. **odm-deploy** -> Takes the output jar from **odm-build** Task and deploys to the target ODM Rule Execution Server (res). The parameters are
  - odm_url -> The res url
  - odm_secret -> The basic-auth credentials for res.
  - target_dir -> this is the sub-directory within the git source repo where the target directory containing the jar file created by **odm-deploy** is present.
4. If the deployment is successful, **odm-deploy** script ends with a success message, else exits with an error

## Building Task specific images  <a name="images"></a>
1. Refer to the Dockerfile present in current directory. We create an image that contains all the dependencies to run an ODM build. This is used by the **odm-build** Task.
2. docker build -t maven:latest .
3. docker tag maven:latest
4. oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
5. HOST=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
6. docker tag maven:latest $HOST/cp4ba-build/maven:latest
7. docker push $HOST/cp4ba-build/maven:latest

