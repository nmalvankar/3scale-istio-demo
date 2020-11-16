# 3scale istio demo

## Install OCS on OCP 4.x
```cd ocs-install

git clone https://github.com/openshift/openshift-cns-testdrive.git -b ocp4-dev content

oc get nodes -l node-role.kubernetes.io/worker -l '!node-role.kubernetes.io/infra','!node-role.kubernetes.io/master'

oc get machinesets -n openshift-machine-api | grep -v infra

CLUSTERID=$(oc get machineset -n openshift-machine-api -o jsonpath='{.items[0].metadata.labels.machine\.openshift\.io/cluster-api-cluster}')
echo $CLUSTERID

Check the availability zone and region
          placement:
            availabilityZone: us-east-1a
            region: us-east-1

Check the value of ami for other worker nodes and use the same for OCS worker nodes
          ami:
            id: ami-08f17f5bd2210c9fa

cat content/support/ocslab_cluster-workerocs.yaml | sed "s/CLUSTERID/$CLUSTERID/g" | oc apply -f -

oc get machines -n openshift-machine-api | egrep 'NAME|workerocs'

watch "oc get machinesets -n openshift-machine-api | egrep 'NAME|workerocs'"

oc get nodes -l node-role.kubernetes.io/worker -l '!node-role.kubernetes.io/infra','!node-role.kubernetes.io/master'

oc create namespace openshift-storage

oc label namespace openshift-storage "openshift.io/cluster-monitoring=true"

oc get nodes --show-labels | grep storage-node |cut -d' ' -f1

watch oc -n openshift-storage get csv
```

Set ocs-storagecluster-cephfs as the default storage class for the OCP cluster

## Install 3scale API Management

oc apply -f 3scale-install/apimanager.yaml -n 3scale

## OSSM Installation

1. Install ElasticSearch Operator
2. Install Jaeger Operator
3. Install Kiali Operator
4. Install OSSM Operator

Install the service mesh control plane with 3scale adapter

```oc apply -f ossm/install/full-install-3scale.yaml -n istio-system
```

## Install bookinfo example application
```cd bookinfo

oc new-project bookinfo

oc apply -f member-roll.yaml

oc apply -f bookinfo.yaml -n bookinfo

oc apply -f bookinfo-gateway.yaml -n bookinfo

oc apply -f destination-rule-all.yaml -n bookinfo
```

## Install 3scale adapter

Pay attention to the below configurations:

In the global section, the value of disablePolicyChecks is set to false:

    global:
      disablePolicyChecks: false
      
Threescale component configurations in spec section:

    threeScale:
      enabled: true
      image: 3scale-istio-adapter-rhel8
      tag: 1.0.0
      PARAM_THREESCALE_ALLOW_INSECURE_CONN: false
      PARAM_THREESCALE_CACHE_ENTRIES_MAX: 1000
      PARAM_THREESCALE_CACHE_REFRESH_RETRIES: 1
      PARAM_THREESCALE_CACHE_REFRESH_SECONDS: 180
      PARAM_THREESCALE_CACHE_TTL_SECONDS: 300
      PARAM_THREESCALE_CLIENT_TIMEOUT_SECONDS: 10
      PARAM_THREESCALE_GRPC_CONN_MAX_SECONDS: 60
      PARAM_THREESCALE_LISTEN_ADDR: 3333
      PARAM_THREESCALE_LOG_GRPC: false
      PARAM_THREESCALE_LOG_JSON: true
      PARAM_THREESCALE_LOG_LEVEL: debug
      PARAM_THREESCALE_METRICS_PORT: 8080
      PARAM_THREESCALE_REPORT_METRICS: true

### Configure thee 3scale adapter

```SM_CP_NS=istio-system
BOOKINFO_NS=bookinfo
API_MANAGER_NS=3scale
API_ADMIN_ACCESS_TOKEN=XXXXXXXXXX
SYSTEM_PROVIDER_URL=https://3scale-admin.apps.cluster-8244.8244.example.opentlc.com
HANDLER_NAME=threescale
SERVICE_ID=3

echo $SM_CP_NS
echo $BOOKINFO_NS
echo $API_MANAGER_NS
echo $API_ADMIN_ACCESS_TOKEN
echo $SYSTEM_PROVIDER_URL
echo $HANDLER_NAME
echo $SERVICE_ID

cd 3scale-istio
oc exec -n ${SM_CP_NS} $(oc get po -n ${SM_CP_NS} -o jsonpath='{.items[?(@.metadata.labels.app=="3scale-istio-adapter")].metadata.name}') -it -- ./3scale-config-gen --url ${SYSTEM_PROVIDER_URL} --name ${HANDLER_NAME} --token ${API_ADMIN_ACCESS_TOKEN} -n ${SM_CP_NS} > threescale-adapter-config.yaml

oc apply -f threescale-adapter-config.yaml
or 
oc create -f threescale-adapter-config.yaml -n $SM_CP_NS

Check
oc get handler threescale -n $SM_CP_NS -o yaml


Patch the productpage deployment to use 3scale adapter
patch="$(oc get deployment -n "${BOOKINFO_NS}" productpage-v1 --template='{"spec":{"template":{"metadata":{"labels":{ {{ range $k,$v := .spec.template.metadata.labels }}"{{ $k }}":"{{ $v }}",{{ end }}"service-mesh.3scale.net/service-id":"'"${SERVICE_ID}"'","service-mesh.3scale.net/credentials":"'"${HANDLER_NAME}"'"}}}}}' )"

echo $patch

oc patch -n "${BOOKINFO_NS}"  deployment productpage-v1 --patch ''"${patch}"''
```

Update the 3scale configuration to use istio and update the product

Configuration page > update configuration

Create new application for productpage

```export USER_KEY=6b2774a23842ca6b7aa66f666222a3a9

curl -v -k `echo "${GATEWAY_URL}/productpage"`

curl -v -k `echo "$GATEWAY_URL/productpage?user_key=$USER_KEY"`
```

## References: 
OCS Installation:
http://ocp-ocs-admins-labguides.6923.rh-us-east-1.openshiftapps.com/workshop/ocs4

3scale-istio adapter: 
https://www.opentlc.com/labs/3scale_advanced_implementation/03_2_API_Mgmt_Adapter_Lab.html#_3scale_api_service_configuration


## Force delete OCS
Force delete 
curl -k -H "Content-Type: application/json" -H "Authorization: Bearer $(oc whoami -t)" -X PUT --data-binary @openshift-storage.json  https://$(oc whoami --show-server)/api/v1/namespaces/openshift-storage/finalize

