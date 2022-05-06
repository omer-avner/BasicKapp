# BasicKapp
a simple python application ready to be deployed on EKS

### About the python app's image

```dockerfile
FROM python:3-slim

# Override the images default base directory using docker build --build-arg SERVER_BASE_DIR=<base_dir>
ARG SERVER_BASE_DIR=/usr/local/bin
EXPOSE 8000/tcp
WORKDIR $SERVER_BASE_DIR

# Add the argument --directory <dir_path> to change the base directory at runtime
ENTRYPOINT ["python3", "-m", "http.server", "--bind", "0.0.0.0", "8000"]
```
The application's main service is a simple python http server serving files from a local directory tree. Since the image's main proccess will run python were using a slim version of the python3 image (slim version will reduce security risks and storage space exhaustion). The files displayed via the http server are sourced in the main proccess's workdir which  means that the image's default workdir will also be the default base directory of the http server. For better abstraction i've created a Dockerfile argument for the server's default workdir, this way we could easily override the default workdir at the image build time (see the said Dockerfile for reference)

try building the image as u will `docker build -t python-app:v0.0.2 --build-arg SERVER_BASE_DIR=/ image/`
try running `docker run -dit -p 8000:8000 --name python python-app:v0.0.2 --directory /usr/local/`


### About the application's chart

### About the CI/CD workflow
