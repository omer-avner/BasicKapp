# BasicKapp
A simple python application ready to be deployed on EKS.

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

- You can go ahead and try  building the image as u will, with a default base directory of your choise, here is an example for overriding the default `/usr/local/bin` base directory with the `/` as base directory: 
`docker build -t python-app:v0.0.2 --build-arg SERVER_BASE_DIR=/ image/`
- Try running the image localy and setting the directory at runtime: `docker run -dit -p 8000:8000 --name python ghcr.io/omer-avner/pyhton-app:14e4e52a839d8c116309971e387f2e7f72452374 --directory /root`

Notice that the entrypoint of the image determines that the http server will **always bind to all interfaces at port 8000** (thats why `EXPOSE 8000/tcp` is mentioned explicitly in the Dockerfile). Since were using an `ENTRYPOINT` instead of a simple `CMD` command were able to add runtime arguments such as `--directory <dir_path>` in order to change the default server base directory (SERVER_BASE_DIR).

## About the application's chart
There is nothing super fnacy with the helm chart i've created for the application, its composed of the following templates:
1. `_helpers.tpl`- contains templates used throughout the chart's manifests such as the applications lables and the index.html data.
2. `configMap`- containg the file that will be served by the python application.
3. `deployment`- 1 replica of python fantasy! using volume mounts it mounts the configMap's index html file to the directory served by the http server.
4. `service`- i chose to use service of type LB to expose the service via a public address (nodePort is not secured, clusterIP is internal only etc...). Gladly our EKS cluster runs on the AWS cloud so were using the ELB service to get a public hostname for our service. Sadly (and due to lack of time) the hostname of our service is auto generated by the ELB service, ideally i would have deployed an AWS ALB controller and created an Ingress object with custome hostname (tbh i might couldnt have done that without proper AWS premissions, the controller needs to authenticate itself with other AWS services, i could have used nginx ingress controller as well).

The chart contains the following **mandatory** values in its values.yaml:
| Key         | Value       |
| ----------- | ----------- |
| namespace   | Should be a namespace that already exists within the cluster |
| pythonApp.name | Used throughout every manifests name as so: <pythonApp.name>-<resourceType> |
| pythonApp.image | The image's registry and repo path |
| pythonApp.tag | The desired tag of the image |
| pythonApp.baseDir | The directory path that will contain the configMap generated files and will be served by the http server |


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
3. Building the image and pushing it- were using a built-in action that uses the `image/Dockerfile` and pushed a new tag based on the current commit's sha.
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
3. Deploying the Helm Chart- since helm uses the kubeconfig data in order to deploy the chart, i've created an enviorment secret named `KUBECONFIG` that contains a base64 of both the data required to authenticate to the eks cluster and the MUST HAVE enviorment vars that we need for the aws CLI auth plugin (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY).
```yml
# kubeconfig snippet
      exec:
         apiVersion: client.authentication.k8s.io/v1alpha1
         command: aws
         args:
           - --region
           - <AWS_REGION>
           - eks
           - get-token
           - --cluster-name
           - <CLUSTER_NAME>
         env:
           - name: "AWS_ACCESS_KEY_ID"
             value: <AWS_ACCESS_KEY_ID>
           - name: "AWS_SECRET_ACCESS_KEY"
             value: <AWS_SECRET_ACCESS_KEY>
           - name: "AWS_DEFAULT_REGION"
             value: <AWS_REGION>
```
 Using this secret we can run a simple helm uppgrade command that either deploys the chart from scratch or upgrades an existing installation.
```yml
      - name: helm deploy
        uses: koslib/helm-eks-action@master
        env:
          KUBE_CONFIG_DATA: ${{ secrets.KUBECONFIG }}
        with:
          command: helm upgrade python-application --install --wait charts/python-application
```

## Testing this repo
Lets begin by testing this repo's different workflows:
- In order to trigger the build-and-push-image workflow all we need to do is make changes in the `/image` dir and push them to the main branch. We can try changing the `$SERVER_BASE_DIR` argument in the Dockerfile and a new image will be built and pushed to ghcr.io/omer-avner/pyhton-app repo with the current push's commit sha.

- In order to trigger the deploy-chart workflow all we need to do is make changes in the `charts/python-application` dir and push them to main branch. We can try changing the **pythonApp.indexHtml** template in `_helpers.tpl`, this should trigger a helm upgrade, and after restarting the python-application's pod we could see the new html data in our browser.

Now we can try and reach the application trough our own browser:
1. Run `kubectl get svc -n omer` to get the LB service's hostname. You should see an `EXTERNAL-IP` similer to this (this is the auto generated hostname): `a520894768d35449ba6ef2f15ecbd3c6-219064021.us-east-1.elb.amazonaws.com`
2. Search the said hostname in your browser. Since the service is mapped to port 80 you dont even need to specify a certain port. Notice that if u just created a new installation from scratch it might take a few seconeds for the new dns record to take place, in that cenario you might get dns lookup errors from the browser, dont worry about it and keep waiting, relevant errors might be connection timed out and etc. ![The app works](/images/working_app.png)
