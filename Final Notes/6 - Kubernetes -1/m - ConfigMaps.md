![[Pasted image 20260814115521.png]]

![[Pasted image 20260814115558.png]]


![[Pasted image 20260814115633.png]]
![[Pasted image 20260814115652.png]]

![[Pasted image 20260814115745.png]]
![[Pasted image 20260814115830.png]]

A **ConfigMap** stores non-sensitive configuration data as key-value pairs, separately from your container image — so you can change config without rebuilding the image or hardcoding values into your Deployment YAML. (For sensitive data like passwords or tokens, you'd use a `Secret` instead — same mechanics, different intent, which we listed together back in the kind/apiVersion table.)

## Why it exists

Without it, you'd either bake config into the image (rebuild every time a setting changes) or hardcode env vars directly in the Deployment spec (mixes app config with infra definition, and can't be reused across multiple Deployments). A ConfigMap decouples the two.

## Example YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: my-app-namespace
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
```

Notice — like `Secret` — there's no `spec` field; it's just `data` directly, since a ConfigMap isn't describing running behavior, just storing values. Values are always strings; you can store a single key-value pair or an entire config file as one multi-line value (like `config.yaml` above).

## Three ways to consume a ConfigMap in a pod

**1. As individual environment variables**

```yaml
containers:
  - name: app
    image: my-app:1.0
    env:
      - name: APP_ENV
        valueFrom:
          configMapKeyRef:
            name: app-config
            key: APP_ENV
```

**2. As a full set of environment variables at once**

```yaml
containers:
  - name: app
    image: my-app:1.0
    envFrom:
      - configMapRef:
          name: app-config
```

This dumps every key in the ConfigMap into the container's environment automatically — less explicit, but less repetitive.

**3. As mounted files (a volume)**

```yaml
containers:
  - name: app
    image: my-app:1.0
    volumeMounts:
      - name: config-volume
        mountPath: /etc/app
volumes:
  - name: config-volume
    configMap:
      name: app-config
```

This creates a file at `/etc/app/config.yaml` inside the container, containing exactly the multi-line value from the ConfigMap — useful when your app expects a config _file_, not env vars.

## Common commands

```bash
# apply from YAML
kubectl apply -f configmap.yaml

# create imperatively from literals
kubectl create configmap app-config --from-literal=APP_ENV=production --from-literal=LOG_LEVEL=info

# create from an existing file
kubectl create configmap app-config --from-file=config.yaml

# list
kubectl get configmaps
kubectl get cm -n my-app-namespace

# view contents
kubectl describe cm app-config
kubectl get cm app-config -o yaml

# edit live
kubectl edit cm app-config

# delete
kubectl delete cm app-config
```

## An important gotcha worth knowing

**Env vars sourced from a ConfigMap do NOT auto-update in a running pod** if you edit the ConfigMap afterward — the pod needs to be restarted (or recreated) to pick up new values, since env vars are injected only at container start. **Mounted-file ConfigMaps behave differently** — the file inside the pod _does_ update automatically after a short delay (usually under a minute), because the kubelet periodically re-syncs mounted ConfigMap volumes. This is one of the more common points of confusion when people expect a live-reload and don't get one via env vars.

## How this connects to what we've covered

This slots directly into the `containers:` block from the command/args discussion — `env`, `envFrom`, and `volumeMounts` all sit alongside `command`/`args`/`image` in the same container spec we've been building up across this conversation.



SAMPLE
Update the environment variable on the POD to use only the `APP_COLOR` key from the newly created ConfigMap.

Note: Delete and recreate the POD. Only make the necessary changes. Do not modify the name of the Pod.


```yaml
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: webapp-color
  name: webapp-color
  namespace: default
spec:
  containers:
  - env:
    - name: APP_COLOR
      valueFrom:
       configMapKeyRef:
         name: webapp-config-map
         key: APP_COLOR
    image: kodekloud/webapp-color
    name: webapp-color
```


### INJECTING TO A POD

### ConfigMap — inject into a running pod

**Step 1 — edit the ConfigMap**

bash

```bash
kubectl edit configmap app-config -n my-app-namespace

# or patch a specific key directly
kubectl patch configmap app-config -n my-app-namespace \
  --type merge -p '{"data":{"LOG_LEVEL":"debug"}}'
```

**Step 2 — get the running pod to pick it up**

If mounted as a **volume**, the file updates automatically inside the container within ~60-90 seconds:

bash

```bash
kubectl exec -it my-app-pod -- cat /etc/app/config.yaml
```

Most apps still won't reload just because the file changed — they read it once at startup and cache it in memory. If injected as **env vars**, there's no auto-update at all — env vars are frozen at container start, full stop.

**Step 3 — force a reload if the app won't pick it up on its own**

bash

```bash
# Deployment-managed pods — safest, does a rolling restart
kubectl rollout restart deployment my-app -n my-app-namespace
kubectl rollout status deployment/my-app -n my-app-namespace

# bare pod (no Deployment) — no rolling restart exists, just recreate
kubectl delete pod my-app-pod
kubectl apply -f pod.yaml
```


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