# BasicKapp
a simple python application ready to be deployed on EKS

## About the python app's image

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

notice that the entrypoint of the image determines that the http server will *always bind to all interfaces at port 8000* (thats why `EXPOSE 8000/tcp` is mentioned explicitly in the Dockerfile), but since were using an `ENTRYPOINT` instead of a simple `CMD` command were able to add runtime arguments such as `--directory <dir_path>` in order to change the default server base directory (SERVER_BASE_DIR).


## About the application's chart

## About the CI/CD workflow
This repo containes two different workflows:
- build-and-push-image
- deploy-chart

### build-and-push-image
The build-and-push workflow is a simple ci procedure that's triggered in any push to main branch that contains changes to the `image` directory. The workflow is composed of the following steps:
1. Checking out the repo- this way we can use the repo's directory structure as part of our build proccess.
```yml
      - name: Check out the repo
        uses: actions/checkout@v2
```
2. Logging into the registry- we need to access and push the new image to the repo using the pushing account's PAT and username.
```yml
      - name: Log in to GitHub Docker Registry
        uses: docker/login-action@f054a8b539a109f9f41c372932f1ae047eff08c9
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```
3. building the image and pushing it- were using a built-in action that uses the `image/Dockerfile` and pushed a new tag based on the current commit's sha.
```yml
      - name: Build and push Docker image
        uses: docker/build-push-action@ad44023a93711e3deb337508980b4b5e9bcdc5dc
        with:
          file: image/Dockerfile
          context: .
          push: true
          tags: |
            ghcr.io/omer-avner/pyhton-app:${{ github.sha }}
```

### deploy-chart
The deploy-chart workflow is a simple cd procedure that either installs or upgrades the application's Helm Chart over an EKS cluster. The workflow accures at any push to main branch that containes changes to the `chart/python-application` directory and its composed of the following steps:
1. Not setting up helm- the assignment was to setup helm but i found a more simple action that runs an image that already contains helm. The helm image is rebuilt at any action trigger, might take longer but its more secure.
2. Checking out the repo- for obvious reasons we need the files in this repo.
3. Configuring AWS creds- there are a few MUST HAVE enviorment vars that we need to create for the aws CLI. Helm 
 to create the relevant environment secrets
