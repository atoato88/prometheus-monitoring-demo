# prometheus-monitoring-demo

This repo include demonstration assets which monitors example application by prometheus.

# [OpenStack Days Tokyo 2018 / Cloud Native Days Tokyo 2018](http://openstackdays.com/) Demo Scenario

## 環境準備
-   GKE
    -   GKE で 1.9.7-gke.3 クラスタを作成する。
        -   もしくは　gcloud コマンドでデプロイする。
    -   download `gcloud` command and extract it.
    -   ```
        cd google-cloud-sdk/
        ./install.sh
        gcloud init
        gcloud component install kubectl
        gcloud container clusters get-credentials cluster-1  -z us-central1-a
        kubectl get nodes # for testing
        ```
-   kubectl plugins
    -   ```
        brew tap superbrothers/kubectl-service-plugin
        brew install kubectl-service-plugin
        kubectl plugin service --help # for testing
        ```
-   helm
    -   download `helm` from [here](https://github.com/helm/helm/releases) and extract it.
    -   ```
        mv helm /usr/local/bin/
        helm help # for testing
        ```
    -   ```
        kubectl create serviceaccount tiller --namespace kube-system
        cat <<EOF > rbac-config.yaml
        apiVersion: v1
        kind: ServiceAccount
        metadata:
          name: tiller
          namespace: kube-system
        ---
        apiVersion: rbac.authorization.k8s.io/v1beta1
        kind: ClusterRoleBinding
        metadata:
          name: tiller
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: ClusterRole
          name: cluster-admin
        subjects:
          - kind: ServiceAccount
            name: tiller
            namespace: kube-system
        EOF
        kubectl create -f rbac-config.yaml
        helm init --service-account tiller
        kubectl -n kube-system get pod -l app=helm # for testing
        helm version # for testing
        ```
    -   Appendix
        -   Do following for resetting helm
            ```
            helm reset -f
            ```

## Install prometheus-operator
-   ```
    helm repo add coreos https://s3-eu-west-1.amazonaws.com/coreos-charts/stable/
    helm install coreos/prometheus-operator --name prometheus-operator --namespace monitoring
    helm install coreos/kube-prometheus --name kube-prometheus --namespace monitoring
    ```

## Deploy Prometheus and Grafana for Kubernetes
-   ```
    helm install coreos/kube-prometheus --name kube-prometheus --namespace monitoring
    ```
-   ```
    kubectl -n monitoring port-forward $(kubectl -n monitoring get pods -l app=prometheus -o custom-columns=NAME:.metadata.name --no-headers) 8001:9090 &
    open -a "/Applications/Google Chrome.app" 'http://localhost:8001/'
    
    kubectl -n monitoring port-forward $(kubectl -n monitoring get pods -l app=kube-prometheus-grafana -o=custom-columns=NAME:.metadata.name --no-headers) 8002:3000 &
    open -a "/Applications/Google Chrome.app" 'http://localhost:8002/'
    ```
-   Look "Nodes" or "Kubernetes Capacity Planning" or "Kubernetes Cluster Health" in Grafana dashboard.


## Deploy example app
```
git clone https://github.com/atoato88/prometheus-monitoring-demo.git
cd prometheus-monitoring-demo

helm install --name example-app --namespace example example-app/.

kubectl -n example get pods
kubectl -n example get services
```

## Test access from worker pod
-   Open another terminal.
-   ```
    export SVC=$(kubectl -n example get service -l app=example-app -o custom-columns=IP:.spec.clusterIP --no-headers)
    echo ${SVC} # this value is used for $HOST env in Pod

    kubectl -n example run term --image alpine -it

    # Attach pod if already running pod. 
    kubectl -n example attach -it $(kubectl -n example get pod -o custom-columns=NAME:.metadata.name --no-headers -l run=term)

    # set env
    export HOST=10.11.243.251; export PORT=8888

    # check 200 page
    wget http://${HOST}:${PORT}/ -O -

    # check 404 page
    wget http://${HOST}:${PORT}/err -O -

    # check /metrics
    wget http://${HOST}:${PORT}/metrics -O -

    # start loop access
    export HOST=10.11.243.251; export PORT=8888; while : ; do if [[ $((${RANDOM} % 4)) = 0 ]]; then wget http://${HOST}:${PORT}/err -O - ; else wget http://${HOST}:${PORT} -O -; fi; sleep 1; done
    ```

## Deploy Prometheus and Grafana for example app
-   Open another terminal.
-   ```
    helm dependency build .
    helm install --name example-prometheus --namespace example prometheus-demo/.

    kubectl -n example create -f servicemonitor-manual.yaml
    kubectl -n example create -f configmap-example-app-dashboard.yaml
    
    kubectl -n example get pods
    kubectl -n example get services

    kubectl -n example port-forward $(kubectl -n example get pods -l app=prometheus -o custom-columns=NAME:.metadata.name --no-headers) 8003:9090 &
    open -a "/Applications/Google Chrome.app" 'http://localhost:8003/'

    kubectl -n example port-forward $(kubectl -n example get pods -l app=example-prometheus-grafana -o=custom-columns=NAME:.metadata.name --no-headers) 8004:3000 &
    open -a "/Applications/Google Chrome.app" 'http://localhost:8004/'
    ```
-   Look "Example App" in Grafana dashboard.

## Scale-out example app
-   ```
    helm upgrade example-app example-app/. --set replicaCount=5
    ```
-   Watch for increasing instance.


## Cleanup

### Cleanup for Deploy prometheus
```
helm delete --purge example-prometheus
kubectl -n example delete -f configmap-example-app-dashboard.yaml
kubectl -n example delete -f servicemonitor-manual.yaml

export PVC=$(kubectl -n example get pvc -o custom-columns=NAME:.metadata.name --no-headers)
expotr PV=$(kubectl -n example get pvc -o custom-columns=PV:.spec.volumeName --no-headers)
kubectl -n example delete pvc ${PVC}
kubectl -n example delete pv ${PV}
```

### Cleanup for Deploy example app
```
helm delete --purge example-app
```

### Cleanup for Test access from Pod
```
kubectl -n example delete deployments term
kubectl -n example delete pods $(kubectl -n example get pods -l run=term -o custom-columns=NAME:.metadata.name --no-headers)
```

