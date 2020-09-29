# Open MLOPS: MLOPS Open Source bundle

This Helm charts bundles open source software stack for advanced ML operations

## Chart Details

The Open MLOPs chart includes the following stack:

* Nuclio - https://github.com/nuclio/nuclio
* MLRun - https://github.com/mlrun/mlrun
* Jupyter - https://github.com/jupyter/notebook (+MLRun integrated)
* NFS - https://github.com/kubernetes-retired/external-storage/tree/master/nfs
* MPI Operator - https://github.com/kubeflow/mpi-operator

## Installing the Chart
Create a namespace for the deployed components:
```bash
$ kubectl create namespace mlops
```

Add the v3io-stable helm chart repo
```bash
$ helm repo add v3io-stable https://v3io.github.io/helm-charts/stable
```

To work with the open MLOPS stack, you must an accessible docker-registry. The registry's URL and credentials
are consumed by the applications via a pre-created secret

To create a secret with your docker-registry details:
```bash
kubectl --namespace mlops create secret docker-registry registry-credentials \
    --docker-username <registry-username> \
    --docker-password <login-password> \
    --docker-server <server URL, e.g. https://index.docker.io/v1/ > \
    --docker-email <user-email>
```

To install the chart with the release name `my-mlops` use the following command, 
note the reference to the pre-created `registry-credentials` secret in `global.registry.secretName`, 
and a `global.registry.url` with an appropriate registry URL which can be authenticated by this secret:

```bash
$ helm --namespace mlops \
	install my-mlops \
	--render-subchart-notes \
    --set global.registry.url=<registry URL e.g. index.docker.io/iguazio > \
    --set global.registry.secretName=registry-credentials \
    v3io-stable/open-mlops
```

Forward the nuclio dashboard port:
```sh
kubectl -n mlops port-forward $(kubectl get pods -n mlops -l nuclio.io/app=dashboard -o jsonpath='{.items[0].metadata.name}') 8070:8070
```

Forward the mlrun dashboard port:
```sh
export POD_NAME=$(kubectl --namespace mlops get pods -l "app.kubernetes.io/name=mlrun,app.kubernetes.io/instance=my-mlops,app.kubernetes.io/component="ui"" -o jsonpath="{.items[0].metadata.name}")
echo "Visit http://127.0.0.1:8080 to use your application"
kubectl --namespace mlops port-forward $POD_NAME 8080:80
```

### Install Kubeflow

MLRun enables you to run your functions while saving outputs and artifacts in a way that is visible to Kubeflow Pipelines.
If you wish to use this capability you will need to install Kubeflow on your cluster.
Refer to the [**Kubeflow documentation**](https://www.kubeflow.org/docs/started/getting-started/) for more information.

### Usage
Your applications are now available in your local browser:
- nuclio - http://localhost:8070
- mlrun - http://locahost:8080
- jupyter-notebook - http://localhost:30040

### Start Working

- Open Jupyter Notebook on [**jupyter-notebook UI**](http://localhost:30040) and run the code in 
[**examples/mlrun_basics.ipynb**](https://github.com/mlrun/mlrun/blob/master/examples/mlrun_basics.ipynb) notebook.

> **Note:**
> - You can change the ports by providing values to the helm install command.
> - You can add and configure a k8s ingress-controller for better security and control over external access.

## Advanced Chart Configuration
Configurable values are documented in the `values.yaml`, and the `values.yaml` of all sub charts. 
Override those [in the normal methods](https://helm.sh/docs/chart_template_guide/values_files/).

## Uninstalling the Chart
```bash
$ helm --namespace mlops uninstall my-mlops
```

#### Note on terminating pods and hanging resources
It is important to note that this chart generates several persistent volume claims and also provisions an NFS
provisioning server, to provide the user with persistency (via pvc) out of the box.
Because of the persistency of PV/PVC resources, after installing this chart, PVs and PVCs will be created,
And upon uninstallation, any hanging / terminating pods will hold the PVCs and PVs respectively, as those
Prevent their safe removal.
Because pods stuck in terminating state seem to be a never-ending plague in k8s, please note this,
And don't forget to clean the remaining PVCs and PVCs

Handing stuck-at-terminating pods:
```bash
$ helm --namespace mlops delete pod --force --grace-period=0 <pod-name>
```

Reclaim dangling persistency resources:

| WARNING: This will result in data loss! |
| --- |

```bash
# To list PVCs
$ helm --namespace mlops get pvc
...

# To remove a PVC
$ helm --namespace mlops delete pvc <pvc-name>
...

# To list PVs
$ helm --namespace mlops get pv
...

# To remove a PVC
$ helm --namespace mlops delete pvc <pv-name>

# Remove hostpath(s) used for mlrun (and possibly nfs). Those will be created, by default under /tmp, and will contain
# your release name, e.g.:
$ rm -rf my-mlops-open-mlops-mlrun
...
```
