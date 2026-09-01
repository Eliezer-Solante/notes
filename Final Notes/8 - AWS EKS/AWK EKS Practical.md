
The practical assessment will take place on KodeKloud's AWS EKS Playground. You will bootstrap the EKS lab using the playground's AWS CloudShell.

1. Using Chrome browser, log in to your KodeKloud account.
    
2. Go to [https://kodekloud.com/cloud-playgrounds/aws](https://kodekloud.com/cloud-playgrounds/aws)
    
3. Log in to the AWS console using the given IAM username and password.
    
4. Open CloudShell and download `eks-bootstrap`. Run the `eks-bootstrap` script to provision the broken EKS cluster. Cluster creation takes about 15 minutes.
    
    ```shell
    wget https://aws-assessment-resources.ap-south-1.linodeobjects.com/aws-eks-v2/eks-bootstrap
    chmod +x eks-bootstrap
    ./eks-bootstrap
    ```
    
5. Do not run `instance-reboot` and `deploy-backend-db` unless explicitly instructed on specific questions.