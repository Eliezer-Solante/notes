![[Pasted image 20260814111340.png]]
![[Pasted image 20260814111331.png]]

In Kubernetes, "commands and arguments" refers to how you override what a container actually runs when it starts — this is a container-spec feature, distinct from `kubectl` commands themselves (which we've been using throughout this conversation).

## The core concept

Every container image has a built-in **ENTRYPOINT** and **CMD** baked in by its Dockerfile — these define what process runs when the container starts. Kubernetes lets you override either one directly in the pod spec using two fields: `command` and `args`.

|Kubernetes field|Overrides Docker's...|Purpose|
|---|---|---|
|`command`|`ENTRYPOINT`|The actual executable to run|
|`args`|`CMD`|Arguments passed to that executable|

## Example YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-command-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["/bin/sh", "-c"]
      args: ["echo Hello from Kubernetes && sleep 3600"]
```

This is equivalent to running:

```bash
/bin/sh -c "echo Hello from Kubernetes && sleep 3600"
```

## Why this matters in practice

- **Debugging generic images** — `busybox`, `alpine`, `ubuntu` don't have a long-running default process, so pods using them exit immediately unless you give them something to do:

```yaml
containers:
  - name: debug
    image: busybox
    command: ["sleep"]
    args: ["3600"]
```

- **Reusing one image for different jobs** — instead of building separate images, you can run the same image with different `args` to perform different tasks (common in Jobs/CronJobs).
    
- **Passing environment-driven arguments**:
    

```yaml
containers:
  - name: app
    image: my-app:1.0
    command: ["./start.sh"]
    args: ["--config=/etc/app/config.yaml", "--verbose"]
```

## The four combinations, and what happens

|Dockerfile has|Pod spec has|Result|
|---|---|---|
|ENTRYPOINT + CMD|nothing|both from image are used|
|ENTRYPOINT + CMD|only `args`|image's ENTRYPOINT + your `args` (replaces CMD)|
|ENTRYPOINT + CMD|only `command`|your `command` replaces ENTRYPOINT; image's CMD is ignored entirely|
|ENTRYPOINT + CMD|both `command` and `args`|both from image are fully overridden|

This trips people up most on the third row — setting `command` alone silently discards the image's default `CMD`, even if you didn't touch `args`.

## Verifying at runtime

```bash
kubectl apply -f pod.yaml

# confirm what actually ran
kubectl logs custom-command-pod

# inspect the resolved command/args on a live pod
kubectl get pod custom-command-pod -o jsonpath='{.spec.containers[0].command}'
kubectl describe pod custom-command-pod   # shows Command/Args in the output
```

## How this connects to what we've covered

This slots into the same `containers:` block we've used in every Pod/Deployment YAML so far — it sits alongside `image`, `ports`, `resources`, and `env`. It's especially relevant for **init containers** (which we mentioned briefly earlier) — those almost always rely on a custom `command`/`args` to run a one-off setup task rather than a long-running service.



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
```