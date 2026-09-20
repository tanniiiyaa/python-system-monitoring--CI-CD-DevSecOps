# AWS Deployment

## Application container

The application listens on port 5000 inside the container.

Example image:

```text
cheesepopcorn/python-system-monitoring:<tag>
```

Run the container:

```bash
docker run -d \
  --name python-system-monitoring \
  -p 5000:5000 \
  --restart unless-stopped \
  cheesepopcorn/python-system-monitoring:<tag>
```

Verify:

```bash
docker ps
curl http://localhost:5000/
```

## Application Load Balancer

- Scheme: internet-facing
- Listener: HTTP :80
- Target group protocol: HTTP
- Target port: 5000
- Health check path: /
- Targets: both application EC2 instances

The application security group should allow port 5000 from the ALB security group rather than exposing the application port to the entire internet.

## Deployment model

The current validated project used a published Docker tag and manually pulled/run that image on both EC2 application servers.

A future Jenkins CD stage should:

1. Pull the new image tag.
2. Replace the running container on App Server 1.
3. Verify the application locally.
4. Update App Server 2.
5. Let the ALB continue serving healthy targets.
