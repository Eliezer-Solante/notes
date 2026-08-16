![[Pasted image 20260730133834.png]]![[Pasted image 20260730133930.png]]
## 📦 Docker Images

- **Definition**: A Docker image is a read-only template that contains everything needed to run an application — code, runtime, libraries, environment variables, and configuration files.
    
- **Layers**: Images are built in layers, meaning changes (like installing a package) create a new layer. This makes them efficient and reusable.
    
- **Dockerfile**: Images are usually built from a Dockerfile, which is a script of instructions (e.g., `FROM ubuntu`, `RUN apt-get install python3`).
    
- **Immutable**: Once built, images don’t change. If you need updates, you create a new image version.
    

## 🗄️ Docker Registries

- **Definition**: A registry is a storage and distribution system for Docker images.
    
- **Docker Hub**: The default public registry where millions of prebuilt images are available (e.g., `nginx`, `mysql`).
    
- **Private registries**: Companies often host their own registries for internal images, ensuring security and control.
    
- **Pull & Push**:
    
    - **Pull**: Download an image from a registry (`docker pull nginx`).
        
    - **Push**: Upload your custom image to a registry (`docker push myapp:v1`).

    
![[Pasted image 20260730134133.png]]
![[Pasted image 20260730134159.png]]
The relationship between **Docker images** and **containers** is like the difference between a **blueprint** and a **house built from that blueprint**.

## 📦 Docker Images

- **Definition**: A read-only template that defines what goes inside a container (app code, libraries, dependencies, configs).
    
- **Static**: Images don’t change once built; they’re immutable snapshots.
    
- **Source**: Created from a Dockerfile or pulled from a registry like Docker Hub.
    

## 🏠 Docker Containers

- **Definition**: A running instance of an image.
    
- **Dynamic**: Containers can be started, stopped, modified, or deleted.
    
- **Lifecycle**: When you run an image, Docker creates a container process based on it.
    

## 🔗 Relationship

- **Image → Container**: You can think of an image as the recipe, and a container as the dish prepared from that recipe.
    
- **Multiple containers from one image**: You can run many containers from the same image (e.g., multiple instances of `nginx`).
    
- **Containers depend on images**: Without an image, you cannot create a container.



