## Argocd sync & wait

Proceso para realizar tarea de sincronización y espera de estado de aplicaciones en argocd, vinculando login y permisos con políticas RBAC de Openshift.

En este procedimiento, se creará una imagen de contenedores que contiene cliente oc y argocd cli para realizar las acciones.

Se agrega una Task de Tekton, para realizar pruebas de integración con pipelines. Pero también se dejan ejemplos de utilización con docker directo, para integrar con otras herramientas de CI/CD.

### Creación docker image local

```
docker build -t oc-argocd-cli -f docker/Dockerfile .
```

### Creación docker image Openshift

```
oc project openshift
```
```
oc new-build oc-argocd-cli --binary --strategy docker
```
```
oc start-build argocd-oc --from-dir docker
```

### Creación de SA OCP
En este procedimiento, se creará una service account en Openshift y además un role custom para que dicha SA tenga acceso y permisos solo a los recursos que necesita para realizar las tareas de sincronización.

```
oc project openshift-gitops
```
```
oc create sa argo-sync
```
```
oc apply -f templates/role.yaml
```
```
oc adm policy add-role-to-user argo-sync system:serviceaccount:openshift-gitops:argo-sync
``` 

Posterior a la creación de la SA, se obtiene el token de acceso

```
export SA_TOKEN=$(oc create token argo-sync -n openshift-gitops)
```


### Utilización con Tekton
Para la utilización con tekton, se crea una Task, que tiene como parámetro el nombre de la aplicación a sincronizar, la url del servidor de argocd, la url de la API del cluster Openshift y el token de la SA a utilizar. 

Para fines prácticos, la url del cluster openshift y el token de acceso, se vinculan directamente a la Task desde el template, pero pueden ser tratados como parámetros adicionales.

```
oc process -f templates/sync-wait-custom-template.yaml -p URL_API_OCP=<url_api_openshift_cluster> -p SA_TOKEN=$SA_TOKEN -o yaml | oc apply -f -
```

### Utilización con docker

Para realizar pruebas locales o integraciones con otras herramientas de CI/CD, se puede utilizar la imagen docker creada localmente.

Para validar el comportamiento, podemos iniciar el contenedor con  los siguientes parámetros:

```
docker run -it oc-argocd-cli /bin/bash
```

Una vez dentro del contenedor, procedemos a ejecutar los siguientes comandos:

```
export ARGOCD_REPO_SERVER_NAME=openshift-gitops-repo-server
export SA_TOKEN=<saToken>
export URL_API_OCP=<urlApiOpenshift>
export URL_ARGOCD=<urlArgoServer>
export APP_NAME=<appName>
```
```
oc login --token=${SA_TOKEN} --server=${URL_API_OCP}
```
```
kubectl config set-context --current --namespace=openshift-gitops
```
```
argocd login ${URL_ARGOCD} --core
```
```
argocd app sync ${APP_NAME} 
```
```
argocd app wait ${APP_NAME} --health
```

NOTA: En caso de no contar con certificados válidos en la Api de OCP, agregar el flag '--insecure-skip-tls-verify' al comando de login y el flag --insecure a los comandos de argocd cli.
         