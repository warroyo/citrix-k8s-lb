# citrix-k8s-lb
This repo uses the [citrix ingress controller](https://github.com/citrix/citrix-k8s-ingress-controller) as a load balancer service for k8s. we will also use the [citrix ipam controller]() to help with IP selection for the service IP. The current citrix docs are outdated in some places and I had to jump around to a few places to get it working correctly. so this aggregates that info and yaml.
here is a pretty good diagram of what is going to be happening.

![diagram](https://github.com/citrix/citrix-k8s-ingress-controller/raw/master/docs/media/type-loadbalancer.png)


## Install

### install the VIP CRD

More info on the VIP CRD [here](https://github.com/citrix/citrix-k8s-ingress-controller/blob/master/docs/crds/vip.md)

deploy the VIP CRD using this command:

```bash
kubectl create -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/crd/vip/vip.yaml
```

### Create a ADC system user

we need to create a system user on the netscaler device for the ingress controller to use. you read more [here](https://developer-docs.citrix.com/projects/citrix-k8s-ingress-controller/en/latest/deploy/deploy-cic-yaml/#create-system-user-account-for-citrix-ingress-controller-in-citrix-adc) or run the commands below.

1. ssh into the the device 

2. add a user 

```bash
add system user cic <password>
```

3. add a new policy

```bash
add cmdpolicy cic-policy ALLOW "^(?!shell)(?!sftp)(?!scp)(?!batch)(?!source)(?!.*superuser)(?!.*nsroot)(?!install)(?!show\s+system\s+(user|cmdPolicy|file))(?!(set|add|rm|create|export|kill)\s+system)(?!(unbind|bind)\s+system\s+(user|group))(?!diff\s+ns\s+config)(?!(set|unset|add|rm|bind|unbind|switch)\s+ns\s+partition).*|(^install\s*(wi|wf))|(^(add|show)\s+system\s+file)"
```

4. bind the user to the policy

```bash
bind system user cic cic-policy 0
```

5. create a k8s secret for the user

```bash
kubectl create secret  generic nslogin --from-literal=username='cic' --from-literal=password='mypassword'
```

### install the Ingress controller

installing the ingress controller will setup a cluster role and service account as well as a deployment for the actual controller.

1. edit the yaml in `ingress-controller.yml` you will need to update the ip address of the netscaler device(line 86). more docs on which ip to use can be found [here](https://developer-docs.citrix.com/projects/citrix-k8s-ingress-controller/en/latest/deploy/deploy-cic-yaml/#prerequisites) see the pre-reqs about determining the `NS_IP`

2. once updated run the below command to install

```bash
kubectl apply -f ingress-controller.yml
```

### Install the IPAM Controller

the Ipam controller will allow for automatic assigning of IPs based on ranges that you provide. see the options for providing IP ranges [here](https://developer-docs.citrix.com/projects/citrix-k8s-ingress-controller/en/latest/network/type_loadbalancer/#vip_range)

1. edit the `ipam-controller.yml` file and update the `VIP_RANGE` (line 59)

2. install the controller

```bash
kubectl apply -f ipam-controller.yml
```

### Test it

assuming all has gone well, deploy a service of type `LoadBalancer` and you will see a VIP created in the netscaler as well as a service group with the worker nodes as pool members.

## Troubleshooting

the best place to troubleshoot is to look at the logs for the ingress controller pod.