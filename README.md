# GinFlix: Video on Demand application with Kubernetes


## Introduction
This project is a scalable Video on Demand (VoD) application designed for streaming video content through a web interface built with Django. The system architecture includes a PostgreSQL database for storing videos' metadata, an Nginx streamer server for video delivery, and a Caddy reverse proxy to unify access points. The entire infrastructure is deployed on a Kubernetes cluster, ensuring high availability, and scalability, with persistent storage for both the database and the streamer servers.

## Getting Started
### Required Packages
We run the K8s cluster using Kind (Kubernetes in Docker), hence the first thing we need to have is Docker. Refer to this [Docker installation guide](https://docs.docker.com/engine/install/)  
Once Docker installed allow your user to run Docker:
```bash
sudo usermod -aG docker $USER
```
Then restart the terminal to take changes into account.  
Now we can install `kind` with the following commands:
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.16.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/bin/kind
```
Finally we need to have `kubectl` to communicate with the k8s cluster's control plane.
```bash
sudo snap install kubectl --classic
echo 'source <(kubectl completion bash)' >> ~/.bashrc
exec bash
```
### Deploying the Service
In order to automate the process of creating the infrastructure, and launching the application, we use a Makefile that combine each set of related commands. This makes it very easy and fast to start or stop our application.
#### Creating the K8s Cluster
```bash
make cluster-create
```
#### Starting the Servers
```bash
make start
```
#### Copying a Test Video to the Streamer
```bash
make copy
```
#### Adding the Video to the Web Application
- Open the application at the address `http://localhost/admin`
- Login using the username:`admin@ginflix.com` and password:`admin`
- Use the menu tab `MAIN/Movies +Add` to add a new video
- In the Link entry write: `http://localhost/stream/stock.mp4`
- Fill the other entries with arbitrary information and press `Save`
- Now in the `http://localhost/` page you should see the new video you have just added, and clicking on it should start the video streaming
#### Cleaning
```bash
make clean
```

## System Architecture
### Overview of the Architecture
The video on demand application is built on a scalable architecture comprising four main components: a PostgreSQL database server for storing video metadata, a Django-based web server for presenting the user interface and managing video information, an Nginx-based streamer server for delivering video content, and a Caddy reverse proxy to unify access to the web and streamer servers.

![architecture](./assets/architecture.png)
### Database Server
The PostgreSQL database server is responsible for storing metadata about the videos available for streaming. It maintains information such as video titles, descriptions, and file locations. The web server interacts with the database server to read and write metadata, enabling users to add new video information and retrieve details such as the number of video views.
### Web Server
The web server hosts a Django application that provides the user interface for the video on demand service. It allows users to add and manage video metadata and to view available videos. The web server communicates directly with the PostgreSQL database server to perform CRUD (Create, Read, Update, Delete) operations on video metadata. Additionally, it provides endpoints for the reverse proxy to access the web application.
### Streaming Server
The streamer server runs an Nginx application that is responsible for serving the video content to users. Ideally we'd have more than one streamer server. Each streamer server handles video streaming requests by accessing the video files provided at a certain destination folder (`/var/www/html/`). It is the server responsible for delivering the video content to the end-users.
### Reverse Proxy
The Caddy reverse proxy serves as the gateway for external access to the application. It listens to incoming requests and routes them to either the web server or the streamer servers based on the request path. By managing the external endpoint, Caddy ensures that users can interact with the web interface and stream videos from a single unified address, abstracting the internal server structure.

## Implementation Details
### Postgres Database Server
- **Implementation**: The PostgreSQL database server is implemented as a Kubernetes deployment. This deployment ensures that the database service runs consistently and can be scaled if necessary. The database itself is stored in a persistent volume to ensure data availability.
- **Deployment**: A Kubernetes service of type ClusterIP is used to provide internal network connectivity to the PostgreSQL database. This service allows the web server to communicate with the database.
- **Communication**: The web server communicates with the PostgreSQL database server through the Kubernetes service. This allows the web server to perform CRUD operations on the database, such as adding video metadata or retrieving information about available videos.
### Django Web Server
- **Implementation**: The web server is implemented as a Django application running within a Docker container. The Docker container is then deployed as a Kubernetes deployment, ensuring that the web server can be easily managed and scaled.
- **Deployment**: A Kubernetes service of type NodePort is used to expose the web server to external traffic. This service allows users to access the web interface of the video on demand application from outside the Kubernetes cluster.
### Nginx Streaming Server
- **Implementation**: The streamer server is implemented as an Nginx application running withing a Docker container, that is then deployed as a Kubernetes deployment. A configMap is used in order to pass the default website configuration to the Nginx container
- **Deployment**: A Kubernetes service of type NodePort is used to expose the streamer server to external traffic. This service allows users to access the streaming functionality of the application from outside the Kubernetes cluster.
- **Communication**: The streamer server does not directly communicate with the other servers. Instead, it serves video content to users who request it through the web server. Users interact with the web interface provided by the web server, which in turn communicates with the streamer server to stream videos.
### Caddy Reverse Proxy
- **Implementation**: The reverse proxy server is implemented as a Caddy server running within a Docker container. The Caddy server is configured to route incoming requests to the appropriate backend servers based on the request path.
- **Deployment**: The Caddy server is deployed as a standalone Docker container. It is not managed by Kubernetes but runs alongside the Kubernetes cluster
- **Communication**: The reverse proxy server does not directly communicate with the other servers. Instead, it acts as a middleman, routing incoming requests to the appropriate backend server based on the requested URL path. This allows users to access both the web interface and streaming functionality of the application through a single external endpoint.

## References
- [Kubernetes training](https://gitlab.telecom-paris.fr/jean-louis.rougier/k8s-hands-on-training)
- [Kubernetes Up & Running book](https://www.oreilly.com/library/view/kubernetes-up-and/9781098110192/)