# Sample HELM chart for Cisco Thousand Eye Agent

This is a sample repo for deploying Cisco Thousand Eye Agent containers in a kubernetes cluster via HELM. We have exposed several custom values which you can optionally change at run-time. The only requirement is that you insert your Thousand Eyes account token ID (in base64 format). By default, this chart will create a kubernetes secret object with your account token information and then it will create a kubernetes deployment object which instantiates a pod with the agent container. By default no incoming ports are opened for incoming test streams (that is an advanced setting described below). Also, by default, all agent logs and config files are mounted in a kubernetes local emptyDir on the host running the pod, these files are removed when the HELM chart is deleted. This action can also can be changed in the advanced settings.

## Prerequisits

It is assumed you have a working kubernetes cluster supporting Helm version 3.0 and a helm client machine (we loaded the helm client on the kube master node). All testing has been done using kubernetes v1.20.5 and helm client v3.5.3. 

## Installing the HELM chart with the basic Default Settings

First get your Thousand Eye account token in base 64 format and save the output. Use **"echo -n 'insert token' | base64"**

example for token=12345ABCDEF
```
echo -n 12345ABCDEF | base64
MTIzNDVBQkNERUY=
```

### Installing the HELM chart:

The Thousand Eye Helm chart can be installed multiple ways. You can choose to use just the helm command line to set needed variables or you can point to a local override values file. We show both approaches below. Also, we are using the packaged helm chart (thousandeyesagent-0.1.0.tgz) in the below examples which has a <version> number. This version number will change in the future, so please insert the proper version number as needed. If you clone the complete repo, you can also point to the un-packaged helm chart (thousandeyesagent) if you are making local modifications. Lastly, we are using the ***"helm upgrade --install"*** command which has the benefit of supporting both upgrading an existing helm deployment or creating a new helm deployment if one does not exist. 

#### Method 1- Installing with Helm CLI
Run the HELM install command and set custom values in the command line with the "set" command. These values will overwrite the same values in the values.yaml file. The chart can either be extracted (packaged) or not. In the below example we are in the top level repo directory and installing the helm release "john" in the default namespace. 

```
helm upgrade --install <release name> thousandeyesagent-<version>.tgz -n <namespace> --set account_token=<base64 account token>
```

example:
```
helm upgrade --install john thousandeyesagent-0.1.0.tgz --set account_token=MTIzNDVBQkNERUY=
```
#### Method 2- Installing with local vlaues override file
Run the HELM install command and point to a custom values file (i.e. myvalues.yaml). These values will overwrite the same values in the chart's values.yaml file. The chart can either be extracted or not. A sample myvalues.yaml is included, but you can use any file name. In the below example we are in the top level repo directory.

```
helm upgrade --install <release name> thousandeyesagent-<0.1.0>.tgz -n <namespace> -f <custom_values_file>
```

example:
```
helm upgrade --install john thousandeyesagent-0.1.0.tgz --set -f myvalues.yaml 
```
where myvalues.yaml: 
```
account_token: MTIzNDVBQkNERUY=
```

Note: You can mix both of the options above such that you can specify command line parameters and also reference a file.

### Verify the deployment

First verify the helm chart has status "deployed".

```
[root@kubemaster ~]# helm ls
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
john                    default         1               2021-07-28 19:43:16.269720261 +0000 UTC deployed        thousandeyesagent-0.2.0 1.16.0
```

Next, verify that all of the kubernetes objects have been created, this would include a kubernetes deployment object, a pod object and a secret object. If using advanced settings (see below), you may also have a services object. 

```
[root@kubemaster ~]# kubectl get deployments.apps
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
john-thousandeyes-k8s     1/1     1            1           11d

[root@kubemaster ~]# kubectl get pods
NAME                                       READY   STATUS    RESTARTS   AGE
john-thousandeyes-k8s-5ff95c9fcc-lkf4b     1/1     Running   0          11d

[root@kubemaster ~]# kubectl get secrets 
NAME                                           TYPE                                  DATA   AGE
john-secret                                    Opaque                                1      11d
sh.helm.release.v1.john.v1                     helm.sh/release.v1                    1      11d
```

## Installing the HELM chart with Advanced Settings

### Changing the image in the override file. 

By default the latest thousandeye agent container will be pulled. If you desire an different version, add the image version to your override file:

example:
```
image:
  repository: thousandeyes/enterprise-agent 
  tag: 1:15 
```

### Picking the specific host (node) to run the probe on

When setnode is enabled=true, a specific k8s cluster node can be selected to run the thousand eye agent pod. When enabled, set the nodename to match the desired node to deploy the pod. By default this is set to false.

example using set command:
```
helm upgrade --install john thousandeyesagent-0.1.0.tgz --set setnode.enabled=true --set setnode.nodename=kubenode1 --set account_token=MTIzNDVBQkNERUY= 
```
example using file:
```
helm upgrade --install john thousandeyesagent-0.1.0.tgz -f myvalues.yaml
```
where myfile.yaml:
```
account_token: MTIzNDVBQkNERUY=

setnode:
  enabled: true
  nodename: kubenode1
```
### Persisting agent data after deletion and for re-deploys

To persist data to a specific directory on the node (where the node can be set via setnode.nodename above), set the persistence flag. 

When persistence is enabled=false (default), kubernetes will mount an EmptyDir for all log and agent data (which is accessible at /var/lib/kubelet/pods/{podid}/volumes/kubernetes.io~empty-dir/ on the specific host, while the pod is created). When the pod/deployment/helm is deleted the EmptyDir is cleaned of it's data. You should then delete the agent from the thousand eye dashbaord. If you re-install with the same helm release name, a new agent will be created using a new log/data file location and the thousand eye dashboard will learn of a new agent (even if the previous agent was not deleted).

When persistence is enabled=true, set the directory which will be mounted on the host, such as "/opt/thousandeyes/" (directory does not need to exist first). Log and agent data would then be saved at that path within a release name directory that is autocreated. This data will persist if the Deployment/HELM chart is deleted. If the same release named chart/deployment is now re-installed, the previous agent data will be picked up and the same agent name already on the the thousand eye dashboard will be used, including all existing test settings.

example myvalues.yaml:
```
persistence:
  enabled: true
  directory_path: /opt/thousandeyes/
```
### Enabling Host Networking

This is an advanced topic, but if you would like the agent to use the IP of the host node interface (and not recieve an IP from the CNI overlay network), then enable this flag. The probe/pod will use the routing table of the host node to select an exiting interface and then use that IP as the source IP. Note if you have multiple NICs on the host this can get very complex. 

example:
```
host_networking_mode:
  enabled: true
```
 
### Opening a incoming port on the agent pod

To open a specific port on the agent to receive "outside-to-inside" test traffic, where the traffic is initiated from the outside (note this is not required if the test source is initiated from the inside (i.e. from the agent)), set allow_outside_initiated_traffic=true. Then, if host_networking_mode is disabled, set the **unique** (cluster wide) port number (from nodePort allowable range 30000-32767) that the incomming traffic will be received on. This will create a kubernetes NodePort service object. If host_networking_mode is enabled, then no kubernetes service object is created as any IP on the host/node can be used but a **unique** (per node) port number is requried (no restriction on allowable range).  

example myvalues.yaml:
```
allow_outside_initiated_traffic:
  enabled: true
  port: 30000
```

When settign up a test stream on the Thousand Eye dashboard, select this agent to receive with port 30000. 

### Tips on Target IP address selection for probe tests where NATs are involved

When settign up a test stream on Thousand Eye dashboard, you will pick a "Target Agent" and a "Source Agent". You will set up the target "Server port" number under the Advanced Settings to match the port configured in your "allow_outside_initiated_traffic" setting. But where do we select the Target IP address to use? If you go to the Agent Settings screen and select your probe, you will see the Private and Public IPs of your probe. In the case of containers and kubernetes, you will see the pods CNI IP as the private IP if host_networking_mode is disabled and you will see the k8s node/host IP if host_networking_mode is enabled. Also, if host_networking is disabled and you are running test traffic targeting the agent from outside of your cluster, then the kubernetes Nodeport service object is used. In this case, the private IP still is shown as the CNI pod IP, but the target IP for tests needs to be set to any of the cluster node IPs (the NodePort service can use any node IP). Regardless, when running tests with private networks with containers, across NATs, you will likley need to set the "Target for Tests" IP address in the agent advanced settings screen. Depending on your test scenario, experimentation to set the right IP is required.


   

