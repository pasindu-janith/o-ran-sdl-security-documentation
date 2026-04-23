# Auth-Agent container

(You don't need to perform this file operations because the auth-agent image has been deployed in the DockerHub pasindujanith/auth-agent:v1)
Create Dockerfile for auth-agent:

```bash
# Use the lightweight Python image
FROM python:3.9-slim

# Install the heavy dependencies during the build process
RUN pip install --no-cache-dir \
    grpcio \
    requests \
    envoyproxy-envoy-protocolbuffers-python \
    envoyproxy-envoy-grpc-python \
    --extra-index-url https://buf.build/gen/python

# Create the working directory
WORKDIR /app

# Tell the container to just run the script when it starts
CMD ["python", "/app/agent.py"]
```


```
# Build the image (this will take 2-3 minutes, but you only do it ONCE)
sudo docker build -t pasindujanith/auth-agent:v1 .

# Log in to Docker Hub
sudo docker login

# Push the image to the internet so your cluster can download it
sudo docker push pasindujanith/auth-agent:v1
```