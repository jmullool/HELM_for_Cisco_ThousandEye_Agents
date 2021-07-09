# Sample HELM chart for Cisco Thousand Eye Agent

This is a sample repo for deploying Cisco Thousand Eye Agent containers in a kubernetes cluster via HELM. We have exposed several custom values which you can optionally change at run-time. The only requirement is that you insert your Thousand Eye account token ID (in base64 format). By default, this chart will create a kubernetes secret object with your account token information and then it will create a kubernetes deployment which instantiates a pod with the agent container. By default no incoming ports are opened for incoming test streams (that is an advanced setting described below). Also, by default, all agent logs and config files are mounted in a kubernetes local emptyDir on the host running the pod, these files are removed when the HELM chart is deleted. This action can also can be changed in the advanced settings.

## Installing the HELM chart with the basic Default Settings

Get your account token in base 64 format and save the output. Use "echo -n '<insert token>' | base64'

example:
>>>echo -n 12345ABCDEF | base64
>>>MTIzNDVBQkNERUY=

### After cloning the repo, the HELM chart can be installed two ways:

(1) run the HELM install command and set custom values in the command line. These values will overwrite the same values in the values.yaml file. The chart can either be extracted or not. In the below example we are in the top level repo directory.

>>>**helm upgrade --install <release name> thousandeyesagent-0.1.0.tgz -n <namespace> --set account_token=<base64 account token>**

example:
>>>helm upgrade --install john thousandeyesagent-0.1.0.tgz -n <namespace> --set account_token=MTIzNDVBQkNERUY=

(2) run the HELM install command and set the custom values in a file (i.e. myvalues.yaml). These values will overwrite the same values in the values.yaml file. The chart can either be extracted or not. A sample myvalues.yaml is included, but you can use any file name. In the below example we are in the top level repo directory.

helm upgrade --install <release name> thousandeyesagent-0.1.0.tgz -n <namespace> -f <custom_values_file>

example:
>>>helm upgrade --install john thousandeyesagent-0.1.0.tgz --set -f myvalues.yaml 

where myvalues.yaml: 

>>>account_token: MTIzNDVBQkNERUY=

----------------------------------------------------------------------------------------------------------------

## Installing the HELM chart with Advanced Settings

### Picking the specific host (node) to run the probe on

When setnode is enabled=true, a specific k8s cluster node can be selected to run the thousand eye agent pod. When enabled, set the nodename to match the desired node to deploy the pod. By default this is set to false.

example:
>>>setnode:
>>>  enabled: true
>>>  nodename: kubenode1

### Persisting agent data after deletion and for re-deploys

To persist data to a specific directory on the node (where the node can be set via setnode.nodename above), set the persistence flag. 

When persistence is enabled=false (default), kubernetes will mount an EmptyDir for all log and agent data (which is accessible at /var/lib/kubelet/pods/{podid}/volumes/kubernetes.io~empty-dir/ on the specific host, while the pod is created). When the pod/deployment/helm is deleted the EmptyDir is cleaned of it's data. You should then delete the agent from the thousand eye dashbaord. If you re-install with the same helm release name, a new agent will be created using a new log/data file location and the thousand eye dashboard will learn of a new agent (even if the previous agent was not deleted).

When persistence is enabled=true, set the directory which will be mounted on the host, such as "/opt/thousandeyes/" (directory does not need to exist first). Log and agent data would then be saved at that path within a release name directory that is autocreated. This data will persist if the Deployment/HELM chart is deleted. If the same release named chart/deployment is now re-installed, the previous agent data will be picked up and the same agent name already on the the thousan eye dashboard will be used, including all test settings.

example:
>>>persistence:
>>>  enabled: true
>>>  directory_path: /opt/thousandeyes/


>>>persistence:
>>>  enabled: false
>>>  directory_path: {}

### Opening a incoming port on the agent pod

To open a specific port on the agent to receive "outside-to-inside" test traffic, where the traffic is initiated from the outside (note this is not required if the test is initiated from the inside (i.e. from the agent)), set allow_outside_initiated_traffic=true. Then set the unique (cluster wide) port number (from nodePort allowable range) that the incomming traffic will be received on. This will create a nodePort service and the port number can not overlap with a pre-existing service using the same port. 

example:
>>>allow_outside_initiated_traffic:
>>>  enabled: true
>>>  port: 30000

>>>allow_outside_initiated_traffic:
>>>  enabled: false
>>>  port: {}
