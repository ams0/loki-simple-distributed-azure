# Loki simple, distributed with Azure storage

This deploys the [loki-simple-scalable](https://github.com/grafana/helm-charts/tree/main/charts/loki-simple-scalable) helm chart with configurable number of readers and writers and a `gateway` component (essentially, an nginx pod and the associated ingress):

Deploy `ingress-nginx` and `cert-manager`

```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace --set-string controller.podLabels."promtail"="true"

helm upgrade --install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

#replace with your email

kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: <your email>
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

```

Create the two secrets, one with the `basic-auth` for promtail and one with the Azure Storage account key:

```
kubectl create ns monitoring
htpasswd -c ./basic-auth promtail
kubectl create secret generic -n monitoring loki-auth --from-file=auth=./basic-auth

export AZURE_ACCOUNT_KEY="xxxx"
kubectl create secret generic -n monitoring loki-secrets --from-literal=AZURE_ACCOUNT_KEY=${AZURE_ACCOUNT_KEY}
```

Finally deploy Loki and promtail, using the provided `values.yaml` file (replace the azure storage account name)

```
helm upgrade loki --install -f values-ss.yaml -n monitoring --create-namespace \
  --set loki.config.common.storage.azure.account_name=storeme \
  --set loki.config.common.storage.azure.container_name=loki \
  grafana/loki-simple-scalable

helm upgrade promtail --install -f values-promtail.yaml  -n monitoring --create-namespace grafana/promtail
```

Please note that you have to attach the label `promtail=true` to pods you want to collect logs from. If no pods are present with that label on a node, the promtail pod on that node will be in a non-ready state.

```
kubectl label po -n ingress-nginx ingress-nginx-controller-7445b7d6dc-n5p2b promtail=true
kubectl label po grafana-7d5c8f6dff-d7278 promtail=true
```

Optionally, you can deploy grafana to check the logs (under the "Explore" tab on the lef):

```
helm upgrade -i -n monitoring grafana grafana/grafana

kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace monitoring port-forward $POD_NAME 3000
```

Add a Loki datasource with the ingress URL and the basic authentication (if you have proper Let'sEncrypt certificates, you won't need to skip TLS verification)

Some LogQL tips:

```
See all logs: {job=~".+"}

{job="mysql"} |= "error"
```

It's handy to import [dashboard 13359](https://grafana.com/grafana/dashboards/13359) if you collect loki logs

## Troubleshooting Loki

Some handy API calls for Loki

```
#replace the unix date
curl -H "Content-Type: application/json" -XPOST  -k --user promtail:xxxx -s "https://loki.aks.cookingwithazure.com/loki/api/v1/push" --data-raw \
  '{"streams": [{ "stream": { "foo": "bar2" }, "values": [ [ "1645725694000000000", "fizzbuzz" ] ] }]}'

curl --user promtail:xxxx https://loki.aks.cookingwithazure.com/loki/api/v1/labels
```

For the way the helm chart deploys Loki, some endpoints are only available from within the cluster:

```
kubectl run -n monitoring tmp-shell --rm -i --tty  --image nicolaka/netshoot -- /bin/bash

curl --user promtail:xxx http://loki-loki-simple-scalable-write:3100/config
curl --user promtail:xxx http://loki-loki-simple-scalable-write:3100/ready
```

## Monitor VMs

It's simple as running promtail and pointing it to the ingress of our Loki deployment. We use a `systemd` unit file to make sure promtail is always running.

```
wget https://github.com/grafana/loki/releases/download/v2.4.2/promtail-linux-amd64.zip
unzip promtail-linux-amd64.zip
mv promtail-linux-amd64 /usr/local/bin/promtail
chmod +x /usr/local/bin/promtail

cat <<EOF > /etc/systemd/system/promtail.service
#/etc/systemd/system/promtail.service
[Unit]
Description=Promtail service
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/promtail -config.file /usr/local/bin/config-promtail.yml

[Install]
WantedBy=multi-user.target
EOF

cat <<EOF > /usr/local/bin/config-promtail.yml
#/usr/local/bin/config-promtail.yml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: https://loki.aks.cookingwithazure.com/loki/api/v1/push
    basic_auth:
      username: promtail
      password: xxxxx
    tls_config:
      insecure_skip_verify: true

scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      source: vm
      job: varlogs
      __path__: /var/log/**/*.log
EOF
```

This will enable and start the promtail service. `journalctl` is the `systemd`-native way to check service logs.

```
systemctl daemon-reload
systemctl start promtail
systemctl enable promtail
journalctl -x -u promtail
```


## Notes

Data persist cluster destroy! 
