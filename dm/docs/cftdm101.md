# Cloud Foundation Toolkit for Deployment Manager 

## Overview

This tutorial shows you how to use the Deployment Manager templates of the Cloud Foundation Toolkit (CFT) to create a number of network resources.

The CFT is a public repository that provides reference templates for Deployment Manager and Terraform which reflect Google Cloud best practices. These templates can be used off-the-shelf to quickly build a repeatable enterprise-ready foundation in Google Cloud. 

If you are unfamiliar with Deployment manager, it is recommended to first take the [Deployment Manager Walkthrough tutorial](https://console.cloud.google.com/cloudshell/editor?cloudshell_git_repo=https%3A%2F%2Fgithub.com%2FGoogleCloudPlatform%2Fdeploymentmanager-samples&cloudshell_tutorial=walkthroughtutorial.md).

+   Creating a deployment configuration
+   Deploying resources
+   Using templates
+   Using helper scripts

After completing this tutorial, you can apply these techniques to carry out tasks such as:

+   Creating and managing Google Cloud Platform (GCP) resources using templates
+   Harnessing templates to reuse deployment paradigms
+   Deploying multiple resources at once
+   Configuring GCP resources such as Cloud Storage, Compute Engine, and Cloud SQL to work together

This tutorial assumes that you are familiar with YAML syntax and are comfortable running commands in a Linux terminal. 

### Select a project

Select a GCP Console project to use for this tutorial.

<walkthrough-project-setup></walkthrough-project-setup>

## Setup

Every command requires a project ID. Set a default project ID so you do not need to provide it every time. 

```sh  
gcloud config set project {{project-id}}  
```

Enable the Compute Engine and Deployment Manager APIs, which you will need for this tutorial.

```sh  
gcloud services enable compute.googleapis.com deploymentmanager.googleapis.com  
```

## Familiarizing with the Cloud Foundation Toolkit for Depoyment Manager

CFT includes the following parts:
+   A comprehensive set of production-ready resource templates that follow Google's best practices, which can be used with the CFT or the gcloud utility (part of the Google Cloud SDK) - see the [template directory](https://github.com/GoogleCloudPlatform/cloud-foundation-toolkit/blob/master/dm/templates/README.md)
+   A command-line interface (henceforth, CLI) that deploys resources defined in single or multiple CFT-compliant config files

Now take a moment to familiarize yourself with the different files that can be found in CFT. The Deployment Manager (DM) templates for CFT can be found [here](https://github.com/GoogleCloudPlatform/cloud-foundation-toolkit/tree/master/dm).


### CFT templates

Under the ‘templates’ folder you can browse through various CFT templates that are grouped by cloud resource e.g. ‘network’, ‘firewall’, ‘external_load_balancer’ etc. CFT templates can be written in Python or Jinja2 and are imported into the configuration file to parameterize your deployment. This allows you to reuse common deployment patterns.

In each template folder you’ll find the following files:

+   README.md - a textual description of the template's usage, prerequisites, etc.
+   resource.py - the Python 2.7 template file
+   resource.py.schema - the schema file associated with the template which describes what the valid input parameters are and the output from the template
+   examples:
    +   resource.yaml - a sample config file that utilizes the template
+   tests:
    +   integration:
        +   resource.yaml - a test config file
        +   resource.bats - a bats test harness for the test config
You can use the templates included in the template library:
+   Via Google's Deployment Manager / gcloud as described in the [Google SDK documentation](https://cloud.google.com/sdk/)
+   Via the CFT, as described in the [CFT User Guide](https://github.com/GoogleCloudPlatform/cloud-foundation-toolkit/blob/master/dm/docs/userguide.md)

### CFT vs DM
The GCP Deployment Manager service does not support cross-deployment references, and the gcloud utility does not support concurrent deployment of multiple inter-dependent configs. CFT expands the capabilities of Deployment Manager and gcloud to support the following scenarios:
+   Creation, update, and deletion of multiple deployments in a single operation which:
    +   Accepts multiple config files as input
    +   Automatically resolves dependencies between these configs
    +   Creates/updates deployments in the dependency-stipulated order, or deletes deployments in a reverse dependency order
+   Cross-deployment (including cross-project) referencing of deployment outputs, which removes the need for hard-coding many parameters in the configs

As with DM, to use CFT, you need to first create the config files for the desired deployments. These configs are YAML structures very similar to, and compatible with, the gcloud config files. The difference is that they contain extra YAML directives and features to support the expanded capabilities of the CFT (multi-config deployment and cross-deployment references).

The CFT templates/config file used in this Lab do not contain functionality that expands the capability of DM. As such all of the resource creation can be achieved using DM/gcloud.

## Creating Firewall rules
Next we are going to create three firewall rules using a CFT template. 

Using Cloud Shell:

1.  Clone the [Deployment Manager samples repository](https://github.com/GoogleCloudPlatform/cloud-foundation-toolkit):

```sh  
git clone https://github.com/GoogleCloudPlatform/cloud-foundation-toolkit  
```

2. Go to the dm directory:

```sh
cd cloud-foundation-toolkit/dm
```

3. Copy the example DM config to be used as a model for the deployment; in this case, examples/firewall.yaml:

```sh
cp templates/firewall/examples/firewall.yaml my_firewall.yaml
```

4. Next we are going to define three firewall rules in the config file by reusing the two existing firewall rule definitions in the config file and adding a third entry which uses network tags.

```sh
nano my_firewall.yaml
```

5. First select an existing VPC network or create a new one in which our firewall rules will be deployed. You can do this in the console by navigating to Networking -> VPC networks. Once you have a VPC go ahead and update the config file with the name of your VPC:

```sh
- name: test-firewall
   type: firewall.py
   properties:
     network: <FIXME:network-name># <== replace with name of your VPC
```

6. Add the following rule below the two example rules in my_firewall.yaml:

```sh
- name: allow-http-from-outside
          allowed:
            - IPProtocol: tcp
              ports:
                - "80"
          description: allow external traffic to reach DMZ instances
          direction: INGRESS
          sourceRanges:
            - 0.0.0.0/0
          targetTags:
            - ”dmz”
```

7. The final config file should look as follows:

```sh
imports:
  - path: templates/firewall/firewall.py
    name: firewall.py

resources:
  - name: test-firewall
    type: firewall.py
    properties:
      network: <YOUR_VPC_NETWORK_NAME># <== Replace accordingly
      rules:
        - name: allow-proxy-from-inside
          allowed:
            - IPProtocol: tcp
              ports:
                - "80"
                - "443"
          description: test rule for network-test
          direction: INGRESS
          sourceRanges:
            - 10.0.0.0/8
        - name: allow-dns-from-inside
          allowed:
            - IPProtocol: udp
              ports:
                - "53"
            - IPProtocol: tcp
              ports:
                - "53" 
          description: test rule for network-test-network
          direction: EGRESS
          priority: 20
          destinationRanges:
            - 8.8.8.8/32
        - name: allow-http-from-outside
          allowed:
            - IPProtocol: tcp
              ports:  
                - "80"
          description: allow external traffic to reach DMZ instances
          direction: EGRESS
          priority: 20 
          destinationRanges:
            - 8.8.8.8/32
          targetTags:
            - "dmz"
```

8. Deploy the firewall rules with the following command where ‘my-firewall-deployment’ is the name we have given the deployment:

```sh
gcloud deployment-manager deployments create my-firewall-deployment \
        --config my_firewall.yaml
```

9. Now in the console navigate to Networking > VPC Network > Firewall rules to verify the firewall rules have been created. Alternatively you can run the following gcloud command:

```sh
gcloud compute firewall-rules list
```

10. To view your deployment, navigate to Tools > Deployment Manager > Deployments

11. To clean up your deployment:

```sh
gcloud deployment-manager deployments delete my-firewall-deployment
```

## Creating a Logsink
Next we are going to create a log sink for your project that exports logs to a pubsub topic.

1. Copy the example DM config to be used as a model for the deployment; in this case, examples/org_logsink_pubsub_destination.yaml

```sh
cp templates/logsink/examples/org_logsink_pubsub_destination.yaml my_logsink.yaml
```

2. Start editing your config file:

```sh
nano my_logsink.yaml
```

3. Since we do not have a pub sub topic yet we will create it together with the log sink by using the relevant portion of the template i.e. delete the first block under "`resources:`". In the remaining portion replace the "`orgId`" property with "`projectId`" since we only want to export project level logs. 

```sh
- name: test-org-logsink-create-pubsub # <== RENAME to project-logsink
    type: logsink.py
    properties:
      orgId: <FIXME:OrgId># <== REPLACE with projectId: <yourprojectId>
```

4. Next specify a pubsub topic name and your user account email address to be given admin rights of the pubsub topic that will be created.

```sh
destinationName: <FIXME:pubsub_topic_name> #<== RENAME to my-pubsub-topic
      destinationType: pubsub
      uniqueWriterIdentity: true
      # Properties for the pubsub destination to be created.
      # Refer to templates/pubsub/pubsub.py.schema for supported properties.
      pubsubProperties:
        topic: <FIXME:topic_name> #<== RENAME to my-pubsub-topic
        accessControl:
          - role: roles/pubsub.admin
            members:
              - user:<FIXME:user_account_email_address> #<==UPDATE
```

5. Finally let’s also add the following query to the log sink so that the exported logs will be filtered to only contain deployment manager related entries

```sh
Filter: resource.type="deployment"
```

6. After modifying the template your config file should look like the following:

```sh
imports:
  - path: templates/logsink/logsink.py
    name: logsink.py

resources:
  
  # Organization sink with a pubsub destination that is created.
  - name: project-logsink
    type: logsink.py
    properties:
      projectId: <yourprojectId>
      # When using a PubSub topic, the value must be the topic ID. The ID must
      # contain only letters (a-z, A-Z), numbers (0-9), or underscores (_).
      # The maximum length is 1,024 characters.
      destinationName: my-pubsub-topic
      destinationType: pubsub
      uniqueWriterIdentity: true
      # Properties for the pubsub destination to be created.
      # Refer to templates/pubsub/pubsub.py.schema for supported properties.
      pubsubProperties:
        topic: my-pubsub-topic
        accessControl:
          - role: roles/pubsub.admin
            members:
              - user:<you_user_account_email_address>
      filter: resource.type="deployment"
```

7. Create your deployment where ‘my-logsink-deployment’ is the name we have given the deployment:

```sh
gcloud deployment-manager deployments create my-logsink-deployment \
    --config my_logsink.yaml
```

8. Now in the console navigate to Stackdriver > Logging > Exports to view the sink that we just created.
9. To view your deployment, navigate to Tools > Deployment Manager > Deployments
10. To clean up your deployment:

```sh
gcloud deployment-manager deployments delete my-logsink-deployment
```

## Creating an External HTTP Loadbalancer

For the final part of this tutorial we will create an external HTTP Loadbalancer

1. Start by copying the example DM config to be used as a model for the deployment; in this case, examples/external_load_balancer_http.yaml:

```sh
cp templates/external_load_balancer/examples/external_load_balancer_http.yaml \
       my_external_load_balancer.yaml
```

2. Open the config file to start editing

```sh
nano my_external_load_balancer.yaml
```

Notice that the config file requires a valid URL to be provided for the healthcheck and the instance group i.e. the template does not create these for you but instead expects these to be existing resources before deploying the external load balancer. 

For this tutorial we will assume that these resources still need to be created. This could be done manually through the console or by using existing templates to deploy the healthcheck and instance group separately after which we could then provide the config file for the external load balancer with the required URLs. In this tutorial, however, we will instead modify the config file to deploy the new healthcheck, instance group and external load balancer in a single deployment.

Also notice from the template [schema](https://github.com/GoogleCloudPlatform/cloud-foundation-toolkit/blob/master/dm/templates/external_load_balancer/external_load_balancer.py.schema), we are importing other modules e.g. backend_service.py, forwarding_rule.py etc. to create the different subcomponents that make up the load balancer.

3. Copy the healthcheck and managed instance group CFT config files into the external load balancer config file and remove the parts we don’t need; for the healthcheck sample config this means we will only keep the part for the HTTP healthcheck as can be viewed [here](https://github.com/GoogleCloudPlatform/cloud-foundation-toolkit/blob/b00ea0c39041d5cd1eec5e0e02db27c4b05c6cd4/dm/templates/healthcheck/examples/healthcheck.yaml#L8-L18).

As mentioned in step 3, the original sample config file expects URLs for an existing healthcheck and instance group resource. Since we are actually going to create these resources in the same deployment, there is no URL to provide. Instead of providing an URL we can use the `selfLink` property to refer to a resource that is still to be created. The syntax is typically `$(ref.<RESOURCE_NAME>.selfLink)`. 

4. Go ahead and name the resources and add the corresponding `selfLink` references. Note from the [outputs](https://github.com/GoogleCloudPlatform/cloud-foundation-toolkit/blob/b00ea0c39041d5cd1eec5e0e02db27c4b05c6cd4/dm/templates/managed_instance_group/managed_instance_group.py.schema#L999-L1001) described in the managed instance group schema, we can determine what the correct `selfLink` is. The `$(ref.<RESOURCE_NAME>.selfLink)` will actually refer to the instance group manager, whereas for the the instance group itself we need to use `$(ref.<RESOURCE_NAME>.instanceGroupSelfLink)`.
 
5. The final config should look as follows:

```sh
imports:
  - path: templates/external_load_balancer/external_load_balancer.py
    name: external_load_balancer.py
  - path: templates/healthcheck/healthcheck.py
    name: healthcheck.py  
  - path: templates/managed_instance_group/managed_instance_group.py
    name: managed-instance-group.py

resources:
  - name: example-http-elb
    type: external_load_balancer.py
    properties:
      portRange: 80
      backendServices:
        - resourceName: default-backend-service
          sessionAffinity: GENERATED_COOKIE
          affinityCookieTtlSec: 1000
          portName: http
          healthCheck: $(ref.my-healthcheck.selfLink)
          backends:
            - group: $(ref.my-instance-group.instanceGroupSelfLink)
              balancingMode: UTILIZATION
              maxUtilization: 0.8
      urlMap:
        defaultService: default-backend-service
  - name: my-healthcheck
    type: healthcheck.py
    properties:
      healthcheckType: HTTP
  - name: my-instance-group
    type: managed-instance-group.py
    properties:
      region: europe-west4
      autoscaler:
        cpuUtilization:
          utilizationTarget: 0.7
        minSize: 1
      targetSize: 3
      instanceTemplate:
        diskImage: projects/ubuntu-os-cloud/global/images/family/ubuntu-1804-lts
        networks:
          - network: projects/automation-247911/global/networks/cft-qwiklab101            
            subnetwork: projects/automation-247911/regions/europe-west4/subnetworks/qwiklab01            
            accessConfigs:
              - type: ONE_TO_ONE_NAT
        machineType: f1-micro
```

6. Create your deployment where 'my-external-load-balancer-deployment' is the name we have given the deployment:

```sh
gcloud deployment-manager deployments create my-external-load-balancer-deployment --config my-external-load-balancer.yaml
```

7. To clean up your deployment:
```sh
gcloud deployment-manager deployments delete my-external-load-balancer-deployment
```

### Next: Wrapping up

## Wrapping up

<walkthrough-conclusion-trophy/>

Congratulations! You've completed the Step-by-Step Walkthrough of Cloud Foundation Toolkit 101 using Deployment Manager!

You learned skills such as:

+   Using existing CFT templates to deploy resources
+   Modifying example config files and setting template properties
+   Combining multiple templates in a single deployment
+   Using references to resources still to be created

### What's Next

Here are some areas to explore as you learn more details about specific Deployment Manager functions:

+   [Explore more complex tutorials](https://cloud.google.com/deployment-manager/docs/tutorials)
+   Page on metadata and startup scripts
+   [Learn about available resource types](https://cloud.google.com/deployment-manager/docs/configuration/supported-resource-types) 
+   [Read the environment variables documentation](https://cloud.google.com/deployment-manager/docs/configuration/templates/use-environment-variables)
+   [Read about importing Python libraries](https://cloud.google.com/deployment-manager/docs/configuration/templates/import-python-libraries)
+   [Understand guidelines for preparing updates](https://cloud.google.com/deployment-manager/docs/deployments/updating-deployments) 

