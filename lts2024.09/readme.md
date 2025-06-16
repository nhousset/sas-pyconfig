# SAS Configurator for Open Source Options

## Overview

With open-source language integration, SAS Viya platform users can decide which
language they want to use for a given task. They can use either the SAS programming
language or an open-source programming language, such as Python, R, Lua, or
Java, to develop programs for the SAS Viya platform. This integration requires
some additional configuration.

SAS Configurator for Open Source is a utility that simplifies the download,
configuration, building, and installation of Python and R from source.
The result is a Python or R build that is located in a persistent volume (PV)
and referenced by a Persistent Volume Claim (PVC). The PVC
and the builds that it contains are then available for pods that require
Python and R for their operations.

SAS Configurator for Open Source can build and install multiple Python and R
builds or versions in the same PV. It can use profiles to handle multiple
builds. Various pods can then reference different versions or
builds of Python and R located in the PV.

SAS Configurator for Open Source also includes functionality to reduce downtime
associated with updates. A given build is located in the PV and referenced
by a pod using a symlink. In an update scenario, the symlink is changed to
point to the latest build for that profile.

For system requirements and a full set of steps to use SAS Configurator for Open
Source, see [SAS Viya Platform:
Integration with External Languages](https://documentation.sas.com/?cdcId=itopscdc&cdcVersion=default&docsetId=dplyexternlang&docsetTarget=titlepage.htm).

## Summary of Steps

Building Python or R requires a number of steps. This section describes the
steps performed by SAS Configurator for Open Source in its operations to
manage Python and R.

SAS Configurator for Open Source only processes configuration changes after
the initial execution of the job. For example, packages are reprocessed only
if a change occurs in the package list and the respective versions of
R or Python remain unchanged. If the version of Python or R changes, then all
steps are performed from the download of the source to the updating of symlinks.

### Download

For Python, downloads the source, signature file, and signer's
key from the configured location. For R, downloads only the source.

### Verify

Verifies the authenticity of the Python source using the signer's key and signature
file. The R source cannot be verified at the time of this writing because signer
keys are not generated for R source.

### Extract

Extracts the Python and R sources into a temporary directory for building.

### Build

Configures and performs a make of the Python and R sources.

### Install

Installs the Python and R builds within the PV and updates supporting
components, such as pip, if applicable.

Builds and installs configured packages for Python and R.

**Note:** Python and R packages that require additional dependencies to be
installed within any combination of the SAS Configurator for Open Source container,
the SAS Programming Environment container, and the CAS Server container are not
supported with the SAS Configurator for Open Source.

### SAS Configurator for Open Source Updates

If everything has completed successfully, creates the symbolic
links, or changes the symbolic links' targets to point to the latest builds
for both Python and R.

### Running SAS Configurator for Open Source with Custom Options at Deployment

THe SAS Configurator for Open Source utility runs a job named sas-pyconfig. When
you enable the utility, the job runs automatically
once, during the initial SAS Viya platform deployment, and runs again
with subsequent SAS Viya updates.

The official documentation for SAS Configurator for Open Source, [SAS Viya Platform:
Integration with External Languages](https://documentation.sas.com/?cdcId=itopscdc&cdcVersion=default&docsetId=dplyexternlang&docsetTarget=titlepage.htm),
provides instructions for configuring and enabling the utility.

### Resource Management

SAS Configurator for Open Source requires more CPU and memory than most
components. This requirement is largely due to Python and R building-related
operations, such as those performed by `configure` and `make`. Because SAS
Configurator for Open Source is disabled by default, pod resources are minimized
so that they are not misallocated during scheduling. The default resource values
are as follows:

```yaml
limits:
  cpu: 250m
  memory: 250Mi
requests:
  cpu: 25m
  memory: 25Mi
```

***Important:*** If the default values are used, pod execution will result in
an OOMKilled (Out of Memory Killed)
status in the pod list and the job does not complete. You must increase
the requests and limits in order for the pod to complete successfully. The
official SAS Configurator for Open Source documentation provides instructions.

- If the environment does not use resource quotas, a CPU request value of 4000m
and a memory request value of 3000mi and no limits provide a good starting point.
No limits will allow the pod to use more than requested resources if they are
available, which can result in a shorter time to completion. With these values,
the pod should complete its operations in approximately 15 minutes and before the
environment is stable enough for widespread use. Differences in hardware
specifications will have an impact on the time it takes for the pod to complete.

- If the environment uses resource quotas, the specified limit values must be
equal to or greater than the respective request values for CPU and memory.

The values of requests and limits can be adjusted to meet specific needs of an
environment. For example, reduce values to allow scheduling within
smaller environments, or increase values to reduce
the time required to build multiple versions of Python and R.

A YAML file is provided in your deployment assets to help you
increase CPU and memory requests. By default, the recommended
CPU and memory requests are specified in the file (change-limits.yaml),
and no limits are specified. Below are some examples of
updates to this file.

#### Changing Resource Limits: Example 1

In this example, SAS Open Source Configuration is configured with a CPU request
value of 4000m and memory request value of 3000mi. No limit to CPU and memory
usage is specified. This configuration should not be used in environments where
resource quotas are in use.

```yaml
---
apiVersion: builtin
kind: PatchTransformer
metadata:
  name: sas-pyconfig-limits
patch: |-
  - op: replace
    path: /spec/jobTemplate/spec/template/spec/containers/0/resources/requests/cpu
    value:
      4000m
  - op: replace
    path: /spec/jobTemplate/spec/template/spec/containers/0/resources/requests/memory
    value:
      3000Mi
  - op: remove
    path: /spec/jobTemplate/spec/template/spec/containers/0/resources/limits/cpu
  - op: remove
    path: /spec/jobTemplate/spec/template/spec/containers/0/resources/limits/memory
target:
  group: batch
  kind: CronJob
  name: sas-pyconfig
  version: v1
#---
#apiVersion: builtin
#kind: PatchTransformer
#metadata:
#  name: sas-pyconfig-limits
#patch: |-
#  - op: replace
#    path: /spec/jobTemplate/spec/template/spec/containers/0/resources/requests/cpu
#    value:
#      4000m
#  - op: replace
#    path: /spec/jobTemplate/spec/template/spec/containers/0/resources/requests/memory
#    value:
#      3000Mi
#  - op: replace
#    path: /spec/jobTemplate/spec/template/spec/containers/0/resources/limits/cpu
#    value:
#      4000m
#  - op: replace
#    path: /spec/jobTemplate/spec/template/spec/containers/0/resources/limits/memory
#    value:
#      3000Mi
#target:
#  group: batch
#  kind: CronJob
#  name: sas-pyconfig
```

#### Changing Resource Limits: Example 2

In this example, both requests and limits values for CPU and memory have been
set to 4000m and 3000mi, respectively. This configuration can be used in an
environment where resource quotas are enabled.

```yaml
#---
#apiVersion: builtin
#kind: PatchTransformer
#metadata:
#  name: sas-pyconfig-limits
#patch: |-
#  - op: replace
#    path: /spec/jobTemplate/spec/template/spec/containers/0/resources/requests/cpu
#    value:
#      4000m
#  - op: replace
#    path: /spec/jobTemplate/spec/template/spec/containers/0/resources/requests/memory
#    value:
#      3000Mi
#  - op: remove
#    path: /spec/jobTemplate/spec/template/spec/containers/0/resources/limits/cpu
#  - op: remove
#    path: /spec/jobTemplate/spec/template/spec/containers/0/resources/limits/memory
#target:
#  group: batch
#  kind: CronJob
#  name: sas-pyconfig
#  version: v1
---
apiVersion: builtin
kind: PatchTransformer
metadata:
  name: sas-pyconfig-limits
patch: |-
  - op: replace
    path: /spec/jobTemplate/spec/template/spec/containers/0/resources/requests/cpu
    value:
      4000m
  - op: replace
    path: /spec/jobTemplate/spec/template/spec/containers/0/resources/requests/memory
    value:
      3000Mi
  - op: replace
    path: /spec/jobTemplate/spec/template/spec/containers/0/resources/limits/cpu
    value:
      4000m
  - op: replace
    path: /spec/jobTemplate/spec/template/spec/containers/0/resources/limits/memory
    value:
      3000Mi
target:
  group: batch
  kind: CronJob
  name: sas-pyconfig
```

### Change the Configuration and Rerun the Job

You can change the configuration and run the sas-pyconfig job again without
redeploying the SAS Viya platform. The [official
SAS Configurator for Open Source documentation](https://documentation.sas.com/?cdcId=itopscdc&cdcVersion=default&docsetId=dplyexternlang&docsetTarget=n09x4mz6i4ygb2n12p0804tvajpt.htm)
describes the steps to run the
job manually and install and configure Python or R from source.

### Disable SAS Configurator for Open Source

By default, SAS Configurator for Open Source is disabled.

1. Determine the exact name of the sas-pyconfig-parameters ConfigMap:

   ```bash
   kubectl get configmaps -n <name-of-namespace> | grep sas-pyconfig`
   ```

   The name will be something like sas-pyconfig-parameters-abcd1234.

2. Edit the ConfigMap using the following command:

   ```bash
   kubectl edit configmap <sas-pyconfig-parameters-configmap-name> -n <name-of-namespace>
   ```

   In this example, `sas-pyconfig-parameters-configmap-name` is the name of the
   ConfigMap from step 1. Change the value of `global.enabled` to `false`.

SAS Configurator for Open Source does not run during a deployment or update of
the SAS Viya platform.

## Default Configuration and Options

The configuration options used by SAS Configurator for Open Source are referenced
from the sas-pyconfig-parameters ConfigMap (provided for you in the
change-configuration.yaml file).
The [official
SAS Configurator for Open Source documentation](https://documentation.sas.com/?cdcId=itopscdc&cdcVersion=default&docsetId=dplyexternlang&docsetTarget=p0kaxaz5x6y3oin1vqnednekblm7.htm)
describes the options
available in the ConfigMap, their purpose, and their default values.

Configuration options fall into two main categories:

- global options

  Options that are applied across or related to all profiles and to the application.

- profile options

  Options that are specific to a profile.

For a description of each global option, including the option to specify an
HTTP or HTTPS web proxy server, see the [official
SAS Configurator for Open Source documentation](https://documentation.sas.com/?cdcId=itopscdc&cdcVersion=default&docsetId=dplyexternlang&docsetTarget=n0pseimf5cq11kn1k16e5w0ma2lc.htm).

Profiles are references to different versions or builds of Python and
R in the PV, enabling SAS Configurator for Open Source to manage multiple
builds of Python or R.

The predefined
Python profile is named "default_py", and the predefined R profile is named
"default_r". Profiles are described in detail in the [official
SAS Configurator for Open Source documentation](https://documentation.sas.com/?cdcId=itopscdc&cdcVersion=default&docsetId=dplyexternlang&docsetTarget=p0kaxaz5x6y3oin1vqnednekblm7.htm).

## Example Patch File 1

The following example change-configuration.yaml file contains the predefined profiles
only:

```yaml
apiVersion: builtin
kind: PatchTransformer
metadata:
  name: sas-pyconfig-custom-parameters
patch: |-
  - op: replace 
    path: /data/global.enabled
    value: "false"
  - op: replace 
    path: /data/global.python_enabled
    value: "false"
  - op: replace 
    path: /data/global.r_enabled
    value: "false"
  - op: replace
    path: /data/global.pvc
    value: "/opt/sas/viya/home/sas-pyconfig"
  - op: replace
    path: /data/global.python_profiles
    value: "default_py"
  - op: replace
    path: /data/global.r_profiles
    value: "default_r"
  - op: replace
    path: /data/global.dry_run
    value: "false"
  - op: replace
    path: /data/global.http_proxy
    value: "none"
  - op: replace
    path: /data/global.https_proxy
    value: "none"
  - op: replace
    path: /data/default_py.configure_opts
    value: "--enable-optimizations"
  - op: replace
    path: /data/default_r.configure_opts
    value: "--enable-memory-profiling --enable-R-shlib --with-blas --with-lapack --with-readline=no --with-x=no"
  - op: replace
    path: /data/default_py.cflags
    value: "-fPIC"
  - op: replace
    path: /data/default_r.cflags
    value: "-fPIC"
  - op: replace
    path: /data/default_py.pip_install_packages
    value: "Prophet sas_kernel matplotlib sasoptpy sas-esppy NeuralProphet scipy==1.10 Flask XGBoost TensorFlow pybase64 scikit-learn statsmodels sympy mlxtend Skl2onnx nbeats-pytorch ESRNN onnxruntime opencv-python zipfile38 json2 pyenchant nltk spacy gensim pyarrow hnswlib==0.7.0 sas-ipc-queue great-expectations==0.16.8"
  - op: replace
    path: /data/default_py.pip_r_packages
    value: "rpy2"
  - op: replace
    path: /data/default_py.pip_r_profile
    value: "default_r"
  - op: replace
    path: /data/default_py.python_signer
    value: https://keybase.io/pablogsal/pgp_keys.asc
  - op: replace
    path: /data/default_py.python_signature
    value: https://www.python.org/ftp/python/3.10.14/Python-3.10.14.tgz.asc
  - op: replace
    path: /data/default_py.python_tarball
    value: https://www.python.org/ftp/python/3.10.14/Python-3.10.14.tgz
  - op: replace
    path: /data/default_r.r_tarball
    value: https://cloud.r-project.org/src/base/R-4/R-4.3.3.tar.gz
  - op: replace
    path: /data/default_r.packages
    value: "dplyr jsonlite httr tidyverse randomForest xgboost forecast"
  - op: replace
    path: /data/default_r.pkg_repos
    value: "https://cran.rstudio.com/ http://cran.rstudio.com/ https://cloud.r-project.org/ http://cloud.r-project.org/"

target:
  version: v1
  kind: ConfigMap
  name: sas-pyconfig-parameters
```

## Example Patch File 2

The following example change-configuration.yaml file adds a Python profile called
"myprofile" to the global.profiles list and adds profile options for
"myprofile". Note that the default Python profile is still listed and will also be
built.

```yaml
apiVersion: builtin
kind: PatchTransformer
metadata:
  name: sas-pyconfig-custom-parameters
patch: |-
  - op: replace
    path: /data/global.enabled
    value: "true"
  - op: replace
    path: /data/global.python_profiles
    value: "default_py myprofile"
  - op: add
    path: /data/myprofile.configure_opts
    value: "--enable-optimizations"
  - op: add
    path: /data/myprofile.cflags
    value: "-fPIC"
  - op: add
    path: /data/myprofile.pip_install_packages
    value: "Prophet sas_kernel matplotlib sasoptpy sas-esppy NeuralProphet scipy==1.10 Flask XGBoost TensorFlow pybase64 scikit-learn statsmodels sympy mlxtend Skl2onnx nbeats-pytorch ESRNN onnxruntime opencv-python zipfile38 json2 pyenchant nltk spacy gensim pyarrow hnswlib==0.7.0 sas-ipc-queue great-expectations==0.16.8"
  - op: replace
    path: /data/myprofile.pip_r_packages
    value: "rpy2"
  - op: replace
    path: /data/myprofile.pip_r_profile
    value: "default_r"
  - op: add
    path: /data/myprofile.python_signer
    value: https://keybase.io/pablogsal/pgp_keys.asc
  - op: add
    path: /data/myprofile.python_signature
    value: https://www.python.org/ftp/python/3.10.14/Python-3.10.14.tgz.asc
  - op: add
    path: /data/myprofile.python_tarball
    value: https://www.python.org/ftp/python/3.10.14/Python-3.10.14.tgz
target:
  version: v1
  kind: ConfigMap
  name: sas-pyconfig-parameters
```
