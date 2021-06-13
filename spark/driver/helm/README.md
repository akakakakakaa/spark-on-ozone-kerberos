# spark driver helm chart
Please ref [Jupyter Notebook & Spark on Kubernetes](https://towardsdatascience.com/jupyter-notebook-spark-on-kubernetes-880af7e06351)

## Prerequisite
1. The spark driver helm chart must share ozone-site.xml used by ozone
    ```
    kubectl get configmap -n ozone config -o jsonpath='{.data.ozone-site\.xml}' > ozone-site.xml
    ```