# k0rdent FluxCD repo template

## Prerequisites

1. `kubectl` binary installed and added to the PATH
2. This repo uses `task-go` utility to run commands. Please [install](https://taskfile.dev/installation/) it and add it to the PATH
3. To generate manifest we use `gomplate` template engine. Please [install](https://docs.gomplate.ca/installing/) it and add to the PATH
4. kubernetes cluster that will be used as the [management cluster](https://k0rdent.github.io/docs/glossary/#management-cluster). Configure kubectl to connect and use this cluster
5. If you decide to bootstrap FluxCD using this repo (other options described in next steps), you need to [install](https://fluxcd.io/flux/installation/) `flux` CLI and add it to the PATH


## Produced GitOps repo structure

```
/
├── base                                                # base configurations that can be reused by kustomizations for management clusters
│   ├── components                                      # management cluster components
│   │   └── k0rdent                                     # k0rdent platform base installation configs
│   └── k0rdent                                         # k0rdent objects base configurations
│       ├── cluster-deployments                         # k0rdent ClusterDeployment base
│       ├── configuration                               # k0rdent configuration (Management, AccessManagement, etc)
│       ├── credentials                                 # k0rdent Credential bases
│       │   ├── aws                                     # k0rdent AWS Credential base
│       │   ├── azure                                   # k0rdent Azure Credential base
│       │   └── openstack                               # k0rdent Openstack Credential base
│       ├── multiclusterservices                        # k0rdent MultiClusterService base
│       └── templates                                   # k0rdent template bases
│           ├── clusters                                # k0rdent ClusterTemplate base
│           └── services                                # k0rdent ServiceTemplate base
└── management-clusters                                 # management clusters configurations
    ├── cluster-1                                       # configuration for the "cluster-1" management cluster
    │   ├── kustomization.yaml                          # management cluster main kustomization object that generates manifests
    │   ├── components                                  # management cluster components configuration
    │   │   └── k0rdent                                 # management cluster k0rdent platform installation
    │   ├── flux                                        # management cluster Flux CD sync configs
    │   │   ├── flux-system                             # management cluster flux-system sync
    │   │   ├── git-repository.yaml                     # management cluster flux GitRepository that is used in k0rdent flux sync configs
    │   │   └── k0rdent-kcm.yaml                        # management cluster k0rdent kcm component flux sync configs
    │   └── k0rdent                                     # management cluster k0rdent objects and configurations
    │      ├── cluster-deployments                      # management cluster ClusterDeployment objects directory
    │      ├── configuration                            # management cluster k0rdent platform configuration
    │      ├── credentials                              # management cluster Credential objects directory
    │      ├── multiclusterservices                     # management cluster MultiClusterService objects directory
    │      ├── templates                                # management cluster custom template objects directory
    │      │   ├── clusters                             # management cluster ClusterTemplate objects directory
    │      │   └── services                             # management cluster ServiceTemplate objects directory
    │      └── k0rdent                                  # management cluster k0rdent objects and configurations
    └── cluster-2                                       
    ...
    └── cluster-2                                       
```

## Setup the repo

### Stage 1. Init configs

> [!IMPORTANT]
> The [config.sample.yaml](./config.sample.yaml) file contains variables that are **vital** to the template process.

1. Genertae the `config.yaml` from the [config.sample.yaml](./config.sample.yaml) configuration file:

    ```shell
    task init
    ```

2. Fill out the `config.yaml` configuration file using the comments in that file as a guide.

### Stage 2. Bootstrap and configuring the Gitops tool

> [!IMPORTANT]
> If you already have the installed FluxCD / ArgoCD in your management cluster, you can skip this stage

#### Configuring FluxCD
1. In the generated [`config.yaml`](./config.yaml) file, specify the list of k0rdent management cluster names or their aliases under the `managementClusters` property. For example, you can separate management clusters by environments - dev, staging, prod, etc. It's required to specify at leas one cluster
2. Switch your local kubectl context to the first kubernetes cluster that will be used as the management cluster
3. Run the bootstrap FluxCD command with the management cluster variable that has the exactly same value as the appropriate one from the `managementClusters` list for the current kubectl context. For example, if you have the `management-cluster-1` value in the `managementClusters` list and you switched the kubectl context to the corresponding cluster:
    ```shell
    MANAGEMENT_CLUSTER=management-cluster-1 task bootstrap:flux 
    ```
4. Repeat steps 2-3 for each management cluster

#### Configuring ArgoCD
1. In the generated [`config.yaml`](./config.yaml) file, specify the list of k0rdent management cluster names or their aliases under the `managementClusters` property. For example, you can separate management clusters by environments - dev, staging, prod, etc. It's required to specify at leas one cluster
2. Switch your local kubectl context to the first kubernetes cluster that will be used as the management cluster
3. Run the bootstrap ArgoCD command with the management cluster variable that has the exactly same value as the appropriate one from the `managementClusters` list for the current kubectl context. For example, if you have the `management-cluster-1` value in the `managementClusters` list and you switched the kubectl context to the corresponding cluster:
    ```shell
    MANAGEMENT_CLUSTER=management-cluster-1 task bootstrap:argo 
    ```
4. Repeat steps 2-3 for each management cluster

### Stage 3. Generate k0rdent configuration

1. Template out all the configuraion files:

    ```shell
    task configure
    ```

2. Push your changes to git:

    ```shell
    git add -A
    git commit -m "initial k0rdent configuration"
    git push
    ```

### Stage 3. (Optional) Connect this repo to the existing FluxCD installation

If you already use FluxCD and didn't install using the current setup guide, add the current repo to it and make it sync Flux configs from the the `management-clusters/<cluster-name>/flux` directory

### Stage 4. Watch the rollout of k0rdent

Currently only KCM k0rdent component is installed with this repo. When KCM controller is installed, it starts to bootstrap all the required configuration, providers and components. To monitor the KCM installation you can watch the `Management` type object. By default, it's called `kcm`. 

> Creation of the Management object can take some time

```shell
kubectl -n kcm-system get management.k0rdent.mirantis.com kcm -o go-template='{{range $key, $value := .status.components}}{{$key}}: {{if $value.success}}{{$value.success}}{{else}}{{$value.error}}{{end}}{{"\n"}}{{end}}'
```

All components in the list must have the value `true`

### Optional: Enable Renovatebot for Dependency Updates

To automatically keep dependencies up to date, you can enable [Renovatebot](https://github.com/renovatebot/renovate) by following these steps:

1. **Run the bootstrap task**
   Execute the following command to set up Renovate:
   ```sh
   task bootstrap:renovate
   ```

2. **Create a GitHub Personal Access Token (PAT)**
   - Go to [GitHub Developer Settings](https://github.com/settings/tokens).
   - Generate a new **fine-grained** or **classic** PAT with the necessary repository permissions.
   - Copy the generated token.

3. **Save the PAT to GitHub Actions Secrets**
   - Navigate to your repository on GitHub.
   - Go to **Settings > Secrets and variables > Actions**.
   - Click **New repository secret** and name it:
     ```
     RENOVATE_TOKEN
     ```
   - Paste the copied token and save.

Once configured, Renovatebot will automatically create pull requests for dependency updates based on the configured rules.

## Adding further Management Clusters
To create manifests for further management clusters we created a [helper tool](https://github.com/Mirantis-PS/gitops-helper) that maintains consistency and reproducibility.

The tool must be from within this cloned repository:
1. Run the gitops-helper tool using Docker
    ```sh
    docker run -it -v $(pwd):/repo ghcr.io/mirantis-ps/gitops-helper:main create managed-cluster
    ```
1. Review and modify the generated configuration files as needed.
1. Commit and push the files to your Git repository.
1. Watch your GitOps-managed cluster to sync the state.
