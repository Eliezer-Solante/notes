![[Pasted image 20260814130951.png]]
c3FsMDE=
cm9vdA==
cGFzc3dvcmQxMjM=
Converting/encode string to a hash
![[Pasted image 20260814131152.png]]
![[Pasted image 20260814132330.png]]

![[Pasted image 20260814132403.png]]

Injecting
![[Pasted image 20260814132454.png]]
![[Pasted image 20260814132535.png]]

![[Pasted image 20260814132612.png]]



---


A **Secret** stores sensitive data — passwords, API tokens, TLS certificates, SSH keys — using the exact same mechanics as a ConfigMap (key-value `data`, no `spec`, same consumption methods), but with handling intended for confidential values. It's worth being upfront about one thing immediately: **Secrets are base64-encoded, not encrypted, by default.** Base64 is just an encoding, not a security measure — anyone with `kubectl get secret -o yaml` access can trivially decode it. Real protection depends on additional cluster configuration, covered below.

## Example YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: my-app-namespace
type: Opaque
data:
  username: YWRtaW4=          # base64 of "admin"
  password: cGFzc3dvcmQxMjM=  # base64 of "password123"
```

You can skip the manual encoding using `stringData` instead — Kubernetes encodes it for you on creation:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: admin
  password: password123
```

## Secret `type` field

Unlike ConfigMap, Secret has a `type` that tells Kubernetes what shape of data to expect:

|Type|Purpose|
|---|---|
|`Opaque`|Generic key-value data (default, most common)|
|`kubernetes.io/tls`|TLS cert + key, expects `tls.crt` / `tls.key`|
|`kubernetes.io/dockerconfigjson`|Docker registry credentials for pulling private images|
|`kubernetes.io/basic-auth`|Username/password pair|
|`kubernetes.io/ssh-auth`|SSH private key|

## Consuming a Secret — same three patterns as ConfigMap

**1. Individual env var**

```yaml
containers:
  - name: app
    image: my-app:1.0
    env:
      - name: DB_PASSWORD
        valueFrom:
          secretKeyRef:
            name: db-credentials
            key: password
```

**2. All keys at once**

```yaml
containers:
  - name: app
    envFrom:
      - secretRef:
          name: db-credentials
```

**3. Mounted as files (generally the preferred method for secrets)**

```yaml
containers:
  - name: app
    volumeMounts:
      - name: secret-volume
        mountPath: /etc/secrets
        readOnly: true
volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
```

Mounting as files is often recommended over env vars for secrets specifically, because environment variables can leak more easily — they show up in `kubectl describe pod`, crash dumps, child process environments, and logging tools that dump env state. Mounted files are comparatively more contained.

## Common commands

```bash
# apply from YAML
kubectl apply -f secret.yaml

# create imperatively from literals
kubectl create secret generic db-credentials --from-literal=username=admin --from-literal=password=password123

# create a TLS secret
kubectl create secret tls my-tls-secret --cert=cert.crt --key=cert.key

# create a docker registry pull secret
kubectl create secret docker-registry regcred --docker-username=user --docker-password=pass --docker-server=myregistry.io

# list (values are hidden in normal output)
kubectl get secrets

# view — note this shows base64, not encryption
kubectl get secret db-credentials -o yaml

# decode a specific value manually
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 --decode

# delete
kubectl delete secret db-credentials
```

## Actually securing Secrets — beyond default behavior

Since base64 alone isn't protection, real Secret security relies on layering these on top:

- **Encryption at rest** — enabled at the cluster level (`EncryptionConfiguration`) so Secrets are actually encrypted in etcd, not just base64-encoded.
- **RBAC** — restrict `get`/`list` on `secrets` to only the service accounts/users that genuinely need them (this is the piece we flagged as a "next topic" back in the namespaces discussion).
- **Avoid committing Secret YAML to git** — since `stringData`/`data` are plaintext-adjacent, tools like **Sealed Secrets** or **External Secrets Operator** (pulling from Vault, AWS Secrets Manager, etc.) exist specifically so real secret values never sit in a git repo.

## How this ties into everything so far

Same container-spec slot as ConfigMap (`env`, `envFrom`, `volumeMounts` — right alongside `command`/`args`/`image` from the last message), same namespace-scoping behavior we covered (a Secret in `dev` isn't visible to pods in `default`), and the same "mounted volumes auto-update, env vars don't" caveat applies here too.


### INJECTING TO A POD

### Secret — inject into a running pod

**Step 1 — edit the Secret**

bash

```bash
kubectl edit secret db-credentials -n my-app-namespace

# or patch — stringData lets you skip manual base64 encoding
kubectl patch secret db-credentials -n my-app-namespace \
  --type merge -p '{"stringData":{"password":"newpass123"}}'
```

**Step 2 — get the running pod to pick it up**

Identical behavior to ConfigMap: **mounted volume** → file auto-updates within ~60-90 seconds; **env var** → frozen at container start, never updates on its own.

bash

```bash
kubectl exec -it my-app-pod -- cat /etc/secrets/password
```

Because Secrets are more sensitive, it's worth double-checking that whatever picked up the new value doesn't leak it — e.g. avoid `kubectl exec ... env` in shared terminals/logs if the Secret is injected as an env var.

**Step 3 — force a reload**

bash

```bash
kubectl rollout restart deployment my-app -n my-app-namespace
kubectl rollout status deployment/my-app -n my-app-namespace
```

Same bare-pod caveat applies — delete and reapply if there's no Deployment managing it.

### The one real difference between the two

![[Pasted image 20260814131700.png]]


Remember, you CANNOT edit specifications of an existing POD other than the below.

- spec.containers[*].image
- spec.initContainers[*].image
- spec.activeDeadlineSeconds
- spec.tolerations
    

For example, you cannot edit the environment variables, service accounts, and resource limits (all of which we will discuss later) of a running pod. But if you really want to, you have two options:

1. Run the `kubectl edit pod` command. This will open the pod specification in an editor (vi editor). Then, edit the required properties. When you try to save it, you will be denied. This is because you are attempting to edit a field on the pod that is not editable.

![Image](https://res.cloudinary.com/cloudusthad/image/upload/v1784803520/courses/b1a5c933-8fde-445e-9fd0-d3404f46b2f8.png)

![Image](https://res.cloudinary.com/cloudusthad/image/upload/v1784803534/courses/8fd62e04-adc6-4647-a252-b802ed097f6f.png)

A copy of the file with your changes is saved in a temporary location, as shown above.

You can then delete the existing pod by running the command:

```
kubectl delete pod webapp
```

Then, create a new pod with your changes using the temporary file:

```
kubectl create -f /tmp/kubectl-edit-ccvrq.yaml
```

2. The second option is to extract the pod definition in YAML format to a file using the command

```
kubectl get pod webapp -o yaml > my-new-pod.yaml
```

Then, make the changes to the exported file using an editor (vi editor). Save the changes

```
vi my-new-pod.yaml
```

Then, delete the existing pod.

```
kubectl delete pod webapp
```

Then, create a new pod with the edited file.

```
kubectl create -f my-new-pod.yaml
```

#### Edit Deployments

With Deployments, you can easily edit any field/property of the POD template. Since the pod template is a child of the deployment specification, with every change, the deployment will automatically delete and create a new pod with the new changes. So, if you are asked to edit a property of a POD part of a deployment, you may do that simply by running the command.

```
kubectl edit deployment my-deployment