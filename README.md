# Add Windows nodes to IBM Cloud Private Linux Cluster



**In this page**
- What you will accomplish
- Preparing the Linux Master
- Preparing a Windows node
- Network topology
- Running a Sample Service

Since the release of Kubernetes 1.9 and Windows Server version 1709, it's the great opportunity to let Windows networking be facilitated so Windows nodes can be added into IBM Cloud Private(ICP) Linux cluster.

This page serves as a guide for getting started joining a brand new Windows node to an existing ICP Linux cluster.

> &#9888; Tip:
> If you would like to deploy a cluster on Azure, the open source ACS-Engine tool makes this easy. A step by step [walkthrough](#https://github.com/Azure/acs-engine/blob/master/docs/kubernetes/windows.md) is available.



## What you will accomplish
- [x] Configured a Linux master node
- [x] Joined a Windows worker node to it
- [ ] Prepared our network topology
- [ ] Deployed a sample Windows service
- [ ] Covered common problems and mistakes



## Prepare the Linux Master
Regardless of whether you followed [the instructions](#https://www.ibm.com/support/knowledgecenter/SS2L37_3.1.0.0/cam_planning.html) or already have an existing cluster, only one thing is needed from the Linux master is Kubernetes' certificate configuration. This could be in /opt/ibm-cloud-private-\<version\>/cluster/cfc-certs/, or elsewhere depending on your setup.



## Prepare the Windows node
> &#9997; Note:
> All code snippets in Windows sections are to be run in elevated PowerShell.

Kubernetes uses [Docker](#https://www.docker.com/) as its container orchestrator, so we need to install it. You can follow the [official Docs](#https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-docker/configure-docker-daemon#install-docker) instructions, the [Docker instructions](#https://store.docker.com/editions/enterprise/docker-ee-server-windows), or try these steps:
```
PS C:\> mkdir C:/k/
PS C:\> cd C:/k/
PS C:\k> Install-Module -Name DockerMsftProvider -Repository PSGallery -Force
PS C:\k> Install-Package -Name Docker -ProviderName DockerMsftProvider
PS C:\k> Restart-Computer -Force
```
If you are behind a proxy, the following PowerShell environment variables must be defined:
```
[Environment]::SetEnvironmentVariable("HTTP_PROXY", "http://<proxy_server>:80/", [EnvironmentVariableTarget]::Machine)
[Environment]::SetEnvironmentVariable("HTTPS_PROXY", "http://<proxy_server>:443/", [EnvironmentVariableTarget]::Machine)
```


## Network topology
There are multiple ways to make the virtual [cluster subnet](#https://docs.microsoft.com/en-us/virtualization/windowscontainers/kubernetes/getting-started-kubernetes-windows#cluster-subnet-def) routable. You can do following for Kubernetes:
- Configure host-gateway mode, setting static next-hop routes between nodes to enable pod-to-pod communication.
- Configure a smart top-of-rack (ToR) switch to route the subnet.
- Use a third-party overlay plugin such as Flannel (Windows support for Flannel is in beta).

### Create the "pause" image
Now that docker is installed, you need to prepare a "pause" image that's used by Kubernetes to prepare the infrastructure pods.
```
PS C:\k> docker pull microsoft/windowsservercore:1709
docker pull microsoft/windowsservercore:1709
1709: Pulling from microsoft/windowsservercore
5847a47b8593: Pull complete
a5b83e25f92a: Pull complete
Digest: sha256:1bd535b56b751a434bb7d70c488567674553d5c2ee53cfec3550cdde3add3073
Status: Downloaded newer image for microsoft/windowsservercore:1709

PS C:\k> docker tag microsoft/windowsservercore:1709 microsoft/windowsservercore:latestx

PS C:\k> docker build -t kubeletwin/pause .
Sending build context to Docker daemon  44.03kB
Step 1/2 : FROM microsoft/nanoserver
latest: Pulling from microsoft/nanoserver
bce2fbc256ea: Pull complete
b1b0c61be11f: Pull complete
Digest: sha256:543145e7387282a60b3d357cd7a9f2c697d52bc45496145f0dcd6bb570ca122e
Status: Downloaded newer image for microsoft/nanoserver:latest
 ---> 1381511ec012
Step 2/2 : CMD cmd /c ping -t localhost
 ---> Running in 35e04c0cfdf0x
 ---> 53c728c46f91
Removing intermediate container 35e04c0cfdf0
Successfully built 53c728c46f91
Successfully tagged kubeletwin/pause:latest
```

> &#9997; Note:
> We tag it as the :latest because the sample service you will be deploying later depends on it, though this may not actually be the latest Windows Server Core image available. It's important to be careful of conflicting container images; not having the expected tag can cause a docker pull of an incompatible container image, causing [deployment problems](#https://docs.microsoft.com/en-us/virtualization/windowscontainers/kubernetes/common-problems#when-deploying-docker-containers-keep-restarting).

### Download Kubernetes binaries for windows
In the meantime while the pull occurs, download the following client-side binaries from Kubernetes:
- kubectl.exe
- kubelet.exe
- kube-proxy.exe

You can download these from the links in the CHANGELOG.md file of the latest 1.9 release. As of this writing, that is [1.9.1](#https://github.com/kubernetes/kubernetes/releases/tag/v1.9.1), and the Windows binaries are [here](#https://storage.googleapis.com/kubernetes-release/release/v1.9.1/kubernetes-node-windows-amd64.tar.gz).Extract the archive and place the binaries in C:\k\.

In order to make the kubectl command available outside of the C:\k\ directory, modify the PATH environment variable:
```
PS C:\k> $env:Path += ";C:\k"
```
If you would like to make this change permanent, modify the variable in machine target:
```
PS C:\k> [Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\k", [EnvironmentVariableTarget]::Machine)
```
Verify binary execution works normally:
```
PS C:\k> kubectl version
Client Version: version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.1", GitCommit:"3a1c9449a956b6026f075fa3134ff92f7d55f812", GitTreeState:"clean", BuildDate:"2018-01-04T11:52:23Z", GoVersion:"go1.9.2", Compiler:"gc", Platform:"windows/amd64"}
```

### Config ICP Linux cluster certificates on Windows node
From ICP Linux node, get below files:
```
~/.kube/config
/opt/ibm-cloud-private-<version>/cluster/cfc-certs/kubecfg.crt
/opt/ibm-cloud-private-<version>/cluster/cfc-certs/kubecfg.key
```
On Windows nodes, put the files into corresponding path:
```
PS C:\k> mkdir etc\cfc\conf
PS C:\k> cp config C:/k/.
PS C:\k> cp kubecfg.key kubecfg.crt etc\cfc\conf\
```
Add the path to PATH environment variable
```
PS C:\k> $env:KUBECONFIG="C:\k\config"
PS C:\k> [Environment]::SetEnvironmentVariable("KUBECONFIG", "C:\k\config", [EnvironmentVariableTarget]::User)
```
Check ICP Linux cluster info from Windows node
```
PS C:\k> kubectl cluster-info
Kubernetes master is running at https://<ICP_master_ip>:8001
catalog-ui is running at https://<ICP_master_ip>:8001/api/v1/namespaces/kube-system/services/catalog-ui:catalog-ui/proxy
Heapster is running at https://<ICP_master_ip>:8001/api/v1/namespaces/kube-system/services/heapster/proxy
image-manager is running at https://<ICP_master_ip>:8001/api/v1/namespaces/kube-system/services/image-manager:image-manager/proxy
CoreDNS is running at https://<ICP_master_ip>:8001/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
metrics-server is running at https://<ICP_master_ip>:8001/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
platform-ui is running at https://<ICP_master_ip>:8001/api/v1/namespaces/kube-system/services/platform-ui:platform-ui/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
If you are receiving a connection error,
```
Unable to connect to the server: dial tcp [::1]:8080: connectex: No connection could be made because the target machine actively refused it.
```
check if the configuration has been discovered properly:
```
PS C:\k> kubectl config view
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://<ICP_master_ip>:8001
  name: icp-cluster1
contexts:
- context:
    cluster: <ICP_cluster_name>
    user: admin
  name: ""
current-context: ""
kind: Config
preferences: {}
users:
- name: admin
  user:
    client-certificate: /etc/cfc/conf/kubecfg.crt
    client-key: /etc/cfc/conf/kubecfg.key
```

### prepare scripts for cluster join
The node is now ready to join the cluster. In two separate, elevated PowerShell windows, run these scripts (in this order).
```
PS C:\k> [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
PS C:\k> wget https://github.com/Microsoft/SDN/archive/master.zip -o master.zip
PS C:\k> Expand-Archive master.zip -DestinationPath master
PS C:\k> mv master/SDN-master/Kubernetes/windows/* C:/k/
PS C:\k> rm -recurse -force master,master.zip
```

## config network as Host-Gateway mode
on all ICP Linux nodes:
```
PS C:\k> CLUSTER_PREFIX="10.1"
PS C:\k> iptables -t nat -F
PS C:\k> iptables -t nat -A POSTROUTING ! -d $CLUSTER_PREFIX.0.0/16 -m addrtype ! --dst-type LOCAL -j MASQUERADE
PS C:\k> sysctl -w net.ipv4.ip_forward=1
PS C:\k> route add -net $CLUSTER_PREFIX.0.0 netmask 255.255.0.0 dev eth0
PS C:\k> route add -net $CLUSTER_PREFIX.1.0 netmask 255.255.255.0 gw $CLUSTER_PREFIX.1.2 dev eth0
```

on Windows node:
```
PS C:\k> $url = "https://raw.githubusercontent.com/Microsoft/SDN/master/Kubernetes/windows/AddRoutes.ps1"
PS C:\k> Invoke-WebRequest $url -o AddRoutes.ps1
PS C:\k> ./AddRoutes.ps1 -MasterIp <ICP_master_ip>
```

## join Windows node to ICP Linux cluster
Run below scripts for cluster join:
```
PS C:\k> $url = "https://raw.githubusercontent.com/Microsoft/SDN/master/Kubernetes/windows/start-kubelet.ps1"
PS C:\k> Invoke-WebRequest $url -o start-kubelet.ps1
PS C:\k> ./start-kubelet.ps1 -ClusterCidr 10.1.0.0/16

PS C:\k> $url = "https://raw.githubusercontent.com/Microsoft/SDN/master/Kubernetes/windows/start-kubeproxy.ps1"
PS C:\k> Invoke-WebRequest $url -o start-kubeproxy.ps1
PS C:\k> ./start-kubeproxy.ps1
```
The Windows will be visible in cluster:
```
PS C:\k> kubectl get nodes
NAME		STATUS     	ROLES                                 		AGE	VERSIONs
<ICP_master_ip>	Ready		etcd,management,master,proxy,worker	9d	v1.11.1+icp-ee
vspw2016x64v	NotReady	<none>                                		37m	v1.9.1
```

### start kubelet and kubeproxy on Windows
Copy crt, key and config files to Windows as below:
```
C:\K\ETC
└───cfc
    ├───conf
    │       kubecfg.crt
    │       kubecfg.key
    │
    ├───kube-proxy
    │   │   kube-proxy-config
    │   │   kube-proxy.crt
    │   │   kube-proxy.key
    │   │
    │   └───etc
    │       └───cfc
    │           └───kube-proxy
    │                   kube-proxy.crt
    │                   kube-proxy.key
    │
    ├───kubelet
    │   │   ca.crt
    │   │   kubelet-config
    │   │   kubelet-service-config
    │   │   kubelet.crt
    │   │   kubelet.key
    │   │
    │   └───etc
    │       └───cfc
    │           └───kubelet
    │                   kubelet.crt
    │                   kubelet.key
    │
    └───pods
            kube-proxy.json
```

> &#9997; Note:
> seems bug for kubelet on Windows, so there're redundancies for hierachy, will update when ICP team solved this.

Run below commands to start kubelet and kube proxy
```
PS C:\k> .\kubelet.exe --feature-gates Accelerators=true,PersistentLocalVolumes=true,VolumeScheduling=true,ExperimentalCriticalPodAnnotation=true --allow-privileged=true --cadvisor-port=0 --docker-disable-shared-pid --require-kubeconfig --kubeconfig=C:\k\etc\cfc\kubelet\kubelet-config --read-only-port=0 --client-ca-file=C:\k\etc\cfc\kubelet\ca.crt --authentication-token-webhook --anonymous-auth=false --network-plugin=cni --pod-manifest-path=C:\k\etc\cfc\pods --cluster-dns=10.0.0.10 --cluster-domain=cluster.local --hostname-override=<Windows_node_ip> --node-ip=<Windows_node_ip> --pod-infra-container-image=kubeletwin/pause:latest --keep-terminated-pod-volumes=false --resolv-conf=C:\Windows\System32\drivers\etc\resolv.conf

PS C:\k> .\kube-proxy.exe --kubeconfig=C:\k\etc\cfc\kube-proxy\kube-proxy-config --hostname-override=<Windows_node_ip> --cluster-cidr=10.1.0.0/16 --proxy-mode=kernelspace
```

You may noticed the Windows node status is NotReady.
```
PS C:\k> kubectl get nodes
NAME		                STATUS     ROLES                                 AGE   VERSIONs
<ICP_master_ip>       Ready		    etcd,management,master,proxy,worker   9d    v1.11.1+icp-ee
<Windows_node_name>   NotReady   <none>                                37m	  v1.9.1
```
That's because ICP is using Calico as network plugin, in the next, we investigate deploying Calico on Windows so the newly joined Windows node can be in Ready status for sample service running.
