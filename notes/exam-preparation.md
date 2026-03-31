# Exam Prep

## Key Considerations

1. Always verify your context - Use kubectl config current-context
2. Use dry-run for validation - `--dry-run --debug` helps catch errors
3. Know your rollback options - Practice helm history and helm rollback
4. Understand values precedence - Command line > values file > chart defaults
5. Master troubleshooting commands - helm status , helm get values , kubectl describe

## Troubleshooting

## API Server

Check if the container is running:

- `watch docker/crictl ps`

You might find logs here:

- `crictl logs CONTAINER_ID`
- `/var/log/pods/kube-system...` (check with `tail -f` or `cat`)
- `cat /var/log/syslog | grep apiserver`
- `journalctl | grep apiserver`

If the API server cannot connect to another host, this might have to do with the `etcd` connection. Check the manifest `/etc/kubernetes/manifests/...` for the configuration and the `etcd` pod/service for the IP and port of `etcd`.

### Kubelet

Check the status of the kubelet (on the node that has problems) with `systemctl status kubelet`.

There might be `kubeadm` and `kubelet` configuration files at `/var/lib/kubelet`.

### Deployments/Pods

You can get logs for all containers of a deployment with `kubectl logs deployments/... --all-containers=true`.

## Pod

- To quickly create a pod for further customization: `kubectl run nginx --image=nginx --dry-run=client -oyaml >> nginx.yaml`
- Enter a pod with `kubectl exec --stdin --tty POD -- /bin/bash` (or `sh`, or ...)
- Pod priority is handled with PriorityClass

To execute some commands on startup and write to host files, you can use something like this:

```yaml
spec:
  containers:
    - name: executor
      image: bash
      command:
        - sh
        - -c
        - 'echo "aba997ac-1c89-4d64\ >> /configurator/config && sleep 1d'
      volumeMounts:
        - mountPath: /configurator/config
          name: configurator-volume
          readOnly: false
  volumes:
    - name: configurator-volume
      hostPath:
        path: /configurator/config
        type: FileOrCreate
```

### (Anti-)Affinity

- For prefer rules use `weight`
- You can use `matchLabels` and `matchExpressions`
- For `podAffinity` use `podAffinityTerm`
- Make sure the topology key is correct

## ConfigMaps

Add as a volume with the following spec:

```yaml
spec:
  containers:
    - name: mypod
      image: redis
      volumeMounts:
        - name: foo
          mountPath: "/etc/foo"
          readOnly: true
  volumes:
    - name: foo
      configMap:
        name: myconfigmap
```

Add as env with e.g.:

```yaml
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        # Define the environment variable
        - name:
            PLAYER_INITIAL_LIVES # Notice that the case is different here
            # from the key name in the ConfigMap.
          valueFrom:
            configMapKeyRef:
              name: game-demo # The ConfigMap this value comes from.
              key: player_initial_lives # The key to fetch.
```

## Service

Quickly create a service with `kubectl expose deployments/NAME --type=ClusterIP --port=80`.

## Ingress

First, check for the ingress class: `kubectl get ingressclass`

Quickly create with `kubectl create ingress --class=... --rule="domain.com/*=svc:80" ...`.

To set up a Ingress the first time:

1. Create a namespace
2. Install the chart with its configurations
3. Check its status: `helm status NAME -n NAMESPACE`
4. And list all releases with `helm list -A`

Step 2:

```sh
helm install my-nginx-ingress nginx-stable/nginx-ingress \
   --namespace nginx-ingress \
   --set controller.replicaCount=2 \
   --set controller.service.type=NodePort \
   --set controller.service.httpPort.nodePort=30080 \
   --set controller.service.httpsPort.nodePort=30443
```

To provide a yaml with values to a release you can use:

```sh
helm upgrade my-nginx-ingress nginx-stable/nginx-ingress \
  --namespace nginx-ingress \
  --values /root/nginx-values.yaml
```

You can verify the values with `helm get values`.

Rollback with `helm rollback CHART NUMBER -n NAMESPACE`.

## NetworkPolicy

Use this for learning: https://editor.networkpolicy.io. Pay special attention to `PolicyTypes` array on top level of spec.

## RBAC

To apply some permissions in multiple namespaces, use a clusterrole but apply a rolebinding per namespace.

## Setting up / maintaining a cluster

- Use `kubeadm init` on the control plane node: https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/
- Then `kubeadm token create --print-join-command`
- Execute the join command on a worker node

### Cluster Certificate Management

- Check certificates with `kubeadm certs` and its subcommands

## Kubernetes Metrics Server

- To get metrics for the containers of a pod: `kubectl top pod NAME --containers` and sort with `--sort-by=...`

## Helm

- Make sure that helm is installed, add the repo for the package and search with `helm search repo`

## User Management

First, create a token (e.g. 4096 RSA) with `openssl genrsa -out user.key 4096`
Afterwards, create a CSR `openssl req -new -key user.key -out user.csr -subj="/CN=user/O=org"`. Then, create a CertificateSigningRequest in Kubernetes:

```sh
cat > /root/gameforge-onboarding/siddhi-csr.yaml <<EOF
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: user
spec:
  request: $(cat user.csr | base64 | tr -d "\n")
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 31536000  # 1 year (365 days)
  usages:
  - client auth
EOF
```

With this you can use `k certificate approve NAME` to approve the CSR. To then read out the decoded certificate: `kubectl get csr NAME -o jsonpath='{.status.certificate}' | base64 -d `

You can verify the certificate with `openssl x509 -in user.crt -text -noout`. Finally you can add the user entry with `k config set-credentials USER --client-certificate ... --client-key ...`. Use `--embed-certs` to embed the certificates in kubeconfig. You can export a kubeconfig file with: `k config view --minify --flatten --context=CONTEXT`

## Testing communication between services

1. Use internal hostnames: `SERVICE.NAMESPACE.svc.cluster.local`

### Misc

#### Secrets within pods

Check if e.g. a service account token is mounted to a pod with: `kubectl exec POD -- cat /var/run/secrets/kubernetes.io/serviceaccount/token`.
