# DEMO

## Admin
```
kubectl config set-context --current --namespace=<insert-namespace-name-here>
# Validate it
kubectl config view --minify | grep namespace:
```

# Deploy with without template

### Deploy container to K8S
```
kubectl run demo --image=cloudnatived/demo:hello --port=9999 --labels app=demo --namespace=default
```

### Forward deployment to local machine
```
kubectl port-forward deploy/demo 9999:8888
```

### Show in browser
```
http://localhost:9999
```

### Check pod status
```
kubectl get pods --selector app=demo
```

### Show deployment (template) of above " deployment"
Deployments start Pods, but there’s a little more to it than that. In fact, Deployments don’t manage Pods directly. That’s the job of the ReplicaSet object.

Run command to show template created with ```kubectl run```
```
kubectl describe deployments/demo
```

### Show how replicaSet keeps the number of pods at 1 (one)
```
# show pods of demo
kubectl get pods --selector app=demo
# delete pods of demo
kubectl delete pods --selector app=demo
# show pods of demo back to what is set in replicaSet
kubectl get pods --selector app=demo
```
### Delete all resource with label app=demo
```
kubectl delete all -l app=demo
```

# Templates

## Deployment

[Deployment template](hello-k8s/k8s/deployment.yaml)

### Apply Deployment template
```
kubectl get deployments

kubectl apply -f hello-k8s/k8s/deployment.yaml --namespace=default

kubectl get deployments
```
### Show pods
```
kubectl get pods --selector app=demo
```
## Service

[Service template](hello-k8s/k8s/service.yaml)
NOTE: see the selector in template.

You can find the ip of the pod, but can change when pod moves to different K8S host

### Apply Service template
```
kubectl get services

kubectl apply -f hello-k8s/k8s/service.yaml --namespace=default

kubectl get services
```
### Port forward service (NOT POD) as seen above
```
kubectl port-forward service/demo 9999:8888
```
### Show in browser
```
http://localhost:9999
```
### Describe pod
```
kubectl get pods
# change the pod identifier
kubectl describe pod/demo-7cbf698c44-sxlbd
```

### Delete all resource with label app=demo
```
kubectl delete all -l app=demo
```

# Resource Requests
[Deployment template](hello-k8s-resources/k8s/deployment.yaml)

### Apply template
```
kubectl apply -f hello-k8s-resources/k8s/deployment.yaml --namespace=default
```
### Show template (on K8S)
```
kubectl describe deployment/demo
```
### Delete all resource with label app=demo
```
kubectl delete all -l app=demo
```

# Probes
Is the pod/deployment alive and ready to service

[Deployment template](hello-k8s-probes/k8s/deployment.yaml)

### Apply template
```
kubectl apply -f hello-k8s-probes/k8s/deployment.yaml --namespace=default
```
### Show template (on K8S)
```
kubectl describe deployment/demo
```
### Delete all resource with label app=demo
```
kubectl delete all -l app=demo
```

# Auto scaling

Deployment template](hello-k8s-autoscale/k8s/deployment.yaml)

### Apply template
```
kubectl apply -f hello-k8s-autoscale/k8s/ --namespace=default
```
### Show template (on K8S)
```
kubectl describe deployment/demo
```

### Horizontal pods Autoscaler
```
kubectl autoscale deployment demo --cpu-percent=50 --min=1 --max=10
```
### Show
```
kubectl get hpa
```
### More load
NOTE new terminal
```
kubectl run -it --rm load-generator --image=busybox /bin/sh
Hit enter for command prompt
while true; do wget -q -O- http://demo; done
```

# Manual scaling
```
kubectl scale --replicas=3 deployment/demo
```
### Only when current is 3
```
kubectl scale --current-replicas=3 --replicas=5 deployment/demo
```
### Watch it change
```
kubectl get pods --watch
```

# Clean up

### Delete all resource with label app=demo
```
kubectl delete all -l app=demo
```