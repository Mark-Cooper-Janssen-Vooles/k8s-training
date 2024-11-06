# k8s-training


## Setup
- confirm kubectl is installed `kubectl version` 
  - once kubectl is set up in the terminal, should be able to run `kubectl get pods`. you may need `kubectl get pods -n "<namespace>"`.
- also installed helm, can check `helm version`

## Lab 1: Clone Trooper Deployment
- `cd lab-1`
- make sure you're in the right k8s context (and namespace)
  - i.e. set docker to use right context or run `kubectl config use-context <context>`
- Create deployment: 
  - run `kubectl apply -f Deployment.yaml`
- check deployment:
  - either `kubectl get pods` if docker is set up to use the right context or `kubectl get deployment`
- Test a pod:
  - Get the IP of the pod: `kubectl get pods -o wide`
  - run curl to test one of the pods: `kubectl run -it --rm mark-curl --image=curlimages/curl --restart=Never -- sh`
    - this spins a new pod up, in this case called "mark", the `--rm` means to remove the pod after it exists, and to not restart it. the `sh` means to start an interactive shell session
    - inside the shell session, run `curl <whatever one of the pods ip's were>:8080/quote`. it shouls respond with a quote.
    - `exit`
- Delete a pod
  - this is to show how the deployment (ReplicaSet) automatically recreates it. 
  - `kubectl delete pod mark-clonetrooper-75c5d89547-d2nxv`
  - you can run `kubectl get pods` again and see it deleted and that a new pod has spun up in the last 10 seconds
  - new pod is automatically re-created and likely with a new IP.
- describe a pod 
  - provides more info on config of the pod `kubectl describe pod mark-clonetrooper-75c5d89547-nzcln`
  - will have info described in the Deployment.yaml under the template/spec area. 

## Lab 2: Clone Trooper Service
- add a service to our existing Clone Trooper pods which we'll test