# cis-configmap-using-as3
In Kubernetes, ConfigMap is an API object used to store non-confidential data in key-value pairs.

ConfigMap allows users to decouple configuration artifacts from image content to keep containerized applications portable. Pods can consume ConfigMaps as environment variables, command line arguments, or as configuration files in a volume.

The CIS is of type NodePort. 
It just communicates directly as NodePort, which is automatically added as pool members of BIG-IP. 
So, as long as it can communicate directly/has route to F5's Self-IP/Management IP 
So if you are using NodePort, you don't need container networking such as Flannel/BGP, as they are

Only to be used to access the pod directly (which involves routing/vxlan)

Tips: 
Don't forget to Increase Restjavad memory on BIG-IP
Please remove all the Ingress resources  that may result to conflicts with the newly created object in CMs. 
For CIS, the Service is the center of the universe.
    It links to the Deployment Configuration via the app label
    It links to the Configuration Map via the cis.f5.com/as3-tenant: <whatever> label
  
```
BIG-IP1: 10.201.10.156 (mgmt) / 172.16.100.156 on Version 16.1.0
Kubernetes Master Node: 10.201.10.151 on Ubuntu 18.04 LTS and kubernetes version v1.25.4; containerd://1.5.9
f5-appsvcs 	3.40.0
```
    
git clone  ____________________________

``` 
kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password="<BIG-IP Password>"
kubectl create serviceaccount bigip-ctlr -n kube-system
kubectl apply -f k8s_rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/F5Networks/k8s-bigip-ctlr/master/docs/config_examples/rbac/clusterrole.yaml
kubectl apply -f https://raw.githubusercontent.com/F5Networks/k8s-bigip-ctlr/master/docs/config_examples/customResourceDefinitions/customresourcedefinitions.yml
kubectl apply -f  cis-deploy.yaml
``` 

Although this example is copied from https://clouddocs.f5.com/training/community/containers/html/class1/module1/lab2.html
Take note that it was an old article and the Ingress Resource does not have ingressclassname configuration in it. 
We are now to specify the ingress controller that should handle the ingress resource by using the ingressClassName 
Source of truth in terms of parameters are now found in here: https://clouddocs.f5.com/containers/latest/userguide/ingress.html

    
To Deploy Hello World: 
``` 
kubectl apply -f deployment-hello-world.yaml
kubectl apply -f nodeport-service-hello-world.yaml
kubectl apply -f configmap-hello-world.yaml
``` 

## Take note that: 
    nodeport-service-hello-world.yaml has the below labels, so make sure before you implement/apply your configmap, you modify it as necessary. 

``` 
apiVersion: v1
kind: Service
metadata:
  name: f5-hello-world-web
  namespace: default
  labels:
    app: f5-hello-world-web
    cis.f5.com/as3-tenant: AS3
    cis.f5.com/as3-app: A1
    cis.f5.com/as3-pool: web_pool
spec:
  ports:
  - name: f5-hello-world-web
    port: 8080
    protocol: TCP
    targetPort: 8080
  type: NodePort
  selector:
    app: f5-hello-world-web
``` 


``` 
kind: ConfigMap
metadata:
  name: f5-as3-declaration
  namespace: default
  labels:
    f5type: virtual-server
    as3: "true"
data:
  template: |
    {
        "class": "AS3",    
        "declaration": {
            "class": "ADC",
            "schemaVersion": "3.10.0",
            "id": "urn:uuid:33045210-3ab8-4636-9b2a-c98d22ab915d",
            "label": "http",
            "remark": "A1 example",
            "AS3": {       ---------------------->> this is your  partition; cis.f5.com/as3-tenant: AS3 
                "class": "Tenant",
                "A1": {    ---------------------->> cis.f5.com/as3-app: A1
                    "class": "Application",
                    "template": "http",
                    "serviceMain": {   ---------------------->>  Name of the virtual server 
                        "class": "Service_HTTP",
                        "virtualAddresses": [
                            "172.16.4.89"
                        ],
                       "pool": "web_pool",   --------------------->>  cis.f5.com/as3-pool: web_pool
                        "virtualPort": 80
                    },
                    "web_pool": {      --------------------->>  cis.f5.com/as3-pool: web_pool
                        "class": "Pool",
                        "monitors": [
                            "http"
                        ],
                        "members": [
                            {
                                "servicePort": 8080,
                                "serverAddresses": []
                            }
                        ]
                    }
                }
            }
        }
    }
```

To Delete Hello-World: 

``` 
kubectl delete -f configmap-hello-world.yaml
kubectl delete -f nodeport-service-hello-world.yaml
kubectl delete -f deployment-hello-world.yaml
``` 


Upon applying the configmap: 
``` 
2023/01/31 08:25:10 [DEBUG] [AS3] Posting AS3 Declaration
2023/01/31 08:25:10 [DEBUG] [AS3] posting request to https://10.201.10.156/mgmt/shared/appsvcs/declare/
2023/01/31 08:25:16 [DEBUG] [AS3] Response from BIG-IP: code: 200 --- tenant:AS3 --- message: success
2023/01/31 08:25:16 [DEBUG] [AS3] Response from BIG-IP: code: 200 --- tenant:kubernetes --- message: no change
2023/01/31 08:25:16 [DEBUG] [AS3] Preparing response message to response handler for arp and fdb config
2023/01/31 08:25:16 [DEBUG] [AS3] Sent response message to response handler for arp and fdb config
2023/01/31 08:25:36 [DEBUG] [2023-01-31 08:25:36,940 __main__ DEBUG] config handler woken for reset
2023/01/31 08:25:36 [DEBUG] [2023-01-31 08:25:36,941 __main__ DEBUG] loaded configuration file successfully
2023/01/31 08:25:36 [DEBUG] [2023-01-31 08:25:36,941 __main__ DEBUG] NET Config: {}
2023/01/31 08:25:36 [DEBUG] [2023-01-31 08:25:36,941 __main__ DEBUG] loaded configuration file successfully
2023/01/31 08:25:36 [DEBUG] [2023-01-31 08:25:36,941 __main__ DEBUG] updating tasks finished, took 0.0005958080291748047 seconds
2023/01/31 08:25:37 [DEBUG] [CORE] Periodic enqueue of Service from Namespace: default, svc: f5-hello-world-web
2023/01/31 08:25:37 [DEBUG] [CORE] Periodic enqueue of Service from Namespace: kube-system, svc: kube-dns
2023/01/31 08:25:37 [DEBUG] [CORE] Periodic enqueue of Service from Namespace: kubernetes-dashboard, svc: dashboard-metrics-scraper
``` 
