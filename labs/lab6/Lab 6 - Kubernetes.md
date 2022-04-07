# Lab 6 - Dapr in Kubernetes
In this lab, you will deploy the DaprHospital application to an Azure Kubernetes (AKS) cluster instance in your Azure subscription.

## Requirements
To finish lab, you will need the following software:
- Azure CLI
- Helm
- Docker Hub account

## Task 1: Provision an Azure SQL Database
1. Make sure to select "Allow Azure services and resources to access this server"

2. Get its connection string and copy it in a notepad

## Task 2: Provision an Azure Service Bus
1. Provision an Azure Service Bus with a Standard tier

2. Get its connection string and copy it in a notepad

## Task 3: Provision an Azure Kubernetes Service (AKS) cluster
1. Execute the following command:
    ```bash
    az aks create 
    --name daprhospitalaks 
    --resource-group daprhospital 
    --node-count 3 
    --node-vm-size Standard_D2s_v3 
    --vm-set-type VirtualMachineScaleSets 
    --enable-addons monitoring
    --generate-ssh-keys
    ```
2. Verify the status by executing the following command:
    ```bash
    az aks show --name daprhospitalaks --resource-group daprhospital
    ```
3. Install kubectl
    ```bash
    az aks install-cli
    ```
4. Get the cluster credentials
    ```powershell
    az aks get-credentials --resource-group daprhospital --name daprhospitalaks
    ```
5. View the cluster nodes
    ```bash
    kubectl get nodes
    ```
6. Initialize Dapr in the cluster
    ```bash
    dapr init -k
    ```
7. Verify Dapr in the cluster
    ```bash
    dapr status -k
    dapr dashboard -k
    ```
    
## Task 4: Update all the connection strings in the appsettings.json files
1. Replace the current connection string with the one that you copied in the notepad from Task 1.

## Task 5: Dockerize the microservices
1. Add a Dockerfile in each microservice project
    a. In Visual Studio: Right click -> Add->Docker Support...
    b. Be sure to use Linux-based images

2. Build each image by executing the following commands:

    ```bash
    docker build . -f .\DaprHospital.Person.Api\Dockerfile -t <YOUR ACCOUNT>/daprhospital.person
    docker build . -f .\DaprHospital.Patient.Api\Dockerfile -t <YOUR ACCOUNT>/daprhospital.patient
    docker build . -f .\DaprHospital.PatientQuery.Api\Dockerfile -t <YOUR ACCOUNT>/daprhospital.patientquery
    docker build . -f .\DaprHospital.Hospital.Api\Dockerfile -t <YOUR ACCOUNT>/daprhospital.hospital
    ```
## Task 6: Push the images  
1. Push the images to your Docker Hub account by executing the following command:
    ```bash
    docker push
    ```

## Task 7: Update the components
1. Inspect and update the 4 in the REPO\src\components folder with the information from the Service Bus and Storage Account that you created in **Task 1**.

2. Copy and replace all 4 modified files to the REPO\deployment folder

## Task 7: Inspect and modify the deployment files
1. Modify the following files and set the image name:
    - daprhospital.person.yaml
    - daprhospital.patient.yaml
    - daprhospital.patientquery.yaml
    - daprhospital.hospital.yaml

## Task 8: Apply the Dapr components
1. Apply the Dapr components in the REPO\deployment folder.  Make sure that they have the correct configuration (**Task 7**).
    ```bash
    kubectl apply -f .\azure-queue-input-binding.yaml
    kubectl apply -f .\azure-queue-output-binding.yaml
    kubectl apply -f .\azure-servicebus-pubsub.yaml
    kubectl apply -f .\azure-table-statestore.yaml
    ```

## Task 9: Deploy the services
1. Deploy the services by executing the following commands:
    ```bash
    kubectl apply -f .\daprhospital.person.yaml
    kubectl apply -f .\daprhospital.patient.yaml
    kubectl apply -f .\daprhospital.patientquery.yaml
    kubectl apply -f .\daprhospital.hospital.yaml
    ```

## Task 10: Create an Ingress Controller
1. Create an Ingress Controller in the AKS cluster by executing the following commands:
    ```bash
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    helm install ingress-nginx ingress-nginx/ingress-nginx --namespace default
    ```

2. Verify in the Ingress Controller is ready with the following command:
    ```bash
    kubectl --namespace default get services -o wide -w ingress-nginx-controller
    ```
    a. The EXTERNAL-IP value is the one that you are going to use in the final step of this lab.

## Task 11: Expose the services
1. Expose the four microservices by executing the following commands:
    ```bash
    kubectl apply -f  .\daprhospital.person.ingress.yaml
    kubectl apply -f  .\daprhospital.patient.ingress.yaml
    kubectl apply -f  .\daprhospital.patientquery.ingress.yaml
    kubectl apply -f  .\daprhospital.hospital.ingress.yaml
    ```

## Task 12: Test
1. Test the microservices in the AKS cluster by sending some requests to their endpoints.  For example: `/patientquery`.