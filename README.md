# Installing Magma Orchestrator for Kubernetes cluster ( MAGMA_version 1.6.1 )

## Install Helm

- To install and activate Magma, you need your Helm commands written for Helm 3.x 

- You can install Helm with the following command:

    >  $ sudo su 

    > ` # curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null `

    > ` $ sudo apt-get install apt-transport-https `

    > ` $ echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list `

    > ` $ sudo apt-get update`

    > ` $ sudo apt-get install helm `

#### Initialize a Helm Chart Repository

- Once you have Helm ready, you can add a chart repository.

    > ` $ helm repo add bitnami https://charts.bitnami.com/bitnami `

- Once this is installed, you will be able to list the charts you can install:

    > ` $ helm search repo bitnami`

## Install docker and pulish the image

### Install docker

> `$ sudo apt install docker.io`

### Create the account in docker website

### Publish the image
> `$ docker login `

- If you see this error:
    
    ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/19.png)
- This error can be fixed by the following the command below:
    - > ` sudo chmod 666 /var/run/docker.sock
  
**If you docker registry is private and self hosted you should do the following :**
> `docker login <REGISTRY_HOST>:<REGISTRY_PORT>`

> `docker tag <IMAGE_ID> <REGISTRY_HOST>:<REGISTRY_PORT>/<APPNAME>:<APPVERSION>`
    
> `docker push <REGISTRY_HOST>:<REGISTRY_PORT>/<APPNAME>:<APPVERSION> `

**Example :**

> docker login repo.company.com:3456
> docker tag 19fcc4aa71ba repo.company.com:3456/myapp:0.1
> docker push repo.company.com:3456/myapp:0.1
## Set up   ` MAGMA_ROOT` environment ` 
-  `MAGMA_ROOT` environment variable is set to the local directory where you cloned the Magma repository:
   
   > ` $ mkdir ~/magma `
    
   > ` $ export MAGMA_ROOT=~/magma `

## Install **Postgresql** with Helm

- > ` $ helm upgrade -i \
    postgresql \
    --create-namespace \
    --namespace orc8r \
    --set global.postgresql.auth.postgresPassword=postgres,global.postgresql.auth.database=orc8r,fullnameOverride=postgresql \
    bitnami/postgresql`

- You should see the result like this:
  ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/1.png)

- Next check the pods with command:

    > ` $ kubectl get pods -A `

    - The result should look like this:

        ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/2.png)
`
    
    - Describe the pod the error with command:

        > ` $ kubectl describe pod postgresql-0 -n orc8r `
    
    - The sceenshot will expose the error:
        ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/3.png)
    
    - The reason for this error because your Statefulset pod postgresqpl-0 does not have Persistent Volume Claim and Persistent Volume.
     
    - Delete the pod first before trying to fix this error:
        > ` $ kubectl get statefulset -A `
    
        > ` $ kubectl delete statefulset postgresql -n orc8r `

        ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/8.png)

    **You can fix this error following the steps below:**

    ### Step 1: Create Persistent Volume 
    
    - Create a Persistent Volume with the following command:

        > `$ vim postgres-pv.yaml`

    - Edit the file and add the following lines:

        ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/4.png)

    - Build Persistent Volume with the command below:
        > ` $ kubectl apply -f postgres-pv.yaml `

    - Check the Persistent Volume with the command below:
        > ` $ kubectl get pv `

        ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/5.png)
    
    ### Step 2: Create Persistent Volume Claim bound with the Persistent Volume

    - Create a Persistent Volume Claim with the following command:

        > `$ vim postgres-pvc.yaml`

    - Edit the file and add the following lines:
    
        ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/6.png)

    - Build Persistent Volume Claim with the command below:
        > ` $ kubectl apply -f postgres-pvc.yaml `

    - Check the Persistent Volume Claim with the command below:
        > ` $ kubectl get pvc `

        ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/7.png)
    
    ### Step 3: Install Helm Chart (Postgresql-database)

    - Install the Helm Chart with the following command:

    > `$ helm install --create-namespace --namespace orc8r postgresql --set auth.postgresPassword=postgres,auth.database=magma bitnami/postgresql --set persistence.existingClaim=postgresql-pvc --set primary.persistence.enabled=false`

    - The result should look like this:
    ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/10.png)

    **Warning:**
    - If you want to delete pods created by helm, you cann't delete the pods by using command ` kubectl delete pod <pod name> -n <namespace>`. Instead you can list the name of `bitnami` charts and delete them by command `helm uninstall`
      - >`$ helm list -A `
    
      - ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/9.png)
      
      - With the charts has namespace " default ":
         > ` $ helm uninstall <name> `
    
      - With the charts has namespace :   
        > ` $ helm uninstall <name> -n <namespace> `
    
    ### Step 4: Check connection with database
    - We need to extract the password of the instance and save it as an environmental variable. That password will be used to create the connection.
    
        ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/11.png)
    
    - Run the following commands:
    - > ` $ export POSTGRES_PASSWORD=$(kubectl get secret --namespace orc8r postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)`
    
    - > ` $ kubectl run postgresql-client --rm --tty -i --restart='Never' --namespace orc8r --image docker.io/bitnami/postgresql:14.4.0-debian-11-r0 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host postgresql -U postgres -d magma -p 5432 `

    - Open the second terminal and type this line:
    - > ` kubectl get pods -A `
    - You should see the result like this:

        - On the first terminal:
            ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/12.png)
        - On the second terminal:
            ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/13.png)
    
    - At this point, we have connected successfully with the database, we can verify our installation by checking the server version.
            ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/14.png) 
    
## Installing Elastic Search with Helm

### Step 1: Add elastic repository:
> `$ helm repo add elastic https://helm.elastic.co`
### Step 2: Create Persistent Volume:

- Edit the file and add the following lines:
- 
    ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/15.png)

- Build the Persistent Volume with the following command:
- 
    > ` $ kubectl apply -f elasticsearch-pv.yaml `

### Step 3: Create Persistent Volume Claim:

- Edit the file and add the following lines:
- 
  ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/16.png)

- Build the Persistent Volume Claim with the following command:
    >  ` $ kubectl apply -f elasticsearch-pvc.yaml `

### Step 4 : Install Elastic Search with Helm:

- Build the Elastic Search with the following command:
    
    >  ` $  helm install elasticsearch --set master.replicas=2,coordinating.service.type=LoadBalancer bitnami/elasticsearch --namespace orc8r --set master.persistence.existingClaim=elastic-pv-claim --set data.persistence.existingClaim=elastic-pv-claim --set volumePermissions.enabled=true --set master.persistence.enabled=false --set data.persistence.enabled=false
`
- The result should look like this:
    
    > ` $ helm list -A `

    ![](D:/Magma/17.png)

    > ` $ kubectl get pods -A `

    ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/18.png)

## Installing Orchestrator of Magma into K8S cluster
First click into this link: https://github.com/magma/magma/tree/v1.6.1 (you can use whatever version you want)

To install components of Orchestrator, you should generate the MAGMA environment:
- > ` mkid magma`
    
- > ` export MAGMA_ROOT=~/magma` 
### Build and publish container images
    
> `$ docker login`
>Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
    
>Username: REGISTRY_USERNAME
>Password:
>Login Succeeded
We provide scripts to build and publish images. The publish script is provided as a starting point, as individual needs may vary.

First define some necessary variables:
> `$ export PUBLISH=${MAGMA_ROOT}/orc8r/tools/docker/publish.sh  # or add to path`
    
> `$ export REGISTRY=registry.hub.docker.com/REGISTRY  # or desired registry`
    
>  `$ export MAGMA_TAG=1.6.0-master  # or desired tag`
    
### Build and publish Orchestrator images (you can skip this step if you want to use the official container images at artifactory.magmacore.org )
    
> `$ cd ${MAGMA_ROOT}/orc8r/cloud/docker`
    
> `$ ./build.py --all`

See this error? ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/28.png)

- This means you lack of some file you need to install with:
> `$ git clone --depth 1 https://github.com/magma/magma.git dp`
    
> `$ cd dp`
    
> `$    git filter-branch --prune-empty --subdirectory-filter dp HEAD `

### Generate secrets:

First you'll need to create a few certs and the kubernetes secrets that orchestrator uses.

Generate the NMS certificate:
> `$ mkdir ~/magma/.cache`
    
> `$ mkdir  ~/magma/.cache/test_certs`
    
> `  $ cd ${MAGMA_ROOT}/.cache/test_certs`
    
> ` $ openssl req -nodes -new -x509 -batch -keyout nms_nginx.key -out nms_nginx.pem -subj "/CN=*.localhost" `

![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/20.png)

Move the certs to the charts directory:
> `$  cd ${MAGMA_ROOT}/orc8r/cloud/helm/orc8r`
    
> `$ mkdir -p charts/secrets/.secrets/certs`
    
> ` $ cp -r ../../../../.cache/test_certs/* charts/secrets/.secrets/certs/.`

Run the commands below:
> `$ helm template orc8r ${MAGMA_ROOT}/orc8r/cloud/helm/orc8r/charts/secrets \
    --namespace orc8r \
    --set-string secret.certs.enabled=true \
    --set-file secret.certs.files."rootCA\.pem"=charts/secrets/.secrets/certs/rootCA.pem \
    --set-file secret.certs.files."bootstrapper\.key"=charts/secrets/.secrets/certs/bootstrapper.key \
    --set-file secret.certs.files."controller\.crt"=charts/secrets/.secrets/certs/controller.crt \
    --set-file secret.certs.files."controller\.key"=charts/secrets/.secrets/certs/controller.key \
    --set-file secret.certs.files."admin_operator\.pem"=charts/secrets/.secrets/certs/admin_operator.pem \
    --set-file secret.certs.files."admin_operator\.key\.pem"=charts/secrets/.secrets/certs/admin_operator.key.pem \
    --set-file secret.certs.files."certifier\.pem"=charts/secrets/.secrets/certs/certifier.pem \
    --set-file secret.certs.files."certifier\.key"=charts/secrets/.secrets/certs/certifier.key \
    --set-file secret.certs.files."nms_nginx\.pem"=charts/secrets/.secrets/certs/nms_nginx.pem \
    --set-file secret.certs.files."nms_nginx\.key\.pem"=charts/secrets/.secrets/certs/nms_nginx.key \
    --set=docker.registry=$DOCKER_REGISTRY \
    --set=docker.username=$DOCKER_USERNAME \
    --set=docker.password=$DOCKER_PASSWORD |
    kubectl apply -f - `

You may see the following error:
![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/22.png)

To fix the eror, add the following line:

> `$ cp ~/magma/orc8r/cloud/deploy/scripts/create_application_certs.sh ${MAGMA_ROOT}/.cache/test_certs`
    
> ` $ cd ${MAGMA_ROOT}/.cache/test_certs `
    
> ` $ ./create_application_certs.sh "/CN=*.localhost" `

>   `  $ cp ~/magma/orc8r/cloud/deploy/scripts/self_sign_certs.sh ${MAGMA_ROOT}/.cache/test_certs`

>` $ sudo su`
    
> ` # ./self_sign_certs.sh "/CN=*.localhost"`
    
> `$ sudo chown user -R ~/magma/.cache/test_certs`
    
> ` $ cd ${MAGMA_ROOT}/orc8r/cloud/helm/orc8r`
    
> ` $ cp -r ../../../../.cache/test_certs/* charts/secrets/.secrets/certs/.`
 
After finishing the copy process, the `~/magma/orc8r/cloud/helm/orc8r/charts/secrets/.secrets/certs` folder should look like this:
![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/23.png)

Run the command again:

> ` $ cd ${MAGMA_ROOT}/orc8r/cloud/helm/orc8r`
    
> ` $ helm template orc8r ${MAGMA_ROOT}/orc8r/cloud/helm/orc8r/charts/secrets \
    --namespace orc8r \
    --set-string secret.certs.enabled=true \
    --set-file secret.certs.files."rootCA\.pem"=charts/secrets/.secrets/certs/rootCA.pem \
    --set-file secret.certs.files."bootstrapper\.key"=charts/secrets/.secrets/certs/bootstrapper.key \
    --set-file secret.certs.files."controller\.crt"=charts/secrets/.secrets/certs/controller.crt \
    --set-file secret.certs.files."controller\.key"=charts/secrets/.secrets/certs/controller.key \
    --set-file secret.certs.files."admin_operator\.pem"=charts/secrets/.secrets/certs/admin_operator.pem \
    --set-file secret.certs.files."admin_operator\.key\.pem"=charts/secrets/.secrets/certs/admin_operator.key.pem \
    --set-file secret.certs.files."certifier\.pem"=charts/secrets/.secrets/certs/certifier.pem \
    --set-file secret.certs.files."certifier\.key"=charts/secrets/.secrets/certs/certifier.key \
    --set-file secret.certs.files."nms_nginx\.pem"=charts/secrets/.secrets/certs/nms_nginx.pem \
    --set-file secret.certs.files."nms_nginx\.key\.pem"=charts/secrets/.secrets/certs/nms_nginx.key \
    --set=docker.registry=$DOCKER_REGISTRY \
    --set=docker.username=$DOCKER_USERNAME \
    --set=docker.password=$DOCKER_PASSWORD |
    kubectl apply -f - `

![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/24.png)
### Create Values File

> `$ cd ${MAGMA_ROOT}/orc8r/cloud/helm/orc8r`
    
> `$ helm install orc8r --namespace orc8r .  --values=${MAGMA_ROOT}/orc8r/cloud/helm/orc8r/examples/minikube.values.yaml`

You may see this error:
![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/27.png)

Run this command to find the location of error file:
> `$ cd ${MAGMA_ROOT}/orc8r/cloud/helm`
    
> `$ grep -rnw . -e "v1beta1"`

The error file is:
![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/26.png)

Edit the file like this:
![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/25.png)

Edit the file   **minikube_values.yml** in directory `${MAGMA_ROOT}/orc8r/cloud/helm/orc8r/examples` like this:

![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/29.png)
![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/30.png)
![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/31.png)

Run the commands below:

> ` $ cd ${MAGMA_ROOT}/orc8r/cloud/helm/orc8r`
    
> ` $ helm install orc8r --namespace orc8r .  --values=${MAGMA_ROOT}/orc8r/cloud/helm/orc8r/examples/minikube_values.yml`

- After finish its installation, you may see this errors:
    #### The first bug:

    ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/32.png)

    - Try to figure out the problems by using these command below:

        > ` $ kubectl describe pods orc8r-nginx-766b7d7fc4-nsrqx -n orc8r`
    
        > ` $  kubectl logs orc8r-nginx-766b7d7fc4-nsrqx -n orc8r `

        ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/33.png)
    - Ok, we figure out the error, now find the file that contains **kube-dns.kube-system.svc.cluster.local**
        > `$ sudo vim ~/magma/orc8r/cloud/helm/orc8r/values.yaml`
    
    - Change the following line:

        ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/34.png)
        
        > `  $ kubectl get svc `

        ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/37.png)
        ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/35.png)
        ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/36.png)
    
    #### The second bug (This problem only happens if you use one database for both NMS and cluster like my case, i only use Postgresql for boths):

    - Try to figure out the problems by using these command below:

        > ` $ kubectl describe pods nms-magmalte-667f8d8d8b-w56dc -n orc8r`
    
        > ` $ kubectl logs nms-magmalte-667f8d8d8b-w56dc -n orc8r`

        ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/38.png)
    
    - Ok so the problems lie on file `~/magma/orc8r/cloud/helm/orc8r/chart/nms/values.yaml`:
      ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/39.png) 
  

Run the commands below again and you can see these problems have been solved:

> ` $ cd ${MAGMA_ROOT}/orc8r/cloud/helm/orc8r`

> ` $ helm install orc8r --namespace orc8r .  --values=${MAGMA_ROOT}/orc8r/cloud/helm/orc8r/examples/minikube_values.yml`

**The result should look like this:**
![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/40.png)

### Access Orc8r
Create an Orc8r admin user
> ` $ kubectl --namespace orc8r port-forward svc/orc8r-nginx-proxy 7443:8443 7444:8444 9443:443`

Now ensure the API and your certs are working:
 
**In tab 1 - You should hang it**
 > `$ kubectl --namespace orc8r port-forward svc/orc8r-nginx-proxy 7443:8443 7444:8444 9443:443`

**In tab 2**
> ` $ export CERTS_DIR=${MAGMA_ROOT}/.cache/test_certs  # mirrored from above`

> `$ curl \
  --insecure \
  --cert ${CERTS_DIR}/admin_operator.pem \
  --key ${CERTS_DIR}/admin_operator.key.pem \
  https://localhost:9443 `

### Access NMS
> ` $ kubectl --namespace orc8r port-forward svc/nginx-proxy 8081:443`

Log in to NMS at https://magma-test.localhost:8081 using credentials: `admin@magma.test/password1234`
    
## Warning Access NMS

If you're just like me, perform Magma NMS Web GUI in another computer ( in my case it is Laptop Window 10 **IP_address: 10.10.y.y**) and the Master node which had been successful to deploy Orchestrator **IP_address=192.168.x.x** , You should follow this step instead:

1. Open the notepad with administration ( Window).
2. Edit to folder C:\Windows\drivers\etc\hosts
look like this:
    ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/41.png)
3. Add the following commands in Orchestrator:

    > `$ kubectl exec -it --namespace orc8r deploy/orc8r-orchestrator --   /var/opt/magma/bin/accessc   add-existing -admin -cert /var/opt/magma/certs/admin_operator.pem   admin_operator`

    > `$ kubectl --namespace orc8r exec -it deploy/nms-magmalte -- yarn migrate`

    > ` $ kubectl --namespace orc8r exec -it deploy/nms-magmalte -- yarn setAdminPassword magma-test admin@magma.test password1234`

    > ` $ kubectl --namespace orc8r exec -it deploy/nms-magmalte -- yarn setAdminPassword master admin@magma.test password1234`

- Open two tab in Orc8r terminal:
    - Tab 1:

        > `$ kubectl --namespace orc8r port-forward svc/orc8r-nginx-proxy 7                                                                              443:8443 7444:8444 9443:443`
    
    - Tab 2: 
        > `$ kubectl --namespace orc8r --address 0.0.0.0 port-forward svc/nginx-proxy 8081:443`
- Open Web browser in your computer - laptop:

    https://magma-test.magma.test:8081
- Log in with:  `admin@magma.test/password1234`
- The result should look like this:
    ![](https://github.com/TranAnh-Tuan/Installing-Magma-Orchestrator-with-Kubernetes/blob/main/Images/43.png)


    ## Author
    ---
    #### Name: Phạm Ngọc Ánh
    #### Email: phamngocanh2711@gmail.com
    ---
    #### Name: Trần Anh Tuấn
    #### Email: tuan-hs11115@ngoisao.edu.vn
  
