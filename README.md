# Instance Groups

## Setting up a single VM

```
gcloud compute instances create nginx \
  	--image-family gci-stable \
    --image-project google-containers \
    --machine-type n1-standard-1
```

Login into the nginx VM

```
gcloud compute ssh nginx
```

Start the Kubernetes kubelet; which will manage our containers:

```
sudo systemctl enable kubelet
sudo systemclt start kubelet
```

Create a Pod manifest to run the nginx container under the Kubernetes kubelet.

```
cat > nginx.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  hostNetwork: true
  containers:
    - name: nginx
      image: nginx:1.10.1
EOF
```

Move the nginx pod manifest into place:

```
sudo mv nginx.yaml /etc/kubernetes/manifests/
```

Allow HTTP traffic

```
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
```

### Test the nginx service

Exit out of the nginx VM and run the following commands to test the nginx instance

```
gcloud compute firewall-rules create nginx --allow tcp:80
```

```
NGINX_PUBLIC_ADDRESS=$(gcloud compute instances describe nginx \
  --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')
```

```
curl http://${NGINX_PUBLIC_ADDRESS}
```

## Automate VM creation using cloud-init

```
cat > cloud-init << EOF
#cloud-config

write_files:
  - path: etc/kubernetes/manifests/nginx.yaml
    permissions: 0644
    owner: root
    content: |
      apiVersion: v1
      kind: Pod
      metadata:
        name: nginx
      spec:
        hostNetwork: true
        containers:
          - name: nginx
            image: nginx:1.10.1
runcmd:
  - systemctl daemon-reload
  - systemctl start kubelet
  - iptables -A INPUT -p tcp --dport 80 -j ACCEPT
  - iptables-save
EOF
```

```
gcloud compute instances create nginx-cloud-init \
  --image-family gci-stable \
  --image-project google-containers \
  --machine-type n1-standard-1 \
  --metadata-from-file user-data=cloud-init
```

Testing the nginx-cloud-init VM:

```
NGINX_CI_PUBLIC_ADDRESS=$(gcloud compute instances describe nginx-cloud-init \
  --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')
```

```
$ curl http://${NGINX_CI_PUBLIC_ADDRESS}
```

Cloud-init is greate for automating the creation of a single VM, but what if you need say 100 VMs just like this one?

## Creating multiple VMs using instance-templates

```
$ gcloud compute instance-templates create nginx-instance-template \
  --image-family gci-stable \
  --image-project google-containers \
  --machine-type n1-standard-1 \
  --metadata-from-file user-data=cloud-init
```

```
$ gcloud compute instance-groups managed create nginx-instance-group \
  --base-instance-name nginx \
  --size 3 \
  --template nginx-instance-template
```

```
NAME                  LOCATION    SCOPE  BASE_INSTANCE_NAME  SIZE  TARGET_SIZE  INSTANCE_TEMPLATE        AUTOSCALED
nginx-instance-group  us-west1-b  zone   nginx               0     3            nginx-instance-template  no
```

```
$ gcloud compute instances list
```
```
NAME        ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP      STATUS
nginx-7dp9  us-west1-b  n1-standard-1               10.240.0.2   104.199.112.124  RUNNING
nginx-sipu  us-west1-b  n1-standard-1               10.240.0.4   104.199.114.154  RUNNING
nginx-zs7v  us-west1-b  n1-standard-1               10.240.0.3   104.199.124.134  RUNNING
```

### Manually scaling an instance group

```
$ gcloud compute instance-groups managed resize nginx-instance-group --size 5
```

```
$ gcloud compute instances list
```
```
NAME        ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP      STATUS
nginx-7dp9  us-west1-b  n1-standard-1               10.240.0.2   104.199.112.124  RUNNING
nginx-o68k  us-west1-b  n1-standard-1               10.240.0.6   104.199.114.46   RUNNING
nginx-sipu  us-west1-b  n1-standard-1               10.240.0.4   104.199.114.154  RUNNING
nginx-sz84  us-west1-b  n1-standard-1               10.240.0.5   104.199.116.59   RUNNING
nginx-zs7v  us-west1-b  n1-standard-1               10.240.0.3   104.199.124.134  RUNNING
```

Scaling back down

```
$ gcloud compute instance-groups managed resize nginx-instance-group --size 1
```

```
$ gcloud compute instances list
```
```
NAME        ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP      STATUS
nginx-zs7v  us-west1-b  n1-standard-1               10.240.0.3   104.199.124.134  RUNNING
```

## Autoscaling

Start with the instance group size set to 1:

```
gcloud compute instance-groups managed resize nginx-instance-group --size 1
```

Set an autoscaling policy:

```
gcloud compute instance-groups managed set-autoscaling nginx-instance-group \
  --max-num-replicas 20 \
  --target-cpu-utilization 0.05 \
  --cool-down-period 90
```

Notice the `nginx-instance-group` has autoscale enabled.

```
$ gcloud compute instance-groups managed list
```
```
NAME                  ZONE        BASE_INSTANCE_NAME  SIZE  TARGET_SIZE  INSTANCE_TEMPLATE        AUTOSCALED
nginx-instance-group  us-west1-b  nginx               1     2            nginx-instance-template  yes
```

The default min size of an instance group is 2. Notice how the instance group has been autoscaled to meet the target size of 2:

```
$ gcloud compute instances list
```
```
NAME        ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP      STATUS
nginx-o3xl  us-west1-b  n1-standard-1               10.240.0.2   104.199.114.154  RUNNING
nginx-zs7v  us-west1-b  n1-standard-1               10.240.0.3   104.199.124.134  RUNNING
```

### Trigger an autoscaling event

Install a load tester:

https://github.com/rakyll/hey

```
go get -u github.com/rakyll/hey
```

### Push some load to one of the nginx machines

Set the public IP address:

```
NGINX_PUBLIC_ADDRESS=$(gcloud compute instances describe nginx-zs7v \
  --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')
```

Test the connection:

```
$ curl http://${NGINX_PUBLIC_ADDRESS}
```

Send load to the nginx instance:

```
hey -n 2000 http://${NGINX_PUBLIC_ADDRESS}
```

Notice the number of VMs have scaled to 4 because of the CPU spike caused by the `hey` command:

```
gcloud compute instances list
```
```
NAME        ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP      STATUS
nginx-o3xl  us-west1-b  n1-standard-1               10.240.0.2   104.199.114.154  RUNNING
nginx-ot4b  us-west1-b  n1-standard-1               10.240.0.5   104.199.112.124  RUNNING
nginx-sf9m  us-west1-b  n1-standard-1               10.240.0.4   104.198.109.21   RUNNING
nginx-zs7v  us-west1-b  n1-standard-1               10.240.0.3   104.199.124.134  RUNNING
```
