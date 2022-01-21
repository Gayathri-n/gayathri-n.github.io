---
layout: post
author: Gayathri
tag: Helm
---

In Helm, one chart can depend on any number of charts. The dependencies are dynamically linked using the `dependencies` section in the `Chart.yaml` which will automatically add the charts to the `charts/` directory or the charts can be directly added to this directory as well.

```yaml
dependencies:
  - name: apache
    version: 1.2.3
    repository: https://example.com/charts
  - name: mysql
    version: 3.2.1
    repository: https://another.example.com/charts
```

One good use case is to have the common templates and utilities as a [Library chart](https://helm.sh/docs/topics/library_charts/) and use it as a dependency to your main Application chart. The dependency section will look something like for a library chart named common:

```yaml
dependencies:
  - name: common
    version: "0.0.1"
    repository: "file://../common"
```

But let's say you have charts A, B which are sub charts under a main application chart App and you also have a library chart named common, as illustrated:

![Chart hierarchy](/assets/images/chart.png)

The common chart should be included as a dependency in the `Chart.yaml` of A and B chart. It is not sufficient to just add it in the App chart.

## Build vs update

The [helm dependency build](https://helm.sh/docs/helm/helm_dependency_build/) command is used to rebuild the `charts/` directory based on `Chart.lock` file. Build is used to reconstruct a chart's dependencies to the state specified in the lock file.

The [helm dependency update](https://helm.sh/docs/helm/helm_dependency_update/) command is used to create the `charts/` directory based on `Chart.yaml`. This command verifies that the required charts, as expressed in 'Chart.yaml', are present in 'charts/' and are at an acceptable version. It will pull down the latest charts that satisfy the dependencies, and clean up old dependencies. On successful update, this will generate a lock file that can be used to rebuild the dependencies to an exact version. 

Dependencies are not required to be represented in 'Chart.yaml'. For that reason, an update command will not remove charts unless they are (a) present in the Chart.yaml file, but (b) at the wrong version. So to remove a dependency, just removing it from the `Chart.yaml` won't suffice. You will need to manually remove the `.tgz` file and the `Chart.lock` file. 



