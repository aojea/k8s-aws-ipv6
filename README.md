# k8s-aws-ipv6

Deploy a kubernetes ipv6 cluster on AWS

Cloudformation base template obtained from:

https://gist.githubusercontent.com/milesjordan/d86942718f8d4dc20f9f331913e7367a/raw/3227d1dff6e9df8a79f395cdb2c62af9899ed777/vpc.yaml

The goal is to run an IPv6 conformance cluster in aws : 1 control-plane + 2 workers

It's out of scope to provide a mechanism to deploy clusters in AWS, there
are much better projects to do that:

* kops
* kubespray
* cluster-api


## Instances

We are going to use the predefined instances created by the Cluster API project on

https://github.com/kubernetes-sigs/image-builder/blob/master/images/capi/Makefile


- [ ] Add OpenSUSE Leap Support
- [ ] Enable IPv6 forwarding (modules?)
`sysctl -w net.ipv6.conf.all.forwarding=1`
- [x] Use latest CNI plugins 0.8.5

## Deployment

1. Create VPC with IPv6 enabled
2. Create Subnet/s with IPv6 autoassign by default
3. Configure Security Groups

NodePorts
Pod and Service Subnets
Host to Host

4. Add Internet Gateway
5. Configure route table
6. Create Master and Worker IAM roles

https://github.com/kubernetes/cloud-provider-aws

6. Spawn instances

ami-02c520e711f7d01cd capa-ami-ubuntu-18.04-1.16.2-00-1579044236 

7. disable the SrcDestCheck attribute
8. Tag instances
kubernetes.io/cluster/kubernetes = owned
Name = k8s


## Kubeadm

Obtain IPv6 address 

```sh
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"` \
&& curl -H "X-aws-ec2-metadata-token: $TOKEN" –v http://169.254.169.254/latest/meta-data/network/interfaces/macs/mac-address/ipv6s
```

```sh
MACADDRESS=`curl http://169.254.169.254/latest/meta-data/network/interfaces/macs`
HOST_IPV6=`curl http://169.254.169.254/latest/meta-data/network/interfaces/macs/${MACADDRESS}ipv6s`
export HOST_IPV6
```


Set hostname

```
hostnamectl set-hostname $(curl http://169.254.169.254/latest/meta-data/local-hostname)
```


Configure Kubeadm for control planes

```yaml
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
apiServer:
  certSANs:
  - localhost
  - ::1
clusterName: test
controlPlaneEndpoint: "[$HOST_IPV6]:6443"
controllerManager:
  extraArgs:
    configure-cloud-routes: "false"
    bind-address: "::"
kubernetesVersion: v1.16.2
networking:
  podSubnet: fd00:10:244::/64
  serviceSubnet: fd00:10:96::/112
scheduler:
  extraArgs:
    address: "::"
    bind-address: "::1"
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
metadata:
  name: config
# we use a well know token for TLS bootstrap
bootstrapTokens:
- token: abcdef.0123456789abcdef
# we use a well know port for making the API server discoverable inside docker network.
# from the host machine such port will be accessible via a random local port instead.
localAPIEndpoint:
  advertiseAddress: $HOST_IPV6
  bindPort: 6443
nodeRegistration:
  kubeletExtraArgs:
    fail-swap-on: "false"
    node-ip: $HOST_IPV6
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
metadata:
  name: config
# configure ipv6 addresses in IPv6 mode
address: "::"
healthzBindAddress: "::"
# disable disk resource management by default
# kubelet will see the host disk that the inner container runtime
# is ultimately backed by and attempt to recover disk space. we don't want that.
imageGCHighThresholdPercent: 100
evictionHard:
  nodefs.available: "0%"
  nodefs.inodesFree: "0%"
  imagefs.available: "0%"
```

substitue env variables:

```sh
envsubst < kubeadm.yaml > config.yaml
```

Configure control-plane node

```
kubeadm init --config config.yaml --ignore-preflight-errors=all -v7
```



configure `--node-ip=$HOST_IPV6` in /var/lib/kubelet/kubeadm-flags.env

and  `systemctl restart kubelet`


Worker nodes:

```yaml
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: JoinConfiguration
discovery:
  bootstrapToken:
    apiServerEndpoint: '[CONTROL_PLANEIP]:6443'
    token: abcdef.0123456789abcdef
    unsafeSkipCAVerification: true
nodeRegistration:
  criSocket: /run/containerd/containerd.sock
  kubeletExtraArgs:
    fail-swap-on: "false"
    node-ip: $HOST_IPV6
```

configure `--node-ip=$HOST_IPV6` in /var/lib/kubelet/kubeadm-flags.env

and  `systemctl restart kubelet`


Check everything is working:



## CNI Plugin

Assuming a flat network we can use [Kindnet]() as CNI plugin


Download docker image:

https://www.dropbox.com/s/yqxarzf8tj83y02/kindnetd.tar.gz

`wget https://www.dropbox.com/s/yqxarzf8tj83y02/kindnetd.tar.gz -O- | gunzip | ctr --namespace=k8s.io images import --no-unpack -`

Download installation manifest:
https://www.dropbox.com/s/ma35vprq69h3ikw/install-kindnet.yaml

`kubectl apply -f https://www.dropbox.com/s/ma35vprq69h3ikw/install-kindnet.yaml`

Can't use latest for local images

`sed -i 's/aojea\/kindnetd/aojea\/kindnetd:0.7.0/' kindnet.yaml`
`ctr --namespace=k8s.io images tag docker.io/aojea/kindnetd:latest docker.io/aojea/kindnetd:0.7.0`


## Check it works

```
root@ip-192-168-0-44:~# kubectl --kubeconfig /etc/kubernetes/admin.conf get nodes -o wide
NAME                                         STATUS   ROLES    AGE     VERSION   INTERNAL-IP                             EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION    CONTAINER-RUNTIME
ip-192-168-0-217                             Ready    <none>   2m1s    v1.16.2   2a05:d012:18:1c00:ff8d:68f6:3204:42c7   <none>        Ubuntu 18.04.3 LTS   4.15.0-1057-aws   containerd://1.3.2
ip-192-168-0-242                             Ready    <none>   2m13s   v1.16.2   2a05:d012:18:1c00:f354:4823:8487:c4d1   <none>        Ubuntu 18.04.3 LTS   4.15.0-1057-aws   containerd://1.3.2
ip-192-168-0-44.eu-west-3.compute.internal   Ready    master   3m      v1.16.2   2a05:d012:18:1c00:b4a4:459e:af22:57ba   <none>        Ubuntu 18.04.3 LTS   4.15.0-1057-aws   containerd://1.3.2
```

```
root@ip-192-168-0-44:~# kubectl --kubeconfig /etc/kubernetes/admin.conf get services -n kube-system
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   fd00:10:96::a   <none>        53/UDP,53/TCP,9153/TCP   3m39s
```


## Deployment

https://cloud.google.com/kubernetes-engine/docs/tutorials/http-balancer/web-deployment.yaml
https://cloud.google.com/kubernetes-engine/docs/tutorials/http-balancer/web-deployment.yaml


## external cloud provider

- [ ] In-Tree
- [ ] Out-of-tree

## Load Balancer

IPv6 addresses are public do we need load balancer?

$0.0294 per Classic Load Balancer-hour (or partial hour)
21.46 USD per month

Multiple IPv6 addresses for instance

https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/MultipleIP.html#working-with-multiple-ipv6

Number of secondary IP addresses limited by instance type

https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI

- Service CIDR a subnet from the Public CIDR
- Use portmap? external-ip? from public CIDR to Service CIDR
- Use a new instance as LB

Another instance?
node port?


## References

https://itnext.io/how-to-run-ipv6-enabled-docker-containers-on-aws-87e090ab0397
https://blog.scottlowe.org/2019/02/18/kubernetes-kubeadm-and-the-aws-cloud-provider/
https://docs.aws.amazon.com/vpc/latest/userguide/vpc-migrate-ipv6.html
https://docs.aws.amazon.com/vpc/latest/userguide/egress-only-internet-gateway.html
https://gist.github.com/milesjordan/d86942718f8d4dc20f9f331913e7367a
https://github.com/aws-quickstart/quickstart-vmware


## Known Issues

Github.com and k8s.io doesn't support IPv6

registry-1.docker.io and hub.docker.com don’t support IPv6, 

Use a S3 bucket

