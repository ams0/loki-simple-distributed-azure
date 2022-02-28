
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace --set-string controller.podLabels."promtail"="true"
helm upgrade --install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: alessandro.vozza@microsoft.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

kubectl create ns monitoring
htpasswd -c ./basic-auth promtail
kubectl create secret generic -n monitoring loki-auth --from-file=auth=./basic-auth

export AZURE_ACCOUNT_KEY="xxxx"
kubectl create secret generic -n monitoring loki-secrets --from-literal=AZURE_ACCOUNT_KEY=${AZURE_ACCOUNT_KEY}

helm upgrade loki --install -f values-ss.yaml -n monitoring --create-namespace grafana/loki-simple-scalable
helm upgrade promtail --install -f values-promtail.yaml  -n monitoring --create-namespace grafana/promtail
helm upgrade -i -n monitoring grafana grafana/grafana

#k label po -n ingress-nginx ingress-nginx-controller-7445b7d6dc-n5p2b promtail=true
k label po grafana-7d5c8f6dff-d7278 promtail=true

kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace monitoring port-forward $POD_NAME 3000

Datasource: Loki: 

See all logs: {job=~".+"}
{job="mysql"} |= "error"
Import 13359

curl -H "Content-Type: application/json" -XPOST  -k --user promtail:sV0AWV72Yj -s "https://loki.aks.cookingwithazure.com/loki/api/v1/push" --data-raw \
  '{"streams": [{ "stream": { "foo": "bar2" }, "values": [ [ "1645725694000000000", "fizzbuzz" ] ] }]}'

curl --user promtail:sV0AWV72Yj https://loki.aks.cookingwithazure.com/loki/api/v1/labels

kubectl run -n monitoring tmp-shell --rm -i --tty  --image nicolaka/netshoot -- /bin/bash
curl --user promtail:sV0AWV72Yj http://loki-loki-simple-scalable-write:3100/config
curl --user promtail:sV0AWV72Yj http://loki-loki-simple-scalable-write:3100/ready

alessandro@cloud:~$ sudo vi /etc/systemd/system/promtail.service
alessandro@cloud:~$ sudo vi /usr/local/bin/config-promtail.yml
alessandro@cloud:~$ sudo systemctl daemon-reload
alessandro@cloud:~$ sudo systemctl start promtail
alessandro@cloud:~$ sudo systemctl enable promtail
Created symlink /etc/systemd/system/multi-user.target.wants/promtail.service → /etc/systemd/system/promtail.service.
alessandro@cloud:~$ sudo systemctl status promtail
journalctl -x -u promtail

# data persist cluster destroy