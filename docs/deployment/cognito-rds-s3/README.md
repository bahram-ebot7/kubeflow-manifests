# Deploying Kubeflow with Amazon Cognito as idP, RDS and S3

This guide describes how to deploy Kubeflow on AWS EKS using Cognito as identity provider, RDS for database and S3 for artifact storage.

## 1. Prerequisites
Follow the pre-requisites section from [this guide](../rds-s3/README.md#1-prerequisites) and setup RDS & S3 from [this guide](../rds-s3/README.md#20-setup-rds-s3-and-configure-secrets) to:
1. Install the CLI tools
1. Clone the repo
1. Create an EKS cluster and
1. Create S3 Bucket
1. Create RDS Instance
1. Configure AWS Secrets for RDS and S3
1. Install AWS Secrets and Kubernetes Secrets Store CSI driver
1. Configure RDS endpoint and S3 bucket name for Kubeflow Pipelines

## Configure Custom Domain and Cognito

1. Follow the [cognito guide](../cognito/README.md) from [section 1.0(Custom Domain)](../cognito/README.md#10-custom-domain) upto [section 4.0(Configure Ingress)](../cognito/README.md#40-configure-ingress) to:
    1. Create a custom domain
    1. Create TLS certificates for the domain
    1. Create a Cognito Userpool
    1. Configure Ingress
2. Deploy Kubeflow. Choose one of the two options to deploy kubeflow:
    1. **[Option 1]** Install with a single command
        ```
        while ! kustomize build docs/deployment/cognito-rds-s3 | kubectl apply -f -; do echo "Retrying to apply resources"; sleep 10; done
        ```
    1. **[Option 2]** Install individual components
        ```
        # Kubeflow namespace
        kustomize build upstream/common/kubeflow-namespace/base | kubectl apply -f -
        
        # Kubeflow Roles
        kustomize build upstream/common/kubeflow-roles/base | kubectl apply -f -
        
        # Istio
        kustomize build upstream/common/istio-1-9/istio-crds/base | kubectl apply -f -
        kustomize build upstream/common/istio-1-9/istio-namespace/base | kubectl apply -f -
        kustomize build upstream/common/istio-1-9/istio-install/base | kubectl apply -f -

        # Cert-Manager
        kustomize build upstream/common/cert-manager/cert-manager/base | kubectl apply -f -
        kustomize build upstream/common/cert-manager/kubeflow-issuer/base | kubectl apply -f -
        
        # KNative
        kustomize build upstream/common/knative/knative-serving/overlays/gateways | kubectl apply -f -
        kustomize build upstream/common/knative/knative-eventing/base | kubectl apply -f -
        kustomize build upstream/common/istio-1-9/cluster-local-gateway/base | kubectl apply -f -
        
        # Kubeflow Istio Resources
        kustomize build upstream/common/istio-1-9/kubeflow-istio-resources/base | kubectl apply -f -
        
        # KFServing
        kustomize build upstream/apps/kfserving/upstream/overlays/kubeflow | kubectl apply -f -
        
        # Central Dashboard
        kustomize build upstream/apps/centraldashboard/upstream/overlays/istio | kubectl apply -f -
        
        # Notebooks
        kustomize build upstream/apps/jupyter/notebook-controller/upstream/overlays/kubeflow | kubectl apply -f -
        kustomize build awsconfigs/apps/jupyter-web-app | kubectl apply -f -
        
        # Admission Webhook
        kustomize build upstream/apps/admission-webhook/upstream/overlays/cert-manager | kubectl apply -f -
        
        # Profiles + KFAM
        kustomize build upstream/apps/profiles/upstream/overlays/kubeflow | kubectl apply -f -
        
        # Volumes Web App
        kustomize build upstream/apps/volumes-web-app/upstream/overlays/istio | kubectl apply -f -
        
        # Tensorboard
        kustomize build upstream/apps/tensorboard/tensorboards-web-app/upstream/overlays/istio | kubectl apply -f -
        kustomize build upstream/apps/tensorboard/tensorboard-controller/upstream/overlays/kubeflow | kubectl apply -f -
        
        # MPI Operator
        kustomize build upstream/apps/mpi-job/upstream/overlays/kubeflow | kubectl apply -f -

        # Training Operator
        kustomize build upstream/apps/training-operator/upstream/overlays/kubeflow | kubectl apply -f -

        # AWS Telemetry - This is an optional component. See usage tracking documentation for more information
        kustomize build awsconfigs/common/aws-telemetry | kubectl apply -f -

        # AWS Secret Manager
        kustomize build awsconfigs/common/aws-secrets-manager/base | kubectl apply -f -

        # Kubeflow Pipelines
        kustomize build awsconfigs/apps/pipeline | kubectl apply -f -

        # Katib
        kustomize build awsconfigs/apps/katib-external-db-with-kubeflow | kubectl apply -f -

        # Configured for AWS Cognito
        
        # Ingress
        kustomize build awsconfigs/common/istio-ingress/overlays/cognito | kubectl apply -f -

        # ALB controller
        kustomize build awsconfigs/common/aws-alb-ingress-controller/base | kubectl apply -f -

        # Envoy filter
        kustomize build awsconfigs/common/aws-istio-envoy-filter/base | kubectl apply -f -        
        ```
1. Follow the rest of the cognito guide from [section 6.0(Updating the domain with ALB address)](../cognito/README.md#60-updating-the-domain-with-ALB-address) to:
    1. Add/Update the DNS records in custom domain with the ALB address
    1. Create a user in Cognito user pool
    1. Create a profile for the user from the user pool
    1. Connect to the central dashboard
