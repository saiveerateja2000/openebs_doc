# openebs_doc

# **Deploying Apache Airflow with OpenEBS for Persistent DAG Storage**

## **📌 Overview**

Apache Airflow requires persistent storage for DAGs, logs, and metadata. In a Kubernetes environment, OpenEBS provides **dynamic storage provisioning**, allowing Airflow components to use Persistent Volumes (PVs) for reliable data storage.

This guide explains **how to install OpenEBS, configure a custom storage class, and use it in Airflow** to store DAGs persistently.

---

## **1️⃣ Installing OpenEBS in Kubernetes**

OpenEBS provides **local and dynamic storage solutions**. To install OpenEBS, run:

```sh
kubectl create namespace openebs
helm repo add openebs https://openebs.github.io/charts
helm repo update
helm install openebs openebs/openebs -n openebs
```

✅ This installs OpenEBS in the `openebs` namespace and sets up storage provisioning.

---

## **2️⃣ Creating a Custom Storage Class for Airflow DAGs**

We create a **StorageClass** named `airflow_ebs`, which OpenEBS will manage.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: airflow_ebs
provisioner: openebs.io/local
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### **Q: Do I need to apply this StorageClass in the **``** namespace?**

No, **StorageClass is a cluster-wide resource**, so applying it in any namespace makes it available to all.

✅ Apply the StorageClass using:

```sh
kubectl apply -f storage-class.yaml
```

---

## **3️⃣ Creating a Persistent Volume (PV) for DAGs**

DAGs are stored in `/ubuntu/svt/airflow/dags` on the host machine. We define a **PersistentVolume (PV)** to map this directory to a Kubernetes resource.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: airflow-dags-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: airflow_ebs
  hostPath:
    path: "/ubuntu/svt/airflow/dags"  # DAGs location on host
```

✅ Apply the PV:

```sh
kubectl apply -f pv-dags.yaml
```

### **Q: How does OpenEBS recognize this Persistent Volume?**

- The **StorageClassName** (`airflow_ebs`) links the PV to OpenEBS.
- OpenEBS **automatically discovers** and provisions storage when a PersistentVolumeClaim (PVC) requests it.

---

## **4️⃣ Creating a Persistent Volume Claim (PVC) for DAGs**

Airflow Helm charts use a **PVC** to request storage dynamically.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: airflow-dags-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: airflow_ebs
  resources:
    requests:
      storage: 5Gi
```

✅ Apply the PVC:

```sh
kubectl apply -f pvc-dags.yaml
```

### **Q: How does Airflow use this PVC?**

- The **PVC binds to the PV**, granting access to `/ubuntu/svt/airflow/dags`.
- Airflow mounts this PVC inside its **containers**, making DAGs available.

---

## **5️⃣ Configuring Airflow to Use Persistent Storage**

In `airflow_values.yaml`, modify Airflow to mount the DAGs directory.

```yaml
dags:
  persistence:
    enabled: true
    existingClaim: airflow-dags-pvc  # Use the PVC we created
    subPath: ""

extraVolumeMounts:
  - name: airflow-dags
    mountPath: /opt/airflow/dags  # Inside Airflow container

extraVolumes:
  - name: airflow-dags
    persistentVolumeClaim:
      claimName: airflow-dags-pvc  # Uses the PVC
```

### **Q: How does Airflow access DAGs from **``**?**

- Kubernetes **mounts** `/ubuntu/svt/airflow/dags` inside the Airflow container at `/opt/airflow/dags`.
- Airflow **expects DAGs in **``, so it can read them seamlessly.

✅ Apply the updated `airflow_values.yaml` during Helm installation:

```sh
helm install airflow apache-airflow/airflow -n airflow --create-namespace -f airflow_values.yaml
```

---

## **6️⃣ Verifying the Setup**

To check if the PV and PVC are bound:

```sh
kubectl get pv
kubectl get pvc -n airflow
```

To check if Airflow pods are running:

```sh
kubectl get pods -n airflow
```

To verify DAGs inside the Airflow container:

```sh
kubectl exec -it <scheduler-pod> -n airflow -- ls /opt/airflow/dags
```

---

## **🚀 Conclusion**

✅ OpenEBS provides **dynamic storage provisioning**.\
✅ DAGs are stored persistently at `` but available inside Airflow at ``.\
✅ This setup ensures **Airflow DAGs are retained across pod restarts**.

---

### **🔥 Key Takeaways**

🔹 **OpenEBS dynamically provisions storage when a PVC requests it**.\
🔹 **A StorageClass (**``**) tells Kubernetes to use OpenEBS for storage**.\
🔹 **Persistent Volumes map host directories to Kubernetes storage resources**.\
🔹 **Airflow mounts the DAGs PVC at **``**, ensuring seamless access**.

📌 **Now you have a production-ready Airflow deployment with OpenEBS!** 🎯

