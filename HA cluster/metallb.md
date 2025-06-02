# MetalLB

> 以下僅簡單紀錄如何安裝 MetalLB，相關介紹後續會再補上。

> [ref](https://metallb.io/installation/)

Install：

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-frr.yaml
```

Configure LB IP pool:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
EOF
```

> 之後的 LB service IP 就會從 192.168.1.240 ~ 192.168.1.250 之間分配

測試看看：

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: test-lb
spec:
  type: LoadBalancer    
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80    
    targetPort: 80
EOF
```
```bash
kubectl get ipaddresspool -n metallb-system
```
```text
NAME         AUTO ASSIGN   AVOID BUGGY IPS   ADDRESSES
first-pool   true          false             ["192.168.1.240-192.168.1.250"]
```

```bash
kubectl get svc test-lb
```
```text
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
test-lb   LoadBalancer   10.109.94.233   192.168.1.240   80:30960/TCP   3m51s
```
