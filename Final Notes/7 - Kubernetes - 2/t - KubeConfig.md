![[Pasted image 20260821112835.png]]
 ![[Pasted image 20260821112914.png]]

 ![[Pasted image 20260821113055.png]]

 ![[Pasted image 20260821113212.png]]

![[Pasted image 20260821113226.png]]

![[Pasted image 20260821113243.png]]

![[Pasted image 20260821113257.png]]

![[Pasted image 20260821113559.png]]
 ![[Pasted image 20260821113527.png]]
 ![[Pasted image 20260821113637.png]]
 What context to start
 ![[Pasted image 20260821113749.png]]

 ![[Pasted image 20260821113832.png]]

To not use the default config file
![[Pasted image 20260821113907.png]]

 To change context
 ![[Pasted image 20260821114017.png]]
 `kubectl config --kubeconfig=/root/my-kube-config use-context research`
 ![[Pasted image 20260821114044.png]]

 ![[Pasted image 20260821114129.png]]


 ![[Pasted image 20260821114253.png]]

 ![[Pasted image 20260821114312.png]]

![[Pasted image 20260821114327.png]]



#### To Set a custom config file as the default kubeconig file. Sample situation

We don't want to specify the **kubeconfig file** option on each `kubectl` command.

Set the **`my-kube-config` file** as the **default kubeconfig file** and make it **persistent across all sessions** without overwriting the existing **`~/.kube/config`**. Ensure any configuration changes **persist across reboots** and new **shell sessions**.

**Note**: Don't forget to **source** the configuration file to take effect in the **existing session**. Example:

> ```bash
> source ~/.bashrc
> ```


**Add the** `my-kube-config` **file to the** `KUBECONFIG` **environment variable.**

1. **Open your shell configuration file:**
```bash
vi ~/.bashrc
```

2. **Add one of these lines to export the variable:**
```bash
export KUBECONFIG=/root/my-kube-config
# OR
export KUBECONFIG=~/my-kube-config
# OR
export KUBECONFIG=$HOME/my-kube-config
```
3. **Apply the Changes:**
    **Reload the shell configuration to apply the changes in the current session:**
```bash
source ~/.bashrc
```