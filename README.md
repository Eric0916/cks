## 2
kubectl create ns qa
kubectl create sa front-sa -n qa
mkdir -p /cks/sa
kubectl run nginx --image=nginx --dry-run=client -o yaml >  /cks/sa/pod1.yaml

## 3
kubectl create ns testing


## 4
kubectl create ns db
kubectl create sa service-account-web -n db
kubectl create role role-1 --verb=create,delete,get --resource=deployments,statefulsets,daemonsets,pods -n db
kubectl create rolebinding pods-get-binding --role=role-1 --serviceaccount=db:service-account-web -n db
cat > web-pod.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: web-pod
  name: web-pod
  namespace: db
spec:
  serviceAccountName: service-account-web
  containers:
  - image: nginx
    name: web-pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
EOF
kubectl apply -f web-pod.yaml

## 5
mkdir -p /etc/kubernetes/logpolicy/
cat > /etc/kubernetes/logpolicy/sample-policy.yaml << EOF
apiVersion: audit.k8s.io/v1
kind: Policy
# Don't generate audit events for all requests in RequestReceived stage.
omitStages:
  - "RequestReceived"
rules:
  # Don't log watch requests by the "system:kube-proxy" on endpoints or services
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: "" # core API group
      resources: ["endpoints", "services"]
  # Don't log authenticated requests to certain non-resource URL paths.    
  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs:
    - "/api*"  # Wildcard matching.
    - "/version"
  # Please do not delete the above rule content, you can continue add it below.
EOF

## 6
mkdir /cks/sec
kubectl create ns istio-system
kubectl create secret generic db1-test -n istio-system \
    --from-literal=username=lady_killer9 \
    --from-literal=password='123456'


## 7
mkdir -p /cks/docker/
cat >/cks/docker/Dockerfile << EOF
FROM ubuntu:latest
USER root
RUN apt get install -y nginx=4.2
ENV ENV=testing
CMD ["nginx -d"]
EOF
cat >/cks/docker/deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lady_killer9_test
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx1
  template:
    metadata:
      labels:
        app: nginx2
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
        securityContext:
            {'capabilities':{'add':['NET_ADMIN'],'drop':['all']},'privileged': True,'readOnlyRootFilesystem': False,'runAsUser': 65535}
EOF

## 8
#wget https://storage.googleapis.com/gvisor/releases/nightly/latest/runsc
#mv runsc /usr/local/bin/runsc
#chmod +x /usr/local/bin/runsc
#/usr/local/bin/runsc install
#vim /etc/containerd/config.toml
#[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
#          runtime_type = "io.containerd.runsc.v1"
#systemctl daemon-reload
#systemctl  restart containerd
kubectl create ns server
kubectl run nginx --image=nginx -n server --dry-run=client -o yaml > nginx.yaml
kubectl create -f nginx.yaml -n server

## 9
kubectl create ns dev-team
kubectl create ns qaqa
cat >products-service.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: products-service
  name: products-service
spec:
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: products-service
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
EOF
kubectl create -f products-service.yaml -n dev-team

## 10
#sudo apt-get install -y wget apt-transport-https gnupg lsb-release
#wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
#echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
#sudo apt-get update
#sudo apt-get install -y trivy

kubectl create ns kamino
kubectl run nginx --image=nginx -n kamino
kubectl run alpine --image=alpine:3.14 -n kamino

## 11. apparmor
yum install -y apparmor-utils
apt-get install -y apparmor-utils
systemctl start apparmor.service
cat >/etc/apparmor.d/nginx_apparmor << EOF
#include <tunables/global>
nginx-profile-3
profile nginx-profile-3 flags=(attach_disconnected) {
  #include <abstractions/base>
  file,
  # Deny all file writes.
  deny /** w,
}
EOF
mkdir -p /cks/KSSH00401/
cat >/cks/KSSH00401/nginx-deploy.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-deploy
spec:
  containers:
  - name: nginx-deploy
    image: busybox
    command: ["sh","-c","echo 'Hello AppArmor!' && sleep 1h"]
EOF
systemctl enable apparmor.service
apparmor_parser -r /etc/apparmor.d/*

## 12 sysdig

## 13
kubectl create ns sec-ns
kubectl create deploy secdep --image=nginx -n sec-ns

## 14 TLS

## 15
kubectl create clusterrolebinding system:anonymous --clusterrole=admin --serviceaccount=default:default
echo "" > /etc/kubernetes/manifests/kube-apiserver.yaml
cat >/etc/kubernetes/manifests/kube-apiserver.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 172.30.1.2:6443
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=172.30.1.2
    - --allow-privileged=true
    - --authorization-mode=AlwaysAllow
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=AlwaysAdmit
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    - --requestheader-allowed-names=front-proxy-client
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-username-headers=X-Remote-User
    - --secure-port=6443
    - --service-account-issuer=https://kubernetes.default.svc.cluster.local
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
    - --service-cluster-ip-range=10.96.0.0/12
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    image: registry.k8s.io/kube-apiserver:v1.30.0
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 172.30.1.2
        path: /livez
        port: 6443
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    name: kube-apiserver
    readinessProbe:
      failureThreshold: 3
      httpGet:
        host: 172.30.1.2
        path: /readyz
        port: 6443
        scheme: HTTPS
      periodSeconds: 1
      timeoutSeconds: 15
    resources:
      requests:
        cpu: 50m
    startupProbe:
      failureThreshold: 24
      httpGet:
        host: 172.30.1.2
        path: /livez
        port: 6443
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    volumeMounts:
    - mountPath: /etc/ssl/certs
      name: ca-certs
      readOnly: true
    - mountPath: /etc/ca-certificates
      name: etc-ca-certificates
      readOnly: true
    - mountPath: /etc/pki
      name: etc-pki
      readOnly: true
    - mountPath: /etc/kubernetes/pki
      name: k8s-certs
      readOnly: true
    - mountPath: /usr/local/share/ca-certificates
      name: usr-local-share-ca-certificates
      readOnly: true
    - mountPath: /usr/share/ca-certificates
      name: usr-share-ca-certificates
      readOnly: true
  hostNetwork: true
  priority: 2000001000
  priorityClassName: system-node-critical
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  volumes:
  - hostPath:
      path: /etc/ssl/certs
      type: DirectoryOrCreate
    name: ca-certs
  - hostPath:
      path: /etc/ca-certificates
      type: DirectoryOrCreate
    name: etc-ca-certificates
  - hostPath:
      path: /etc/pki
      type: DirectoryOrCreate
    name: etc-pki
  - hostPath:
      path: /etc/kubernetes/pki
      type: DirectoryOrCreate
    name: k8s-certs
  - hostPath:
      path: /usr/local/share/ca-certificates
      type: DirectoryOrCreate
    name: usr-local-share-ca-certificates
  - hostPath:
      path: /usr/share/ca-certificates
      type: DirectoryOrCreate
    name: usr-share-ca-certificates
status: {}
EOF

## 16
mkdir -p /etc/kubernetes/epconfig
cat >/etc/kubernetes/epconfig/admission_configuration.json << EOF
{
  "imagePolicy": {
     "kubeConfigFile": "/etc/kubernetes/kube-image-bouncer.yml",
     "allowTTL": 50,
     "denyTTL": 50,
     "retryBackoff": 500,
     "defaultAllow": true
  }
}
EOF
cat >/etc/kubernetes/epconfig/kubeconfig.yaml << EOF
apiVersion: v1
kind: Config
# clusters refers to the remote service.
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/epconfig/external-cert.pem
    server: server
  name: image-checker
contexts:
- context:
    cluster: image-checker
    user: api-server
  name: image-checker
current-context: image-checker
preferences: {}
users:
- name: api-server
  user:
    client-certificate: /etc/kubernetes/epconfig/apiserver-client-cert.pem
    client-key:  /etc/kubernetes/epconfig/apiserver-client-key.pem
EOF
mkdir -p /cks/img/
cat >/cks/img/web1.yaml << EOF
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-latest
spec:
  replicas: 1
  selector:
    app: nginx-latest
  template:
    metadata:
      name: nginx-latest
      labels:
        app: nginx-latest
    spec:
      containers:
      - name: nginx-latest
        image: nginx
        ports:
        - containerPort: 80
EOF
