# Project-1
AKS- Azure Key Vault Integration 

Step 1:  az Login 
Step 2: Resource group creation: az group create --name 1keyvault --location eastus
Step 3: Creation of AKS Cluster and Configuring it with “Azure Key vault Provider” with and for “Secret Store CSI Driver support”
		For creating a AKS cluster with key vault provide and Secret CSI driver
  			az aks create --name keyvault -g 1keyvault --node-count 1 --enable-addons azure-keyvault-secrets-provider --enable-oidc-issuer --enable-workload-identity 
 
	It is creating a Cluster with the name keyvault in the resource group 1keyvault with 1 node and adding azure keyvault secrets provider and enabling OIDC which is open ID Connector, it is a protocol that allows authorisation and authentication for Kubernetes cluster to outside resources and workload identity is used for connecting k8s to any cloud resources and here it is azure. 

Step 4: to get the k8s clusters credentials as we need to connect to the cluster from the local: 
		“az aks get-credentials --resource-group 1keyvault --name keyvault”  
		verify this by using the “kubectl config view”
		to get the current cluster where you are working on use command “kubectl config current-context”
Step 5:  To verify that secrets store Csi driver pods are running or not the command we use is “kubectl get pods -n kube-system -l 'app in (secrets-store-csi-driver,secrets-store-provider-azure)' -o wide”

Step 6: Create a keyvault using  “az keyvault create -n kaushik-keyvault -g 1keyvault -l eastus --enable-rbac-authorization“
		this will create a keyvault named Kaushik-keyvault, in resource group 1keyvault and in region useast and with role based access control which is needed in next steps where I give manage identity. 

Step 7: When you create a keyvault, you can’t view your keys, secrets, and certificates and for that you need to give permission and for that go to 
		IAM- Grant access to the resource- keyvault admin access- add user (kaush910@gmail.com) now you can view all keys, secrets and certificates

Step 8: Create a Key and Secret: For Key - go to the console and create a key- Sai 
							  For Secret- go to console create Secret- Kaushik - Kaushik

Step 9: Configuring the workload Identity:
		Use these commands: “export SUBSCRIPTION_ID=fce606ae-822f-47c3-a3bf-2f09bf7cc683
export RESOURCE_GROUP=1keyvault    
export UAMI=azurekeyvaultsecretsprovider-keyvault             
export KEYVAULT_NAME=kaushik-keyvault
export CLUSTER_NAME=keyvault             

az account set --subscription $SUBSCRIPTION_ID”

Step 10: Create a manage identity: with the name: azurekeyvaultsecretsprovider-keyvault  and done with the command “az identity create --name $UAMI --resource-group $RESOURCE_GROUP

export USER_ASSIGNED_CLIENT_ID="$(az identity show -g $RESOURCE_GROUP --name $UAMI --query 'clientId' -o tsv)"
export IDENTITY_TENANT=$(az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query identity.tenantId -o tsv)”

* Client ID: Identifies the application itself.
* Tenant ID: Identifies the Azure AD directory where the application is registered and where authentication and authorization occur

Step 11: Give permission to the manage identity for the keyvault as it is used for source to connect with outside resources to the k8s cluster 
		Here I am giving admin permission: Command for that is “export KEYVAULT_SCOPE=$(az keyvault show --name $KEYVAULT_NAME --query id -o tsv)

az role assignment create --role "Key Vault Administrator" --assignee $USER_ASSIGNED_CLIENT_ID --scope $KEYVAULT_SCOPE
{”

Step 12: For  kubernetes cluster to connect to the managed identity that I have created, need to setup an OIDC and it can be done through this command
		“export AKS_OIDC_ISSUER="$(az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --query "oidcIssuerProfile.issuerUrl" -o tsv)"
echo $AKS_OIDC_ISSUER”

Step 13: Create a service account where it is accessed to the managed Identity  where it can get permission to the pod of my kubernetes cluster 
		“export SERVICE_ACCOUNT_NAME="workload-identity-sa"
export SERVICE_ACCOUNT_NAMESPACE="default"
This command will export the name and namespace of the service account. 

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: ${USER_ASSIGNED_CLIENT_ID}
  name: ${SERVICE_ACCOUNT_NAME}
  namespace: ${SERVICE_ACCOUNT_NAMESPACE}
EOF

Service account is created and it can be verified using “kubectl get sa”

Step 14: Connect the service account with the manage identity using the command “export FEDERATED_IDENTITY_NAME="aksfederatedidentity" 

az identity federated-credential create --name $FEDERATED_IDENTITY_NAME --identity-name $UAMI --resource-group $RESOURCE_GROUP --issuer ${AKS_OIDC_ISSUER} --subject system:serviceaccount:${SERVICE_ACCOUNT_NAMESPACE}:${SERVICE_ACCOUNT_NAME}”

This pod will have access to other resources and currently the pod is connected to the manage identity which has permissions to the keyvault provider. 

Step 15: Create a secret provider class where keyname: Sai and Secret name: Kaushik is added which grants permission for the cluster to get into keyvault and in the instance have permission to take key and secret which is Sai and Kaushik respectively 

The command for that is “ cat <<EOF | kubectl apply -f -
# This is a SecretProviderClass example using workload identity to access your key vault
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvname-wi # needs to be unique per namespace
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "${USER_ASSIGNED_CLIENT_ID}" # Setting this to use workload identity
    keyvaultName: ${KEYVAULT_NAME}       # Set to the name of your key vault
    cloudName: ""                         # [OPTIONAL for Azure] if not provided, the Azure environment defaults to AzurePublicCloud
    objects:  |
      array:
        - |
          objectName: Kaushik             # Set to the name of your secret
          objectType: secret              # object types: secret, key, or cert
          objectVersion: ""               # [OPTIONAL] object versions, default to latest if empty
        - |
          objectName: Sai                # Set to the name of your key
          objectType: key
          objectVersion: ""
    tenantId: "${IDENTITY_TENANT}"        # The tenant ID of the key vault
EOF
secretproviderclass.secrets-store.csi.x-k8s.io/azure-kvname-wi created”

If there are multiple keys add keys in the above code with a , and same with secrets. 

Step 16: To mount the secrets on the Secret agent I have created a sample pod, to achieve this step created a file named pod-definition.yaml and from the terminal here executed the commands. 
		This is the command for it “# This is a sample pod definition for using SecretProviderClass and workload identity to access your key vault
kind: Pod
apiVersion: v1
metadata:
  name: busybox-secrets-store-inline-wi
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: "workload-identity-sa"
  containers:
    - name: busybox
      image: registry.k8s.io/e2e-test-images/busybox:1.29-4
      command:
        - "/bin/sleep"
        - "10000"
      volumeMounts:
        - name: secrets-store01-inline
          mountPath: "/mnt/secrets-store"
          readOnly: true
  volumes:
    - name: secrets-store01-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "azure-kvname-wi”

Verify the pods using kubectl get pods and I found the pod name busybox-secrets-store-inline which is up and running 

Step 17: For mounting the secrets I have used the structures called volume, so for listing the contents in the volume, the command is “kubectl exec busybox-secrets-store-inline-wi -- ls /mnt/secrets-store/”
The output is the key and secret from the folder mnt/secrets-store/ which is “Sai” and “Kaushik”

Step 18: For the final step : List the contents in the secret and the key: I have got for the key which is a public key and the project is successful. 
