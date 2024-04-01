# Personal home automation server setup

⚠️ Note: This project is meant for personal use, and is not guaranteed to work for anyone else.

## Install or update k3s

```console
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644
```

## Change ingress controller to Contour

Disable default ingress controller

```diff
--- /etc/systemd/system/k3s.service.orig 2021-10-03 13:53:22.887426742 +0000
+++ /etc/systemd/system/k3s.service     2021-10-03 13:51:16.218849272 +0000
@@ -29,4 +29,4 @@
 ExecStart=/usr/local/bin/k3s \
     server \
        '--write-kubeconfig-mode=644' \
-
+       --disable traefik \
+       --disable servicelb
```

Restart k3s

```console
sudo systemctl daemon-reload
sudo systemctl restart k3s.service
```

The global kubeconfig file is stored at `/etc/rancher/k3s/k3s.yaml`.

Deploy Contour

```console
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
```

## Store configuration

```console
# homemetrics
kubectl create configmap homemetrics-config --from-file=secrets/homemetrics/config.yaml --dry-run=client -o yaml | kubectl apply -f -

# homeautomator
kubectl create configmap homeautomator-config --from-file=secrets/homeautomator/config.yaml --dry-run=client -o yaml | kubectl apply -f -

# dyndns
kubectl create configmap dyndns-config --from-file=secrets/dyndns-config.yaml --dry-run=client -o yaml | kubectl apply -f -
gcloud iam service-accounts keys create secrets/gcp-dyndns-client-serviceaccount.json --iam-account="dyndns-client@$(gcloud config get project).iam.gserviceaccount.com" --project="$(gcloud config get project)"
kubectl create secret generic dyndns-gcp-key --from-file=secrets/gcp-dyndns-client-serviceaccount.json --dry-run=client -o yaml | kubectl apply -f -

# cloudflared tunnel
kubectl create secret generic cloudflared-token --from-file=secrets/cloudflared-tunnel-token --dry-run=client -o yaml | kubectl apply -f -
```

````

## Deploy apps

Build custom node-red image and import it to containerd store

```console
podman build --tag nodered:latest docker/nodered/
podman save localhost/nodered:latest | sudo k3s ctr images import -
sudo k3s ctr images ls | grep nodered
````

## Tips

### Storage

Host mounts are used for local storage.
K3s persistent volumes can be directly accessed on host under directory `/var/lib/rancher/k3s/storage/`

### VictoriaMetrics

VictoriamMtrics might go into crashloop after upgrade with error

```
FATAL: cannot create lock file: cannot acquire lock on file "/victoria-metrics-data/flock.lock": resource temporarily unavailable; make sure a single process has exclusive access to "/victoria-metrics-data"
```

```console
kubectl scale deployment victoriametrics --replicas 0
kubectl scale deployment victoriametrics --replicas 1
```

## Backup

```console
mkdir -p backup
BACKUP_TIMESTAMP=$(date +"%Y-%m-%dT%H.%M.%S")
kubectl exec deployment/zigbee2mqtt -c zigbee2mqtt -- tar zvcf - /app/data/ > backup/$BACKUP_TIMESTAMP-zigbee2mqtt.tar.gz
kubectl exec deployment/zwave2mqtt -- tar zvcf - /usr/src/app/store > backup/$BACKUP_TIMESTAMP-zwave2mqtt.tar.gz
kubectl exec deployment/grafana -- tar zvcf - /var/lib/grafana > backup/$BACKUP_TIMESTAMP-grafana.tar.gz
kubectl exec deployment/victoriametrics -- tar zvcf - /victoria-metrics-data > backup/$BACKUP_TIMESTAMP-victoriametrics.tar.gz
kubectl exec deployment/nodered -- tar zvcf - /data > backup/$BACKUP_TIMESTAMP-nodered.tar.gz
```

## Adding new zigbee device

- https://www.zigbee2mqtt.io/advanced/support-new-devices/01_support_new_devices.html#_2-adding-your-device
  https://github.com/Koenkk/zigbee2mqtt/issues?q=is%3Aissue+is%3Aopen+TS011F
  https://www.zigbee2mqtt.io/devices/TS011F_plug_3.html#options

## VictoriaMetrics

- MetricsQL language reference https://github.com/VictoriaMetrics/VictoriaMetrics/wiki/MetricsQL
