![[Pasted image 20260822014401.png]]

![[Pasted image 20260822014551.png]]
![[Pasted image 20260822014622.png]]

![[Pasted image 20260822014646.png]]


INSTALLING HELM

![[Pasted image 20260822014834.png]]

ubuntu
```bash
sudo apt-get install curl gpg apt-transport-https --yes
curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```