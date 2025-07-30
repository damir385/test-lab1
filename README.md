# ArgoCD GitOps Project

This repository contains a simple GitOps project for ArgoCD with a Nginx Helm chart. It demonstrates how to use GitHub Actions to modify ArgoCD configuration and let ArgoCD handle the actual deployment/deletion of applications in Kubernetes, following the GitOps principles. The project allows deploying the same chart to multiple namespaces with different replica counts.

## Repository Structure

```
.
├── .github/workflows/    # GitHub Actions workflows
│   └── deploy.yml        # Workflow for updating ArgoCD configuration
├── charts/               # Helm charts
│   ├── nginx-0.1.0/      # Nginx Helm chart version 0.1.0
│   │   ├── Chart.yaml    # Chart metadata
│   │   ├── values.yaml   # Default values (Nginx 1.21.6)
│   │   ├── templates/    # Kubernetes templates
│   └── nginx-0.2.0/      # Nginx Helm chart version 0.2.0
│       ├── Chart.yaml    # Chart metadata
│       ├── values.yaml   # Default values (Nginx 1.22.0)
│       ├── templates/    # Kubernetes templates
├── config/               # Configuration files
│   └── nginx-deployments/# Deployment configurations for ApplicationSet
│       └── nginx-ns1.yaml# Example deployment configuration
├── manifests/            # Kubernetes manifests
│   └── argocd/           # ArgoCD application manifests
│       └── nginx-applicationset.yaml  # ApplicationSet for managing multiple deployments
└── README.md             # This file
```

## Nginx Helm Chart

The repository includes two versions of a simple Nginx Helm chart:

1. **nginx-0.1.0**: The original version using Nginx 1.21.6
2. **nginx-0.2.0**: An updated version using Nginx 1.22.0

Both charts include the following resources:
- Deployment
- Service
- HorizontalPodAutoscaler (optional)

### Configuration

#### Chart Version 0.1.0

The configuration in `nginx-0.1.0/values.yaml` includes:
- 1 replica (can be overridden through the ApplicationSet)
- Nginx 1.21.6 image
- ClusterIP service type
- Resource limits and requests
- Disabled autoscaling

#### Chart Version 0.2.0

The configuration in `nginx-0.2.0/values.yaml` includes:
- 1 replica (can be overridden through the ApplicationSet)
- Nginx 1.22.0 image (newer version)
- ClusterIP service type
- Resource limits and requests
- Disabled autoscaling

The project uses the Git Generator with Files approach in ArgoCD ApplicationSet to deploy the Nginx chart to multiple namespaces with different configurations. Each deployment is defined in a separate YAML file in the `config/nginx-deployments` directory, with the filename matching the application name (e.g., `nginx-ns1.yaml`).

Each configuration file specifies:
- The name of the application (automatically generated as "nginx-{namespace}")
- The namespace to deploy to
- The number of replicas
- The Helm chart version to use (nginx-0.1.0 or nginx-0.2.0)

The ApplicationSet watches these configuration files and automatically creates or updates applications based on their content.

## GitHub Workflow

The repository includes a GitHub workflow (`deploy.yml`) that follows GitOps principles:
1. Create or update configuration files for deployments in the `config/nginx-deployments` directory
2. Delete configuration files to remove deployments
3. Update configuration files to modify replica count and/or chart version for existing deployments

When deploying, the workflow:
1. Checks if a configuration file already exists for the deployment (using "nginx-{namespace}" as the identifier)
2. If it doesn't exist, creates a new configuration file with the automatically generated name, namespace, replica count, and chart version
3. If it exists, updates the existing configuration file with the new values
4. Creates a pull request using the peter-evans/create-pull-request action, which:
   - Automatically creates a new branch
   - Commits the changes to that branch
   - Creates a pull request from the new branch to the main branch
5. Enables auto-merge for the pull request using the nick-fields/retry action
6. ArgoCD detects the changes in the main branch, processes the updated configuration files, and applies the changes to the Kubernetes cluster

When deleting, the workflow:
1. Deletes the configuration file for the specified deployment
2. Creates a pull request using the peter-evans/create-pull-request action, which:
   - Automatically creates a new branch
   - Commits the changes to that branch
   - Creates a pull request from the new branch to the main branch
3. Enables auto-merge for the pull request using the nick-fields/retry action
4. ArgoCD detects the changes in the main branch, notices the missing configuration file, and removes the application from the cluster

When updating, the workflow:
1. Checks if a configuration file exists for the deployment (using "nginx-{namespace}" as the identifier)
2. If it exists, reads the current values and updates the replica count and/or chart version based on the provided inputs
3. If it doesn't exist, shows an error message
4. Creates a pull request using the peter-evans/create-pull-request action, which:
   - Automatically creates a new branch
   - Commits the changes to that branch
   - Creates a pull request from the new branch to the main branch
5. Enables auto-merge for the pull request using the nick-fields/retry action
6. ArgoCD detects the changes in the main branch, processes the updated configuration file, and applies the updates to the deployment
7. If the chart version was updated, the new chart version is applied, which may include a different Nginx image version (1.21.6 for nginx-0.1.0, 1.22.0 for nginx-0.2.0)

### Workflow Inputs

The workflow accepts the following inputs:
- **action**: `deploy`, `delete`, or `update` (required)
  - `deploy`: Add or update a deployment
  - `delete`: Remove a deployment
  - `update`: Update replica count and/or chart version for an existing deployment
- **namespace**: Kubernetes namespace to deploy to (required)
- **replicaCount**: Number of replicas (required for `deploy` action, optional for `update` action)
- **chartVersion**: Helm chart version to use (required for `deploy` action, optional for `update` action)
  - `nginx-0.1.0`: Original version with Nginx 1.21.6
  - `nginx-0.2.0`: Updated version with Nginx 1.22.0

The application name is automatically generated as "nginx-{namespace}" to simplify the workflow and ensure consistent naming.

### Prerequisites

To use this workflow, you need to:
1. Set up a Kubernetes cluster
2. Install ArgoCD in your Kubernetes cluster
3. Configure ArgoCD to watch this repository
4. Ensure the GitHub workflow has the necessary permissions:
   - `contents: write` - To push changes to the repository
   - `pull-requests: write` - To create and merge pull requests

The workflow is configured with these permissions by default, but if you're using a custom GitHub Actions setup or organization policies, you may need to ensure these permissions are granted.

### Usage

1. Go to the "Actions" tab in your GitHub repository
2. Select the "Update ArgoCD Configuration for Nginx Application" workflow
3. Click "Run workflow"
4. Fill in the required inputs:
   - **action**: Choose one of the following:
     - `deploy`: To create a new configuration file or update an existing one
     - `delete`: To remove a configuration file
     - `update`: To update replica count and/or chart version in an existing configuration file
   - **namespace**: Specify the Kubernetes namespace to deploy to
   - **replicaCount**: Specify the number of replicas (required for `deploy` action, optional for `update` action)
   - **chartVersion**: Select the Helm chart version to use (required for `deploy` action, optional for `update` action):
     - `nginx-0.1.0`: To use the original version with Nginx 1.21.6
     - `nginx-0.2.0`: To use the updated version with Nginx 1.22.0
   
   For the `update` action, you must specify at least one of `replicaCount` or `chartVersion`.
5. Click "Run workflow" to start the update process
6. The workflow will:
   - Create, update, or delete a configuration file in the `config/nginx-deployments` directory
   - Create a pull request with the changes
   - Automatically merge the pull request
7. After the pull request is merged, ArgoCD will:
   - Detect the changes in the repository
   - Process the updated configuration files
   - Apply the changes to the cluster

The application will be named "nginx-{namespace}" automatically, and the configuration file will be named accordingly (e.g., `nginx-ns1.yaml`). This simplifies the deployment process and ensures consistent naming across your cluster.

#### Example Configuration File

Here's an example of what a configuration file looks like:

```yaml
name: nginx-ns1
namespace: ns1
replicaCount: 3
chartVersion: nginx-0.1.0
```

Each configuration file contains these four parameters, which are used by the ApplicationSet to generate the application.

## ArgoCD Integration

This repository is designed to work with ArgoCD following GitOps principles. The `manifests/argocd` directory contains an ArgoCD ApplicationSet manifest that manages multiple deployments of the same Helm chart.

### ArgoCD ApplicationSet

The repository uses an ApplicationSet (`nginx-applicationset.yaml`) with the Git Generator with Files approach to manage multiple deployments of the Nginx Helm chart. The ApplicationSet:
- Uses a Git Generator with Files to scan the `config/nginx-deployments` directory for YAML files
- Treats each YAML file as a deployment configuration
- Creates an application for each configuration file found
- Extracts parameters like name, namespace, replicaCount, and chartVersion from each file
- Selects the appropriate chart directory based on the specified chart version (nginx-0.1.0 or nginx-0.2.0)
- Uses the default image version that comes with each chart version (1.21.6 for nginx-0.1.0, 1.22.0 for nginx-0.2.0)
- Maintains consistent sync policies across all deployments

This approach treats the ApplicationSet as immutable, with all mutations happening in the configuration files rather than in the ApplicationSet itself. This provides better separation of concerns, easier tracking of changes, and more flexibility in managing deployments.

### Chart Version vs. Image Version

This project uses chart versions instead of directly specifying image versions for several benefits:

1. **Default Image Compatibility**: Each chart version comes with a default image version that has been tested and verified to work with that specific chart version.

2. **Handling Value Structure Changes**: Different chart versions may have different values structures. By specifying the chart version and not overriding the image version:
   - The default values structure for that chart version is used
   - You avoid compatibility issues that might arise from mixing newer image versions with older chart structures
   - You can safely upgrade to newer chart versions that might have different value structures

3. **GitOps Best Practices**: This approach follows GitOps best practices by:
   - Versioning the entire chart, not just the container image
   - Ensuring that all changes (including value structure changes) are captured in the Git history
   - Making rollbacks simpler by reverting to a previous chart version

### Setting Up ArgoCD

To use this repository with ArgoCD:

1. Install ArgoCD in your Kubernetes cluster:
   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

2. Access the ArgoCD UI and log in

3. Register this repository with ArgoCD:
   - Go to Settings > Repositories
   - Add this repository URL
   - Configure authentication if needed

4. Apply the ArgoCD ApplicationSet manifest:
   ```bash
   kubectl apply -f manifests/argocd/nginx-applicationset.yaml
   ```

5. ArgoCD will automatically sync the applications when changes are pushed to the repository

### GitOps Workflow

The GitOps workflow with ArgoCD works as follows:

1. The GitHub workflow creates, updates, or deletes configuration files in the `config/nginx-deployments` directory
2. Changes are committed and pushed to the repository through a pull request
3. After the pull request is merged, ArgoCD detects the changes in the repository
4. ArgoCD scans the `config/nginx-deployments` directory and processes the configuration files
5. ArgoCD creates, updates, or deletes applications based on the configuration files
6. ArgoCD continuously monitors the cluster to ensure the desired state matches the actual state

This approach follows the GitOps principle of treating the ApplicationSet as immutable, with all mutations happening in the configuration files. This provides several benefits:

1. **Better Separation of Concerns**: The ApplicationSet defines how to process configuration files, while the configuration files define what to deploy
2. **Easier Tracking of Changes**: Each deployment has its own configuration file, making it easier to track changes to specific deployments
3. **More Flexibility**: Configuration files can be added, updated, or removed without modifying the ApplicationSet itself
4. **Improved Scalability**: The approach scales well to many deployments without making the ApplicationSet unwieldy

## License

This project is licensed under the MIT License - see the LICENSE file for details.