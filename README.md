<style>
  .hint {
    padding: 10px 10px 10px 10px;
    margin: 10px 10px 10px 0px;
    background-color: rgb(214,246,255);
    border-radius: 3px;
  }
  .code {
    padding: 10px 10px 10px 10px;
    margin: 10px 10px 10px 0px;
    background-color: rgb(3,37,85);
    color: rgb(220,213,215);
    border-radius: 3px;
    font-size: 13px;
  }
  .highlight {
    background-color: rgb(249,249,249);
    padding: 3px 5px 3px 5px;
    border-radius: 3px;
  }
</style>


#Add Windows nodes to IBM Cloud Private Linux Cluster



**In this page**
- [What you will accomplish](#header)
- Preparing the Linux Master
- Preparing a Windows node
- Network topology
- Running a Sample Service

Since the release of Kubernetes 1.9 and Windows Server version 1709, it's the great opportunity to let Windows networking be facilitated so Windows nodes can be added into IBM Cloud Private(ICP) Linux cluster.

This page serves as a guide for getting started joining a brand new Windows node to an existing ICP Linux cluster.

<div class="hint"><div>&#9888; Tip:</div>
If you would like to deploy a cluster on Azure, the open source ACS-Engine tool makes this easy. A step by step <a href="https://github.com/Azure/acs-engine/blob/master/docs/kubernetes/windows.md">walkthrough</a> is available.
</div>



##<a name="header"></a>What you will accomplish(updates incoming)
- [x] Configured a Linux master node
- [x] Joined a Windows worker node to it
- [ ] Prepared our network topology
- [ ] Deployed a sample Windows service
- [ ] Covered common problems and mistakes



##Prepare the Linux Master
Regardless of whether you followed [the instructions](#https://www.ibm.com/support/knowledgecenter/SS2L37_3.1.0.0/cam_planning.html) or already have an existing cluster, only one thing is needed from the Linux master is Kubernetes' certificate configuration. This could be in /opt/ibm-cloud-private-\<version\>/cluster/cfc-certs/, or elsewhere depending on your setup.



##Prepare the Windows node:
<div class="hint"><div>&#9997; Note:</div>
All code snippets in Windows sections are to be run in elevated PowerShell.
</div>

Kubernetes uses [Docker](#https://www.docker.com/) as its container orchestrator, so we need to install it. You can follow the [official Docs](#https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-docker/configure-docker-daemon#install-docker) instructions, the [Docker instructions](#https://store.docker.com/editions/enterprise/docker-ee-server-windows), or try these steps:
<div class="code">
PS C:\> mkdir C:/k/<br/>
PS C:\> cd C:/k/<br/>
PS C:\k> Install-Module -Name DockerMsftProvider -Repository PSGallery -Force<br/>
PS C:\k> Install-Package -Name Docker -ProviderName DockerMsftProvider<br/>
PS C:\k> Restart-Computer -Force
</div>
If you are behind a proxy, the following PowerShell environment variables must be defined:
```
[Environment]::SetEnvironmentVariable("HTTP_PROXY", "http://<proxy_server>:80/", [EnvironmentVariableTarget]::Machine)
[Environment]::SetEnvironmentVariable("HTTPS_PROXY", "http://<proxy_server>:443/", [EnvironmentVariableTarget]::Machine)
```



##Network topology

There are multiple ways to make the virtual [cluster subnet](#https://docs.microsoft.com/en-us/virtualization/windowscontainers/kubernetes/getting-started-kubernetes-windows#cluster-subnet-def) routable. You can do following for Kubernetes:
- Configure host-gateway mode, setting static next-hop routes between nodes to enable pod-to-pod communication.
- Configure a smart top-of-rack (ToR) switch to route the subnet.
- Use a third-party overlay plugin such as Flannel (Windows support for Flannel is in beta).

But What ICP using is Calico, we will talk about it on incoming updates.

###Create the "pause" image

Now that docker is installed, you need to prepare a "pause" image that's used by Kubernetes to prepare the infrastructure pods.
<div class="code">
PS C:\k> docker pull microsoft/windowsservercore:1709<br/>
docker pull microsoft/windowsservercore:1709<br/>
1709: Pulling from microsoft/windowsservercore<br/>
5847a47b8593: Pull complete<br/>
a5b83e25f92a: Pull complete<br/>
Digest: sha256:1bd535b56b751a434bb7d70c488567674553d5c2ee53cfec3550cdde3add3073<br/>
Status: Downloaded newer image for microsoft/windowsservercore:1709<br/>
<br/>
PS C:\k> docker tag microsoft/windowsservercore:1709 microsoft/windowsservercore:latest<br/>
<br/>
PS C:\k> docker build -t kubeletwin/pause .<br/>
Sending build context to Docker daemon  44.03kB<br/>
Step 1/2 : FROM microsoft/nanoserver<br/>
latest: Pulling from microsoft/nanoserver<br/>
bce2fbc256ea: Pull complete<br/>
b1b0c61be11f: Pull complete<br/>
Digest: sha256:543145e7387282a60b3d357cd7a9f2c697d52bc45496145f0dcd6bb570ca122e<br/>
Status: Downloaded newer image for microsoft/nanoserver:latest<br/>
 ---> 1381511ec012<br/>
Step 2/2 : CMD cmd /c ping -t localhost<br/>
 ---> Running in 35e04c0cfdf0<br/>
 ---> 53c728c46f91<br/>
Removing intermediate container 35e04c0cfdf0<br/>
Successfully built 53c728c46f91<br/>
Successfully tagged kubeletwin/pause:latest
</div>

<div class="hint"><div>&#9997; Note:</div>
We tag it as the <span class="highlight">:latest</span> because the sample service you will be deploying later depends on it, though this may not actually be the latest Windows Server Core image available. It's important to be careful of conflicting container images; not having the expected tag can cause a <span class="highlight">docker pull</span> of an incompatible container image, causing <a href="https://docs.microsoft.com/en-us/virtualization/windowscontainers/kubernetes/common-problems#when-deploying-docker-containers-keep-restarting">deployment problems</a>.
</div>

###Download Kubernetes binaries for windows
In the meantime while the pull occurs, download the following client-side binaries from Kubernetes:
- kubectl.exe
- kubelet.exe
- kube-proxy.exe
