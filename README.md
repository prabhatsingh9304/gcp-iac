[Albumstore Application on GCP](http://34.110.198.184/)

Click On Title To Open App

1 Overview

The albumstore application is a web-based platform for browsing and purchasing music albums. It is built using Go for the backend and React for the frontend. Deployed on Google Cloud Platform (GCP), it leverages GCP’s infrastructure to ensure scalability, reliability, and security.

2 Architecture Diagram

![](/readme/Aspose.Words.b4fd99a1-c490-4397-a031-082f372c21de.001.png)

Figure 1: Architecture Diagram

3 Backend: Go Application

1. Model

1 type album struct {![](/readme/Aspose.Words.b4fd99a1-c490-4397-a031-082f372c21de.002.png)

2 ID string ‘json:"id"‘

3 Title string ‘json:"title"‘ 4 Artist string ‘json:"artist"‘ 5 Price float64 ‘json:"price"‘ 6 }

2. Deployment

The backend application is containerized and deployed using Docker.

1. Dockerfile for Backend
   1 FROM golang:1.22.5![](/readme/Aspose.Words.b4fd99a1-c490-4397-a031-082f372c21de.003.png)
   1 WORKDIR /
   1 COPY . .
   1 RUN go mod download
   1 RUN go build -o main . 6

7 EXPOSE 8080

8

9 CMD ["./main"]

2. Build and Push Docker Image
   1 docker build --platform linux/amd64 --build-arg DB_URL=’postgres://<username >:<password >@<db_private_ip >:5432/<db_name![](Aspose.Words.b4fd99a1-c490-4397-a031-082f372c21de.004.png) >?sslmode=disable’ -t gcr.io/batch -2024-training/backend -go:latest .
   1 gcloud auth configure -docker
   1 docker push gcr.io/batch -2024-training/backend -go:latest

   4 Frontend: React Application

1. Dockerfile for Frontend

To ensure security and efficiency, the frontend application is built and served using NGINX.

1 # Build stage![](/readme/Aspose.Words.b4fd99a1-c490-4397-a031-082f372c21de.005.png)
1 FROM node:20.15.1 AS build
1 WORKDIR /app
1 COPY package\*.json ./
1 RUN npm install
1 COPY . .
1 ARG REACT_APP_API_URL
1 ENV REACT_APP_API_URL=${REACT_APP_API_URL}
1 RUN npm run build

10

11 # Production stage
11 FROM nginx:alpine
11 COPY --from=build /app/build /usr/share/nginx/html
11 EXPOSE 80
11 CMD ["nginx", "-g", "daemon off;"] 2. Environment Variables

The frontend uses the REACTAPPAPI~~ URLenvironment variable during the build to secure the API URL.![](/readme/Aspose.Words.b4fd99a1-c490-4397-a031-082f372c21de.006.png)![](/readme/Aspose.Words.b4fd99a1-c490-4397-a031-082f372c21de.007.png)

3. Build and Push Docker Image
   1 docker build --platform linux/amd64 -t gcr.io/batch -2024-training/albumstore -frontend:latest --build-arg ![](/readme/Aspose.Words.b4fd99a1-c490-4397-a031-082f372c21de.008.png)REACT_APP_API_URL=http://<load balancer public IP>/api .
   1 gcloud auth configure -docker
   1 docker push gcr.io/batch -2024-training/albumstore -frontend:latest
   5 Architecture Overview
1. Components
1. Client

The client accesses the AlbumBook application via a web browser or HTTP client, sending requests to the public IP of the GCP Load Balancer.

2. GCP Load Balancer

The load balancer manages incoming HTTP requests and distributes them to the appropriate instances.

Public IP: The IP address used by clients to access the application.

Traffic Routing: Configured to route requests based on URL mapping rules.

URL Mapping Configuration:

- Default Service: Routes requests that do not match any specific path pattern to the frontend instance group. defaultService: projects/batch-2024-training/global/backendServices/albumstore-f
- Route Rule for API Requests:
- Match Rule: Requests with the path pattern /api/{path=\*\*} are routed to the backend service.
- Service: Forwards these requests to the backend service (albumstore-b).
- URL Rewrite: Removes the /api prefix from the path before forwarding.
- Configuration:

routeRules:

- matchRules:

  - pathTemplateMatch: /api/{path=\*\*}

    priority: 1

    service: projects/batch-2024-training/global/backendServices/albumstore-b routeAction:

urlRewrite:

pathTemplateRewrite: /{path}

3. Frontend Instance Group

An auto-scaling group of VMs running the frontend application.

Auto-Scaling: Adjusts the number of instances based on traffic.

Dockerized React App: Each instance runs a Docker container hosting the React application. Private IP: Used for internal communication within the GCP network.

4. Backend Instance Group

An auto-scaling group of VMs running the Go application.

Dockerized Go App: The Go application is containerized.

API Handling: Processes API requests routed by the load balancer. Private IP: Used for internal communication within the VPC.

5. Database (PostgreSQL 16.0)

Hosted on a separate VPC managed by GCP.

Private IP and VPC Peering: The database uses a private IP (10.138.10.2) and is peered with the albumstore-VPC for secure communication.

2. Request Flow
1. Client Request

Clients access the public IP of the load balancer, e.g., http://34.110.198.184/.

2. Load Balancer Routing

Routes requests based on the path:

- Unmapped Paths or Root ( / ): Routed to the frontend instance group.
- API Paths ( /api/ {path }): Routed to the backend instance group with URL rewriting.

3. Frontend-Backend Interaction

Frontend makes API calls to the backend via the load balancer.

4. Database Interaction

Backend instances interact with the PostgreSQL database using the private IP over the peered VPC.

5. Response

The backend processes requests and sends responses back through the load balancer to the frontend, which then delivers it to the client.

This architecture provides a robust and scalable deployment for the AlbumBook application on GCP, with effective URL routing and load balancing.

6 Design

1. Service Account Creation

The first step in the design process involved creating a service account named albumstore. This service account was assigned a predefined role of Basic Editor, providing it with the necessary permissions to manage resources within the project.

2. Virtual Private Cloud (VPC) Creation

Using the albumstore service account, a Virtual Private Cloud (VPC) named albumstore-vpc was created. This VPC provides network isolation and control over the networking environment for the AlbumBook application.

3. Subnet Configuration

Within the albumstore-vpc, a subnet was created with an IP range of 10.138.0.0/24. This subnet allocates IP address ranges for the resources in the project, enabling backend and frontend instances to communicate securely within a private IP space.

4. Firewall Rules Setup

To secure and manage traffic within the VPC, two custom firewall rules were created:

- Custom Firewall Rule 1: This rule authorizes all IP ranges within the subnet to access all ports, ensuring seamless communication between instances in the same subnet.
- Custom Firewall Rule 2: This rule allows SSH access from any IP address ( 0.0.0.0/0 ), enabling secure remote management of the instances.

5. Backend Instance Template and Group Setup

For the backend services, an instance template was created using a Docker image stored in Google Container Registry (GCR). This Docker image contains the Go application that serves the backend API for the AlbumBook platform.

- Instance Template: The instance template encapsulates the configuration of the virtual machine, including the Docker image, machine type, and other settings.
- Instance Group: An instance group was created from this template to enable auto-scaling and high availability for the backend services. The group dynamically adjusts the number of instances based on incoming traffic.

6. Backend Load Balancer Configuration

A Google Cloud Load Balancer was set up to distribute incoming requests to the backend instance group. The load balancer provides an external IP, which clients use to access the API. The URL mapping was configured to route /api/{request} paths to the backend instances, while any unmatched requests are redirected accordingly.

7. Frontend Instance Template and Group Setup

For the frontend, another instance template was created, based on a Docker image that includes NGINX and the React ap- plication. The React application is configured to use the Load Balancer’s external IP in its API URL, ensuring seamless communication with the backend services.

- Instance Template: This template defines the configuration for the frontend virtual machines, including the Docker image containing the React app and NGINX.
- Instance Group: Similar to the backend, a frontend instance group was established using this template to ensure scalability and high availability for the frontend services.

8. Frontend Load Balancer Configuration

Finally, the Google Cloud Load Balancer was configured for the frontend instance group. The load balancer’s URL mapping directs any unmatched paths to the frontend instances, effectively serving the React application to users accessing the AlbumBook platform.

7 Logging and Monitoring

1. Dashboard Creation

To effectively monitor the performance and health of the AlbumBook application, a custom dashboard named albumstore-dashboard was created. This dashboard provides a centralized view of key metrics and logs, enabling quick identification and resolution of issues.

7\.1.1 Widgets

The albumstore-dashboard includes the following four widgets, each focusing on critical components of the application’s infrastructure:

- Load Balancer Logs: This widget displays real-time logs from the GCP Load Balancer, allowing monitoring of incoming requests, response times, and any potential errors.
- Frontend Instance Group Logs: This widget tracks logs from the frontend instance group, showing request handling, application errors, and other relevant events.
- Backend Instance Group Logs: This widget monitors the backend instance group, providing insights into API request handling, processing times, and error rates.
- Database Instance Logs: This widget captures logs from the PostgreSQL database instance, including query perfor- mance, connection issues, and other database-related metrics.

2. Uptime Checks

To ensure continuous availability of the key components of the AlbumBook application, three uptime checks were configured:

- Load Balancer Uptime Check: Monitors the availability and response time of the GCP Load Balancer to ensure it is correctly routing traffic.
- Frontend Instance Group Uptime Check: Verifies that the frontend instance group is accessible and serving the React application as expected.
- Backend Instance Group Uptime Check: Ensures that the backend instance group is reachable and handling API requests properly.

3. Alerting Policies

To proactively manage issues, several alerting policies were established:

- Load Balancer Alerting Policy: This policy triggers alerts based on the vminstance~~ check~~ pass metric, notifying the![](/readme/Aspose.Words.b4fd99a1-c490-4397-a031-082f372c21de.009.png) team if the load balancer fails to pass the uptime checks.
- Frontend Instance Group Alerting Policy: Similar to the load balancer policy, this alert monitors the frontend instance group’s uptime and triggers alerts if any issues are detected.
- Backend Instance Group Alerting Policy: This policy monitors the backend instance group’s uptime and triggers alerts for any failures in handling requests.
- Instance Autoscaling Alerting Policy: Named instance-autoscaling , this policy monitors the instance~~ group~~ size metric, with a threshold value set to 1. Alerts are triggered if the instance group size increases above this threshold, indicating potential scaling issues.
- Notification Channel: Alerts are sent to the notification channel set up with the email address.
  8 Conclusion

The AlbumBook application architecture on Google Cloud Platform (GCP) has been carefully designed to meet the key require- ments of scalability, security, high availability, and robust monitoring and logging. Below is a summary of how each of these requirements has been addressed:

1. Data Layer (Database)

- Google Cloud SQL: The application uses a relational database hosted on Google Cloud SQL, providing a fully managed and secure database service.
- Database Replication: To ensure high availability and disaster recovery, database replication has been implemented. A read replica is configured to handle read operations, reducing the load on the primary database.
- High Availability: The database configuration includes multi-zone deployment, ensuring that the database remains available even in the event of a zonal failure.

2. Scalability

- Horizontal Scaling: The architecture is designed to scale horizontally across both the frontend and backend tiers. Auto- scaling instance groups are used to dynamically adjust the number of instances based on traffic and load, ensuring that the application can handle increased demand.
- Load Balancer: GCP Load Balancer distributes traffic across multiple instances, balancing the load and allowing the architecture to scale efficiently.

3. Security

- Network Segmentation: The architecture leverages Virtual Private Clouds (VPCs) and subnets to segment the network, isolating the frontend, backend, and database components for enhanced security.
- IAM Roles: Identity and Access Management (IAM) roles, including the creation of a service account named albumstore with a predefined role of basic-editor, are used to enforce the principle of least privilege, restricting access to resources based on roles.

4. High Availability

- Multi-Zonal Deployment: Both the database and compute instances (frontend and backend) are deployed across multiple zones within the same region. This ensures that the application remains operational even if a specific zone experiences an outage.
- Fault Tolerance: The architecture is designed to tolerate failures at the instance level. In the event of an instance failure, the load balancer automatically reroutes traffic to healthy instances.

5. Monitoring and Logging

- Google Cloud Monitoring and Logging: The application utilizes Google Cloud Monitoring and Logging services to continuously monitor the health and performance of the infrastructure. Logs from the load balancer, frontend, backend, and database are collected and visualized on the albumstore-dashboard.
- Uptime Checks and Alerting Policies: Uptime checks are configured for critical components (load balancer, frontend, and backend), and alerting policies are in place to notify any issues. An additional alerting policy, instance-autoscaling, monitors the instance group size to ensure proper scaling.
  6
