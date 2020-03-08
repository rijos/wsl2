# Running K8s on WSL2
WSL2 uses a thin hypervisor to host the linux kernel providing full linux binary compatibility allowing us 
to use docker and k8s. WSL2 lacks components such as systemd.

## Preparing Windows
WSL2 requires Windows version v2004 and above, as of writing you can attain this version using the Windows insider program. 
To subscribe go to Settings -> Update & Security -> Windows Insider Program. 
I'd recommend using the slow channel but remember, always backup your data first and 
keep in mind that the insider program enables full telemetry.

## Enabling WSL2
Before or after upgrading Windows you must enable the following Windows features:
* Windows Subsystems for Linux 
* Virtual Machine Platform

WSL2 uses parts of Hyper-V but does not require Hyper-V to be installed, you can run this on Windows Home edition as well. 
To enable WSL2 by default use `wsl --set-default-version 2`. 
if you want to upgrading existing images use `wsl <distro> --set-version 2`,
i've read reports of users bricking their images so be careful.

## Installing ubuntu 18.04
You can search for ubuntu in the Microsoft Store

**Note if this is the first installed distribution make sure to enable it by typing `wsl` in the command prompt, else it might not appear in vscode.

## Installing vscode 
Grab the latest version at https://code.visualstudio.com/ and install the _Remote - WSL extension_

## Installing rust
Execute the command below in the WSL2 environment, you can enter this environment by typing `wsl` in de command prompt or using the vscode terminal.
```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
``` 

## Installing docker
```
sudo apt install docker.io
sudo service docker start
```

## Installing kubectl 
```
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmox +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

## Installing Kind (Dockerized kubernetes)
Kind requires go:
```
sudo add-apt-repository ppa:longsleep/golang-backports
sudo apt update
sudo apt install golang-go
```

```
GO111MODULE="on" go get sigs.k8s.io/kind@v0.7.0
sudo mv ~/go/bin/kind /usr/local/bin/kind
```

## Create your first cluster
```
sudo kind create cluster
```
Validate if your cluster is alive `sudo kubectl get all`

To remove the 'sudo' requirement for kubectl
`sudo chown -R ubuntu:ubuntu ~/.kube/` (if ubuntu is your username in wsl)

## Test deployment
Use the following commands:
```
sudo cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
        authorization-mode: "AlwaysAllow"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF
```
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/provider/baremetal/service-nodeport.yaml

kubectl patch deployments -n ingress-nginx nginx-ingress-controller -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx-ingress-controller","ports":[{"containerPort":80,"hostPort":80},{"containerPort":443,"hostPort":443}]}],"nodeSelector":{"ingress-ready":"true"},"tolerations":[{"key":"node-role.kubernetes.io/master","operator":"Equal","effect":"NoSchedule"}]}}}}'

kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/usage.yaml
```

Validate if the  pods are running `kubectl get pods`, once online we can send requests:
```
curl localhost/foo
curl localhost/bar
```

## Shutdown WSL
The virtual machine will keep running in the background in a process called vmmem. To shutdown the virtual machine use `wsl --shutdown`.

## Sources
- https://kind.sigs.k8s.io/docs/user/using-wsl2/
- https://docs.microsoft.com/en-us/windows/wsl/wsl2-install
