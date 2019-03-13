# ACR Task Web App

The sample uses a relay as the front end.
For this we need the relay configuration passed in as values into the task.

## Build and push the app to your registry

```bash
az configure --defaults acr=myregistry
az acr build -t relay:app ./app
```

For this we first create a relay namespace and hybrid connection as follow 

```bash
az configure --defaults group=gateway 
az relay namespace create -n taskgateway

## This doesn't currently work - https://github.com/Azure/azure-cli/issues/8775
# Need to create this in the portal to disable 
# client authorization
az relay hyco create -n app --namespace taskgateway --require-client-authorization false
```

```bash 
# Create the listener key 
az relay hyco authorization-rule create -n listener \
    --hybrid-connection-name app --rights Listen \
    --namespace-name taskgateway \
    -g gateway
```

## Populate your environment variables

```bash
az configure --defaults group=gateway
export RELAY_NAMESPACE_NAME=taskgateway
export RELAY_HYBRID_CONNECTION_NAME=app
export RELAY_SAS_KEY_NAME=listener

export RELAY_NAMESPACE=$(az relay namespace show -n $RELAY_NAMESPACE_NAME \
 --query serviceBusEndpoint -o tsv | awk -F[/:] '{print $4}')
export RELAY_SAS_KEY_VALUE=$(az relay hyco authorization-rule keys list \
-n $RELAY_SAS_KEY_NAME \
--hybrid-connection-name $RELAY_HYBRID_CONNECTION_NAME  \
--namespace-name taskgateway -g gateway \
--query primaryKey -o tsv)
```

## Generate a values file with the required variables

```bash
# generate a .values by replacing the
# environment variables in the ./task/values.yaml template
envsubst < ./task/values.yaml > ./task/.values
```

## Run the task

This uses [`task\acb.yaml`](task\acb.yaml) and passes in the `.values` file generated above, into an Quick task run request.

```bash
az configure --defaults acr=myregistry
az acr run --values .values ./task
```