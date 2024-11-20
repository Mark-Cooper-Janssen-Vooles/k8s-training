# k8s-training

- [Deployment](#lab-1-clone-trooper-deployment)
- [Services](#lab-2-clone-trooper-service)
- [Ingress](#lab-3-clone-trooper-ingress)
- [Deployments part 2](#lab-4-deployments-part-2)
  - deploy an upgrade and roll it back
  - cleanup resources
- [Helm](#lab-5-helm)
- [Resources and autoscaling](#lab-6-resources-and-autoscaling)

## Setup
- confirm kubectl is installed `kubectl version` 
  - once kubectl is set up in the terminal, should be able to run `kubectl get pods`. you may need `kubectl get pods -n "<namespace>"`.
- also installed helm, can check `helm version`

## Lab 1: Clone Trooper Deployment
- deploying an image as a k8s pod with replicas etc
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
    - this spins a new pod up, in this case called "mark", the `--rm` means to remove the pod after it exits, and to not restart it. the `sh` means to start an interactive shell session
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
  - in k8s a service is an abstraction that defines a logical set of Pods and a policy to access them
  - it provides a stable endpoint (IP address and DNS name) for accessing a set of Pods, even when the pods are created and destroyed over time.
- Make sure you're in the right kubectl context still + `cd lab-2`
- Create the service:
  - `kubectl apply -f Service.yaml`
- Check your service:
  - `kubectl get service`
  - view pod endpoints attached to your service: `kubectl get endpoints mark-clonetrooper`
  - see how the service selector matches the pod labels: `kubectl get pod --show-labels`
  - run curl we can test our service: 
    - `kubectl run -it --rm mark-curl --image=curlimages/curl --restart=Never -- sh`
    - test service name: `curl http://mark-clonetrooper/quote`
    - test service.namespace: `curl http://mark-clonetrooper/quote`
    - `exit`
- Describe your service: 
  - provides more info on the config of the service: `kubectl describe service mark-clonetrooper`

## Lab 3: Clone Trooper Ingress
- Add ingress to our existing clone trooper service
  - An ingress is used to route external HTTP(S) traffic to different services within a k8s cluster based on URL paths, hostnames or other rules. It acts as a reverse proxy that can be used to expose multiple services through a single external IP address. 
- `cd lab-3`
- Create the Ingress:
  - `kubectl apply -f Ingress.yaml`
- Check the Ingress
  - `kubectl get ingress`
  - note in the address you can see the ELB (elastic load balancer)'s url. Note that when accessing it though you'll use the HOSTS url.
- Test Ingress 
  - oopen the hosts URL in a browser: mark-clonetrooper.biz.<someDomain>.com/quote and you should see a quote 
- Describe Ingress:
  - `kubectl describe ingress mark-clonetrooper`
  - the rules show the host mapped to the backend Service (i.e the Service.yaml from lab-2)

## Lab 4: Deployments part 2
- this lab will deploy an upgrade and roll it back 
- `cd lab-4`, the deployment.yaml here has the image tag incremented to 0.2. 
- Apply the updated deployment: `kubectl apply -f Deployment.yaml`
- View Pod deployment:
  - `kubectl get pods` - it is terminating the old ones and starting new ones.
  - now go to view it: `http://mark-clonetrooper.biz.<someDomain>.com/quote` - it now only does Jar Jar Binks quotes, we need to roll back
- View Rollout Revisions
  - `kubectl rollout history deployment mark-clonetrooper`
  - it should say there are two revisions 1 and 2.
- View ReplicaSets
  - `kubectl get replicaset`
  - it should list two for mark-clonetrooper (one of them is currently deployed)
  - `kubectl describe replicaset mark-clonetrooper-75c5d89547` (mark-clonetrooper-75c5d89547 is the old one not deployed)
    - should see the version of the image here as 0.1 - we can confirm this is the one we want to roll back to. 
    - we should also see: `deployment.kubernetes.io/revision: 1` - confirming revision 1 is the one we want to rollback to.
- Perform rollback
  - `kubectl rollout undo deployment mark-clonetrooper --to-revision=1` which spins up two new pods and deletes the old ones. 
- Test rollback
  - go back to url http://mark-clonetrooper.biz.<someDomain>.com/quote and see non-jarjar binks quotes!
- View Rollouts
  - `kubectl rollout history deployment mark-clonetrooper`
  - theres now a revision 2 and 3 (revision 1 is promoted to revision 3)
- Cleanup resources
  - `kubectl delete ingress mark-clonetrooper`
  - `kubectl delete service mark-clonetrooper`
  - `kubectl delete deployment mark-clonetrooper`

## Lab 5: Helm
- helm is used to help template an application 
  - an example is the values.yml file
  - values in yalues.yaml fie are injected into the templates folder
  - a Chart is made up of a web-application (in this case ours is called 'clonetrooper', the folder)
    - the chart.yaml contains basic info and metadata 
    - values.yaml is the default values file (used unless overridden)
    - the templates folder
      - uses the values and generates valid k8s manifest files (covered in labs 1-4 above)

- release the clonetrooper app as before but wrapped in a helm package 
- Inspect chart:
  - chart.yaml (the info file)
  - values.yaml - the default values file (used in the files insid the templates folder)
  - template - the template files which will become manifest files
- the myvalues.yaml file overwrites the values.yaml file 
- Preview Manifests
  - cd `lab-5`
  - previewing manifests follows the format: `helm template {name of release} {location of chart} -f {values}`
  - to see what will be generated, `helm template mark-clonetrooper clonetrooper`
    - the above pumps out just using the values.yaml, not our custom myvalues.yaml file.
    - to use myvalues.yaml file: `helm template mark-clonetrooper clonetrooper -f myvalues.yaml`, and we now see it renamed. 
- Release
  - the format for releasing is `helm install {name of release} {location of chart} -f {values}`
  - to release ours: `helm install mark-clonetrooper clonetrooper -f myvalues.yaml`
  - should be viewable now: http://mark-clonetrooper.biz.{domain}.com/quote
- Upgrade a release
  - pass the image tag over the command line: `helm upgrade mark-clonetrooper clonetrooper -f myvalues.yaml --set image.tag=0.2`
    - right-most values on the commandline will overwrite any other values. 
    - Multiple values files are frequently used. Usually there will be a common values file for an app that covers things like name, port etc. Then another values file per environment, test/uat/prod. That is then deployed like so: `helm upgrade appname generic-chart -f common-values.yaml -f prod-values.yaml --set image.tag=0.2`
      - prod-values.yaml will overwrite common-values.yaml and --set image overwrites both.
  - the URL should now show jar jar binks quotes only
- Rollback (using helm)
  - view all helm releases: `helm list`
  - view history of releases: `heml history mark-clonetrooper` // shows revision 1 and 2
  - perform rollback: `helm rollback mark-clonetrooper 1`
  - view rollback results: `helm history mark-clonetrooper` or `kubectl rollout history deployment mark-clonetrooper`
- Cleanup resources (using helm)
  - `helm uninstall mark-clonetrooper`

## Lab 6: Resources and autoscaling
- version of helm you use needs to match k8s server version
- this lab uses a php-apache app that performs CPU intensive operations for every page request to see the HPA (horizontal pod autoscaler) add more pods as the load increases, using a helm generic-chart to deploy it.
- Update helm repos
  - we need to add the generic-chart repo to Helm's supported repos: 
````
helm repo add helm-incubator https://{someurl}.com/helm-incubator

helm repo update

helm repo list
# NAME            URL
# helm-incubator  https://{someurl}.com/helm-incubator
````
- Release
  - try running: `helm install mark-hpa-test generic-service --repo https://{someurl}.com/helm-incubator -f myvalues.yaml --set image.tag=0.1`
  - helm displays a summary of the release and some notes 
- View Results
  - `kubectl get hpa` will show min pods, max pods and replicas 
    - shows targets as 5%/50%, 23%/75% min pods 1, max pods 5, replicas 1
      - note the replicas are what increase 
    - 75% is the memory target (in the myvalues.yaml)
    - 50% is the CPU target
    - targets listed as current/target so 5% is current cpu usage, scale at 50%. and 23% current memory target, scale at 75%.
  - `kubectl get pods` 
  - `kubectl describe hpa mark-hpa-test` gives more info. 
- Generate load
  - we will run 'busybox' to query the service endpoint and generate load 
  - run the command in another terminal to watch your HPA: `kubectl get hpa mark-hpa-test --watch`
  - in the other terminal run `kubectl run -i --tty mark-busybox --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://mark-hpa-test; done"`
  - as the targets for CPU go over 50%, the replicas should increase. as the replicas increase, the target should come down. it should show 5/5 replicas created.
- Cleanup
  - stop the busy box and run `helm uninstall mark-hpa-test`

  ## Lab 7: Troubleshooting 
  - after installing the service, `helm install mark-troubleshoot generic-service --repo https://{somerepo} -f myvalues.yaml` 
    - get the pod name `kubectl get pods`
    - check the logs `kubectl logs <podname>`
    - fix the issues in the .yaml (typos, env variable missing) then uninstall `helm uninstall mark-troubleshoot` and reinstall 
    - go to the url, success!