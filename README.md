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

### Stage 2. Bootstrap or configure FluxCD

> [!IMPORTANT]
> If you already have the installed Flux CD in your management cluster, you can skip this stage

1. In the generated [`config.yaml`](./config.yaml) file, specify the list of k0rdent management cluster names or their aliases under the `managementClusters` property. For example, you can separate management clusters by environments - dev, staging, prod, etc. It's required to specify at leas one cluster
2. Switch your local kubectl context to the first kubernetes cluster that will be used as the management cluster
3. Run the bootstrap FluxCD command with the management cluster variable that has the exactly same value as the appropriate one from the `managementClusters` list for the current kubectl context. For example, if you have the `management-cluster-1` value in the `managementClusters` list and you switched the kubectl context to the corresponding cluster:
    ```shell
    MANAGEMENT_CLUSTER=management-cluster-1 task bootstrap:flux 
    ```
4. Repeat steps 2-3 for each management cluster

If it's planned to use multiple management clusters and each of them will use the separate FluxCD installation:

1. In the generated [`config.yaml`](./config.yaml) file, specify the list of management cluster names or their aliases under the `environments` property. For example, you can separate management clusters by environments - dev, staging, prod, etc.
2. Switch your local kubectl context to the first kubernetes cluster that will be used as the management cluster
3. Run the bootstrap FluxCD command with the environment variable that has the exactly same value as the appropriate one from the `environments` list for the current management cluster. For example, if you have the `dev` value in the `environments` list and you switched the kubectl context to the corresponding cluster:
    ```shell
    ENVIRONMENT=dev task bootstrap:flux 
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

