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
