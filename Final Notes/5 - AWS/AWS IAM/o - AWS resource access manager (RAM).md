![[Pasted image 20260805162809.png]]This diagram illustrates the **AWS RAM workflow** — the step-by-step process of creating a Resource Share. It maps directly to what we just discussed, showing the flow visually:

## The 4-Step RAM Workflow

**1. AWS Resource Access Manager (starting point)** Share resources across AWS accounts or AWS Organizations by creating a **Resource Share** — this is the container/object that holds everything below.

**2. Select Resources** Choose the specific resource(s) you want to include in the Resource Share (e.g., a VPC subnet, Transit Gateway, License Manager configuration). The icons here (grid, lock, target) represent generic shareable resource types.

**3. Specify Principals** Define **who** should get access to those resources — this can be:

- Specific AWS **account(s)**
- An **Organizational Unit (OU)**
- The entire **AWS Organization**

**4. Share Resources** Once configured, the specified principals (accounts/OUs) **immediately gain access** to use the shared resource(s) — without needing to duplicate the resource or set up separate cross-account IAM roles.

## Key Insight from the Diagram

Notice the **dashed border wrapping the entire flow** — this represents the **Resource Share** itself as a single object. Everything inside it (the selected resources + the specified principals) is bundled together as one "Resource Share," which is the actual construct you create and manage in RAM.

## Simplified Mental Model

```
Resource Share = Resources (what) + Principals (who)
```

Once both are defined and the share is created, RAM handles granting the appropriate access automatically — the principals can now use the shared resource directly in their own account.

## Connecting to Real-World Use

This matches the multi-account example from before: a **networking account** creates a Resource Share containing VPC subnets (Select Resources), specifies the Dev/Staging/Prod account IDs (Specify Principals), and once shared (Share Resources), those accounts can launch EC2 instances directly into the shared subnets — no VPC peering, no duplicate infrastructure, no per-resource IAM cross-account roles needed.