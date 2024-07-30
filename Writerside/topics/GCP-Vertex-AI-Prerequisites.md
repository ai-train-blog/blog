# GCP Vertex AI Prerequisites

We describe installation of Google Cloud Command Line Interface 
[`gcloud CLI](https://cloud.google.com/cli) and 
discuss
how we use it get access to Google Cloud infrastructure, how to enable the necessary
services, how to provide proper roles, how to use service account and how to
build a training
Docker image.

## Setup Global Variables {id="global_variables"}

We establish global shell variables for the project and place them
into `env.sh`:

```Bash
# Google owner account

USER_ACCOUNT=username@domain.com
# Billing
export PROJECT_ID=ray-whisper

# Infrastructure location
export REGION=us-central1

# AI Training Service Account
export SERVICE_ACCOUNT_NAME='ray-ai-training'
export SERVICE_ACCOUNT="${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

# Training artifacts: training docker build
export REPO="${PROJECT}-tune"
export TRAIN_IMAGE="${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/train"

# Cloud Storage
salt=$(echo $RANDOM | md5sum | head -c3)
export BUCKET_NAME="${PROJECT_ID}-${salt}"
export BUCKET_URI="gs://${BUCKET_NAME}"
```

The strategy would be then to start the virtual environment as 
previously described,
source the env variables and then start jupyter lab. Here, our virtual environment
is named after the `$PROJECT_ID`
```Bash
pyenv activate ray-whisper
. ./env.sh
jupyter lab
```
We do it in this order to allow the virtual environment to set up, 
then we declare the shell globals. This way jupyter inherits the shell
configuration.

## Install GCP `gcloud` command-line interface

Google Cloud CLI 
[`gcloud`](https://cloud.google.com/sdk/gcloud) 
is used to create and manage Google Cloud resources and services 
directly on the command line or via scripts. 
By default, the gcloud CLI installs commands that are at the General Availability 
level, considered fully stable and available for production use.
Additional functionality is available
The recommended installation steps
for different platforms can be found 
[here](https://cloud.google.com/sdk/docs/install).

> **Important**
>
> 1. For MacOS it is imperative to make a distinction between 
> Intel CPUs and Apple M1 Silicon because the two have different tar.gz sources.
> 
> 2. It is possible to install `gcloud` using its own python version 3.11. It
> may be a good idea to allow `gcloud` to operate using its own python environment, 
> without any reference to or cross-pollination with `pyenv`. 
{style="note"}

If you use a MacBook Air with Apple M1 Silicon, then you can
download the relevant `arm` cli as follows:

```Bash
$ wget https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-darwin-arm.tar.gz
--2024-07-11 15:37:49--  https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-darwin-arm.tar.gz
Resolving dl.google.com (dl.google.com)... 142.251.46.174, 2607:f8b0:4005:812::200e
Connecting to dl.google.com (dl.google.com)|142.251.46.174|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 52249141 (50M) [application/gzip]
Saving to: ‘google-cloud-cli-darwin-arm.tar.gz’

google-cloud-cli-da 100%[===================>]  49.83M  25.0MB/s    in 2.0s

2024-07-11 15:37:52 (25.0 MB/s) - ‘google-cloud-cli-darwin-arm.tar.gz’ saved [52249141/52249141]
```
{collapsible="true" collapsed-title="wget https://gl.google.com/.../google-cloud-cli-darwin-arm.tar.gz"}

Unpack the gzipped tape-archive with

```Bash
$ tar xvzf google-cloud-cli-darwin-arm.tar.gz
x google-cloud-sdk/.install/.download/
x google-cloud-sdk/.install/bq-nix.manifest
x google-cloud-sdk/.install/bq-nix.snapshot.json
x google-cloud-sdk/.install/bq.manifest
x google-cloud-sdk/.install/bq.snapshot.json
x google-cloud-sdk/.install/core-nix.manifest
x google-cloud-sdk/.install/core-nix.snapshot.json
x google-cloud-sdk/.install/core.manifest
x google-cloud-sdk/.install/core.snapshot.json
x google-cloud-sdk/.install/gcloud-crc32c-darwin-arm.manifest
x google-cloud-sdk/.install/gcloud-crc32c-darwin-arm.snapshot.json
x google-cloud-sdk/.install/gcloud-crc32c.manifest
x google-cloud-sdk/.install/gcloud-crc32c.snapshot.json
x google-cloud-sdk/.install/gcloud-deps.manifest
x google-cloud-sdk/.install/gcloud-deps.snapshot.json
x google-cloud-sdk/.install/gcloud.manifest
x google-cloud-sdk/.install/gcloud.snapshot.json
x google-cloud-sdk/.install/gsutil-nix.manifest
x google-cloud-sdk/.install/gsutil-nix.snapshot.json
x google-cloud-sdk/.install/gsutil.manifest
x google-cloud-sdk/.install/gsutil.snapshot.json
...
~ 22k lines
...
x google-cloud-sdk/platform/gsutil/third_party/urllib3/test/with_dummyserver/test_poolmanager.py
x google-cloud-sdk/platform/gsutil/third_party/urllib3/test/with_dummyserver/test_proxy_poolmanager.py
x google-cloud-sdk/platform/gsutil/third_party/urllib3/test/with_dummyserver/test_socketlevel.py
x google-cloud-sdk/properties
x google-cloud-sdk/rpm/mapping/command_mapping.yaml
x google-cloud-sdk/rpm/mapping/component_mapping.yaml
```
{collapsible="true" collapsed-title="tar xvzf google-cloud-cli-darwin-arm.tar.gz"}

Change to the directory with unpacked code and follow the instructions from the installation
script:

```Bash
$ cd google-cloud-sdk
                        
$ bash install.sh
Welcome to the Google Cloud CLI!

To help improve the quality of this product, we collect anonymized usage data
and anonymized stacktraces when crashes are encountered; additional information
is available at <https://cloud.google.com/sdk/usage-statistics>. This data is
handled in accordance with our privacy policy
<https://cloud.google.com/terms/cloud-privacy-notice>. You may choose to opt in this
collection now (by choosing 'Y' at the below prompt), or at any time in the
future by running the following command:

    gcloud config set disable_usage_reporting false

Do you want to help improve the Google Cloud CLI (y/N)? N (sorry Google, I don't wanna overburden you with data)

Your current Google Cloud CLI version is: 483.0.0
The latest available version is: 483.0.0

┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                    Components                                                   │
├───────────────┬──────────────────────────────────────────────────────┬──────────────────────────────┬───────────┤
│     Status    │                         Name                         │              ID              │    Size   │
├───────────────┼──────────────────────────────────────────────────────┼──────────────────────────────┼───────────┤
│ Not Installed │ App Engine Go Extensions                             │ app-engine-go                │   4.7 MiB │
│ Not Installed │ Appctl                                               │ appctl                       │  18.5 MiB │
│ Not Installed │ Artifact Registry Go Module Package Helper           │ package-go-module            │   < 1 MiB │
│ Not Installed │ Cloud Bigtable Command Line Tool                     │ cbt                          │  17.8 MiB │
│ Not Installed │ Cloud Bigtable Emulator                              │ bigtable                     │   7.3 MiB │
│ Not Installed │ Cloud Datastore Emulator                             │ cloud-datastore-emulator     │  36.2 MiB │
│ Not Installed │ Cloud Firestore Emulator                             │ cloud-firestore-emulator     │  45.2 MiB │
│ Not Installed │ Cloud Pub/Sub Emulator                               │ pubsub-emulator              │  63.7 MiB │
│ Not Installed │ Cloud Run Proxy                                      │ cloud-run-proxy              │  11.7 MiB │
│ Not Installed │ Cloud SQL Proxy v2                                   │ cloud-sql-proxy              │  13.8 MiB │
│ Not Installed │ Google Container Registry's Docker credential helper │ docker-credential-gcr        │   2.2 MiB │
│ Not Installed │ Kustomize                                            │ kustomize                    │   7.6 MiB │
│ Not Installed │ Log Streaming                                        │ log-streaming                │  12.3 MiB │
│ Not Installed │ Minikube                                             │ minikube                     │  36.4 MiB │
│ Not Installed │ Nomos CLI                                            │ nomos                        │  30.1 MiB │
│ Not Installed │ On-Demand Scanning API extraction helper             │ local-extract                │  14.1 MiB │
│ Not Installed │ Skaffold                                             │ skaffold                     │  24.8 MiB │
│ Not Installed │ Terraform Tools                                      │ terraform-tools              │  66.4 MiB │
│ Not Installed │ anthos-auth                                          │ anthos-auth                  │  21.9 MiB │
│ Not Installed │ config-connector                                     │ config-connector             │  92.7 MiB │
│ Not Installed │ enterprise-certificate-proxy                         │ enterprise-certificate-proxy │   9.1 MiB │
│ Not Installed │ gcloud Alpha Commands                                │ alpha                        │   < 1 MiB │
│ Not Installed │ gcloud Beta Commands                                 │ beta                         │   < 1 MiB │
│ Not Installed │ gcloud app Java Extensions                           │ app-engine-java              │ 127.5 MiB │
│ Not Installed │ gcloud app PHP Extensions                            │ app-engine-php               │  21.9 MiB │
│ Not Installed │ gcloud app Python Extensions                         │ app-engine-python            │   5.0 MiB │
│ Not Installed │ gcloud app Python Extensions (Extra Libraries)       │ app-engine-python-extras     │   < 1 MiB │
│ Not Installed │ gke-gcloud-auth-plugin                               │ gke-gcloud-auth-plugin       │   4.2 MiB │
│ Not Installed │ istioctl                                             │ istioctl                     │  25.7 MiB │
│ Not Installed │ kpt                                                  │ kpt                          │  14.8 MiB │
│ Not Installed │ kubectl                                              │ kubectl                      │   < 1 MiB │
│ Not Installed │ kubectl-oidc                                         │ kubectl-oidc                 │  21.9 MiB │
│ Not Installed │ pkg                                                  │ pkg                          │           │
│ Installed     │ BigQuery Command Line Tool                           │ bq                           │   1.7 MiB │
│ Installed     │ Cloud Storage Command Line Tool                      │ gsutil                       │  11.3 MiB │
│ Installed     │ Google Cloud CLI Core Libraries                      │ core                         │  18.9 MiB │
│ Installed     │ Google Cloud CRC32C Hash Tool                        │ gcloud-crc32c                │   1.2 MiB │
└───────────────┴──────────────────────────────────────────────────────┴──────────────────────────────┴───────────┘
To install or remove components at your current SDK version [483.0.0], run:
  $ gcloud components install COMPONENT_ID
  $ gcloud components remove COMPONENT_ID

To update your SDK installation to the latest version [483.0.0], run:
  $ gcloud components update


Modify profile to update your $PATH and enable shell command completion?

Do you want to continue (Y/n)? Y

The Google Cloud SDK installer will now prompt you to update an rc file to bring
 the Google Cloud CLIs into your environment.

Enter a path to an rc file to update, or leave blank to use
[${HOME}/.bash_profile]:

```
{collapsible="true" collapsed-title="cd google-cloud-sdk; bash install.sh"}

We untarred the `gcloud` source to `${HOME}/var`, purely by choice. We need to 
place `${HOME}/var` into shell path in order to use `gcloud`.
The exectuted installation script will append an `if-clause` to `.bash_profile`. 
This addition updates the shell path:
```Bash
# The next line updates PATH for the Google Cloud SDK.
if [ -f ${HOME}/var/google-cloud-sdk/path.bash.inc ]; then
  . ${HOME}/var/google-cloud-sdk/path.bash.inc'
fi
```
The bash code that checks for existence of `path.bash.inc` in reality shows
expanded `${HOME}` and the whole path is enclosed with single quotes. 
The code here is slightly edited and provided for generality.
For extra caution, the path(s) can be placed in double quotes.

It is also advantageous to add the following lines to `.bash_profile`
to enable command completion in bash:
```Bash
# Append to $HOME/.bash_init
# The next line enables shell command completion for gcloud.
if [ -f ${HOME}/var/google-cloud-sdk/completion.bash.inc ]; then
  . ${HOME}/var/google-cloud-sdk/completion.bash.inc'
fi
```
Command completion then can be used by hitting `tab` on the command line.

Here is the output on the screen, with a note that for these changes to take effect, 
a new shell should be started:
```Bash
Backing up [${HOME}/.bash_profile] to [${HOME}/.bash_profile.backup].
[${HOME}/.bash_profile] has been updated.

==> Start a new shell for the changes to take effect.


Google Cloud CLI works best with Python 3.11 and certain modules.

Download and run Python 3.11 installer? (Y/n)?
```
Here you can decouple your dev env python 3.10 managed by pyenv from 
`gcloud` python 3.11 by pressing `Y`. That sounds like a safe way to proceed to 
avoid dependency hell on updates.

To complete the installation, follow the prompts. When done, start a new shell 
for changes to take effect.

## Get Started with GCP `gcloud` CLI

With the `gcloud` CLI installed, preferably within its own python 3.11 environment, you are 
ready to start issuing commands.

### Initialize the `gcloud` CLI
 
Google 
[recommends](https://cloud.google.com/sdk/docs/initializing) performing the initial setup 
by running

```Bash
gcloud init
```
Following instructions, you will be able to create or select a configuration 
if prompted, complete the authorization step when prompted, and choose or create a 
current Google Cloud project if prompted.

If there is a single available project, `gcloud init` selects it. Otherwise, you can select
a project from the list of projects for which you have **Owner**, **Editor** or **Viewer** 
permissions. If there is nothing, you can go to the console (the GCP gui in the browser)
and figure out how to create a project.

> **Note**
> 
> If you choose to create a project, you also need to 
> [enable billing](https://cloud.google.com/billing/docs/how-to/modify-project)
> on the project to be able to use GCP services. Welcome to FinOps!
{style="note"}

You may also choose a default Compute Engine zone, if prompted. If not, we will 
specify it up later.

To verify what just happened, issue
```Bash
gcloud config list
```
and if there is anything you do not like, exit with Ctrl+C and redo `gcloud init`.

### Configure the `gcloud` CLI

In order to get access to GCP, we need to have a project for which billing is 
enabled. If there are any issues with `gcloud init`, the following commands
can be issued to set the project and owner account.
```Bash
gcloud config set project ${PROJECT_ID}
gcloud config set account ${USER_ACCOUNT}
```
The owner can then impersonate a service account for specific purpose.

To authenticate with GCP, issue:
```Bash
gcloud auth application-default login
```
Following the prompts places credentials to 
`${HOME}/.config/gcloud/application_default_credentials.json`. These credentials
are then used be used by any library that requests Application Default Credentials (ADC).
In this case, we use python SDK.
We need to set up a project which i

### Authorize the `gcloud` CLI


In order to access Google Cloud, you have to authorize the Google Cloud CLI. 

In dev scenario, the flow is to authorize the 
There are two types of accounts that can be authorized with GCP: a user account and
a service account.

A [`user account`](https://cloud.google.com/docs/authentication#user-accounts)
is a Google Cloud account that allows end users to authenticate to applications.
For example, if you use gmail and decide to start using GCP, your user account is 
your google account. For most common use cases, especially interactively using 
the gcloud CLI, using a user account is best practice. A user account is also 
recommended if using the gcloud CLI from the command line, or if using/writing 
scripts on a single machine.

A [service account](https://cloud.google.com/docs/authentication#service-accounts) 
is a Google Cloud account associated with a Google Cloud project and 
not a specific user. GCP provides a number of built-in service accounts when using 
various GCP services. Actually, those service cannot do anything without having
service accounts (that is agents) attached to them.

A service account is recommended to run gcloud CLI scripts on multiple machines.
Service accounts are accounts that do not represent a human user. They provide a way 
to manage authentication and authorization when a human is not directly involved, 
such as when an application needs to access Google Cloud resources. Service accounts 
are managed by 
[IAM](https://cloud.google.com/iam/docs/apis).

Both user accounts and service accounts are considered principals. 
A 
[principal](https://cloud.google.com/docs/authentication#principal)
is an identity that can be granted access to resources. Whether to use a user account 
or a service account to authenticate depends ona a use case. One can also use both, 
each at different stages of the project or different dev environments.

In the previous section we described `gcloud init`. That command authorizes access 
and performs other common setup steps.

The following command authorizes the user account access only:
```Bash
gcloud auth init
```
The details of the steps that follow are 
[available](https://cloud.google.com/sdk/docs/authorizing#auth-login)
in documentation.

Our case, fine-tuning a foundational model, requires running code in a distributed 
environment on Google Cloud. According to the 
[provided](https://cloud.google.com/docs/authentication#auth-decision-tree) 
decision tree
![](service_account_decision_tree.png){width="650" style="block"}
we proceed along the following path:

1. Are you running code in a single-user development environment? Yes
2. Does your use case require a service account? Yes

> Note: How do we know that our use case requires a service account? 
> 
> Well, we could just use user credentials for a quick and dirty project 
> completion. However, using a service account is the recommended option.

Then we need to 
[impersonate](https://cloud.google.com/docs/authentication/use-service-account-impersonation#gcloud-config)
a service account with user credentials.

> Note:
> If this was a bigger project where we run code in a multi-user development 
> environment, then the decision flow would be:
> 1. Are you running code in a single-user development environment? No
> 2. Are you running code in Google Cloud? Yes
> 3. Are you running containers in Google Kubernetes Engine or GKE Enterprise? No

In any case, we need to 
[create](https://cloud.google.com/iam/docs/service-accounts-create#creating) 
a service account for AI training. Once we have created the service account,
we can 
[attach](https://cloud.google.com/iam/docs/attach-service-accounts#attaching-to-resources)
it to a resource (Vertex AI) by granting the service account required roles. 
That way the service account can access the appropriate resources. 
Eventually, we can
[delete](https://cloud.google.com/iam/docs/service-accounts-delete-undelete#deleting)
the service account. Or not.

### **How exactly does `path.bash.inc` modify the PATH?** {collapsible="true"} 

Here is the content of `path.bash.inc`:

```Bash
script_link="$( command readlink "$BASH_SOURCE" )" || script_link="$BASH_SOURCE"
apparent_sdk_dir="${script_link%/*}"
if [ "$apparent_sdk_dir" == "$script_link" ]; then
  apparent_sdk_dir=.
fi
sdk_dir="$( command cd -P "$apparent_sdk_dir" > /dev/null && command pwd -P )"
bin_path="$sdk_dir/bin"
if [[ ":${PATH}:" != *":${bin_path}:"* ]]; then
  export PATH=$bin_path:$PATH
fi
```
Is that some cool shit or what? Pass this script to ChatGPT and see what it returns.

### **More trivia on GCP Service Accounts** {collapsible="true"}

Google recommends that service accounts be kept instead of deleted.
The preferred method of handling unused service accounts is to 
[disable](https://cloud.google.com/iam/docs/service-accounts-disable-enable#disabling) 
them. Keep in mind that service accounts count towards the service account quota, but deleted
service accounts do not.

If you delete a service account and then create a new service account with the same name, 
the new service account is treated as a separate identity; it does not inherit the 
roles granted to the deleted service account. In contrast, when you delete a 
service account, then undelete it, the service account's identity does not
change, and the service account retains its roles.

When a service account is deleted, its role bindings are not immediately removed; 
they are automatically purged from the system after a maximum of 60 days. Until 
that time, the service account appears in role bindings with a `deleted:` prefix 
and a `?uid=NUMERIC_ID` suffix, where NUMERIC_ID is a unique numeric ID for the 
service account.

## Setup a Project in GCP

GCP bills through Organizations, Folders or Projects. Organization can be a company, 
for example, which can further be divided into folders (divisions: ML, DevOps, 
Sales, etc.), and then into projects. If you use GCP through your personal `@gmail.com` 
account, you are unlikely to belong to an organization. 

#### **Welcome to FinOps** {collapsible="true"}

Not belonging to an organization may be a problem if you require any type of 
support beyond 
[Community Support](https://console.cloud.google.com/support/community?project=ray-whisper).
As a free agent and a member of `Community Support` you have at your disposal access to
[Google Cloud Community](https://www.googlecloudcommunity.com/gc/Google-Cloud/ct-p/google-cloud),
Stack Overflow, 
[Server Fault](https://serverfault.com/questions/tagged/google-cloud-platform)
or
[Google Developers](https://cloud.google.com/support/docs/issue-trackers).

Without an organization, even if you wanted to pay for tech support, you could not. 
The basic, lowest tier of Google tech support
[costs](https://cloud.google.com/support)
$29/month plus 3% of the bill, but again, without an organization meting out 
a resource quota for you, the only option is going back in the infinite loop of 
`Community Support`. Doable, but what's your time worth?

If you do not belong to an organization, billing is project-based. There is a limited number
of projects one can start without GCP approval. In order to start using `gcloud` to 
interact and work with GCP, Google needs a payment method on file. Welcome to FinOps 101.

With a brand new GCP account, Google offers a $300 credit. That credit comes at a cost - 
you cannot provision any GPUs! You may be able to do some basic things, 
like kicking the tires, but that's it.

In summary, you need a Project ID to get access to GCP infrastructure, which you need to tie
to a payment account.
[No tickie, no laundry!](https://www.youtube.com/watch?v=97TW1Zm1uMI&t=15s)

If you are a member of an organization that has established relationship with GCP, then 
it is up to your org to assign you the quota for what and how much you can use. In either 
case, good luck! And welcome to FinOps 102 with extra negotiation tactics.

#### **Welcome to SecOps** {collapsible="true"}
Now that you have a project through your organization, our you were able to muster a project
on your own without an organization, the next stop is Identity Access Management (IAM).

[IAM](https://cloud.google.com/security/products/iam) 
provides:
1. Enterprise-grade access control
2. Smart access control
3. Granular context-aware access
4. Streamlined compliance with a built-in audit trail
5. Enterprise identity made easy
6. Workforce Identity Federation
7. Organization Policies
8. Single access control interface
9. Fine-grained control
10. Automated access control recommendations
11. Context-aware access
12. Flexible roles
13. Web, programmatic, and command-line access
14. Support for Cloud Identity
15. Built-in audit trail

And the best of all, it is free of charge! Fuckin' awesome! More reading to go through.

All that means, even though you may the quota to use a cloud resource, you still 
need the right to access it. 

### Setup a project in GCP with `gcloud`

There are multiple ways of creating a project in GCP: in the console (or GCP GUI), 
by issuing a gcloud command, or by issuing an API call.

There is no challenge in clicking through the user interface to create
a project, if you haven't already when you initialized your account. 
Considering that building jsons in preparation for sending naked API calls 
that require authentication require more work then desired, the happy medium is to 
use `gcloud`:

```Bash
export PROJECT_ID=ray-whisper
gcloud projects create $PROJECT_ID
```



## Install Vertex AI Python Package(s)

Google 
[Vertex AI](https://cloud.google.com/vertex-ai/docs/start/introduction-unified-platform)
is an integrated suite of machine learning tools and services for building, using and 
deploying ML models.

[Vertex AI](https://cloud.google.com/vertex-ai/docs)
offers several developer tools:
1. Python
[software development kit](https://cloud.google.com/vertex-ai/docs/python-sdk/use-vertex-ai-python-sdk) 
(SDK);
2. Vertex AI
[Workbench](https://cloud.google.com/vertex-ai/docs/workbench/introduction) - a Jupyter 
notebook-based dev environment run on GCP instances.
3. Terraform infrastructure-as-code (IaC) to provision resources and permissions for multiple
Google Cloud services, including 
[terraform for Vertex AI](https://cloud.google.com/vertex-ai/docs/start/use-terraform-vertex-ai).

An advanced version of Vertex AI Workbench is 
[Google Colab](https://colab.research.google.com/). 
An even more advanced version of Google Colab is 
[Colab Enterprise](https://cloud.google.com/colab/docs/introduction).

We start by installing the Vertex AI python SDK, `google-cloud-aiplatform` in our 
dev environment. We need the functionality of
[Ray Client](https://docs.ray.io/en/latest/cluster/running-applications/job-submission/ray-client.html)
from within python, either in a local Jupyter notebook, in Vertex AI Workbench,
in Google Colab or in Colab Enterprise. To 
[install](https://cloud.google.com/vertex-ai/docs/open-source/ray-on-vertex-ai/set-up) 
Vertex AI SDK narrowly focused
to work with Ray, we use the following pip command:
```Bash
pip install google-cloud-aiplatform[ray]
```

## Enable Vertex AI Services

When working with Vertex AI in GCP cloud console on a project for the first time,
the user typically gets prompted to enable all Vertex AI-related APIs. 
We can achieve the same thing with `gcloud` by carefully searching for 
the exact services' names, stealing the names from other projects, or 
by painful trial and error.

For AI training on Ray, with `gcloud` we enable the following 11 APIs:

```Bash
gcloud services enable \
  compute.googleapis.com \
  iam.googleapis.com \
  iamcredentials.googleapis.com \
  monitoring.googleapis.com \
  logging.googleapis.com \
  notebooks.googleapis.com \
  aiplatform.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  container.googleapis.com \
  cloudresourcemanager.googleapis.com
```

If one of the services is missing, you are likely to get blocked. 
Without `compute`, there are no virtual machines that can be started. 
Without `iam` and `iam`-related APIs we are unable to authorize services. 
Arguably, we can survive
without `monitoring` and `logging`, but only for the first few minutes 
before things have a chance to go wrong. 
We do wanna work in `notebooks` (Workbench, Colab, laptop).
Since this project is all about AI training, try dropping `aiplatform` to see how much
mileage you can get. To build a custom training container (who needs a plain
vanilla container, anyway?), we require `artifactregistry`, `cloudbild` and 
`container`. And the last one is the kicker - `cloudresourcemanager`: try launching 
a cluster without that one.

Under the prevalent SecOps security policies of the
[`least privilege`](https://cloud.google.com/iam/docs/using-iam-securely#least_privilege),
unless you **ask**, you ain't getting it. If there are additional services that are 
required, there is ample 
[documentation](https://cloud.google.com/products) 
to **seek** through. There is a also little trick with `gcloud`
to list what services are available, and there are quite a few of those
```Bash
$ gcloud services list --available | wc -l
    4975
```
Luckily not all of them are enabled.
```Bash
$ gcloud services list --enabled | wc -l
      44
```
This number is bigger than 11, which may be for several reasons. Some of the
services get enabled by default through initialization, some other services 
get enabled after trying everything you can Google that sounds reasonable
in response to annoying exceptions in your way of using Google Cloud services.
That is, when you persistently *knock* at the door trying everything you can think 
of to open, but it just wouldn't budge.

A prime example could be cloud storage. Because we need to keep model artifacts and
data on Google Cloud Storage, we may need to enable one or all three of the following:
```Bash
storage-api.googleapis.com           Google Cloud Storage JSON API
storage-component.googleapis.com     Cloud Storage
storage.googleapis.com               Cloud Storage API
```
How do we do that? It is as Biblical as in 
[Mathew 7:7](https://www.bible.com/bible/111/MAT.7.NIV).
Because just when you think you are done with enabling all possible services, different
roles require different privileges. In that case `Mathew 7:7` offers great support
that has been known for at least two millennia.

## Create a Google Cloud Storage Bucket

When we installed `gcloud`, we also installed `gsutil` - a Python application for access to 
Google Cloud Storage from the command line. We use `gsutil` to create a bucket:

```Bash
gsutil mb -l ${REGION} ${BUCKET_NAME}
```
We confine the bucket to a `${REGION}` because we do not need the data globally available.
This way we minimize the cost (FinOps) and latency - welcome to MLOps!

## AI Training Service Account {id="service_account"}

We have already established that for this work Google recommends using a service account.
This service account will be used for Ray cluster management and for AI training on the 
provisioned architecture.

> Note
>
> Vertex AI automatically creates several 
> [service *agents*](https://cloud.google.com/iam/docs/service-agents)
> (rather than accounts)
> for `aiplatform.googleapis.com`. 
> Few of them are:
> 1. [AI Platform Service Agent](https://cloud.google.com/iam/docs/service-agents#ai-platform-service-agent)
> with role 
> [`roles/aiplatform.serviceAgent`](https://cloud.google.com/iam/docs/understanding-roles#aiplatform.serviceAgent).
> and email
> `service-${PROJECT_NUMBER}@gcp-sa-aiplatform.iam.gserviceaccount.com`.
> 2. [AI Platform Tuning Service Agent](https://cloud.google.com/iam/docs/service-agents#ai-platform-fine-tuning-service-agent)
> with role 
> [`roles/aiplatform.tuningServiceAgent`](https://cloud.google.com/iam/docs/understanding-roles#aiplatform.tuningServiceAgent)
> and email
> `service-${PROJECT_NUMBER}@gcp-sa-aiplatform-ft.iam.gserviceaccount.com`.
> ...
> 
> Service agents allow Google Cloud services to access resources. 
> Some service agent roles contain very powerful permissions, but there is not 
> one size that
> fits all. It may be then appealing to assign pre-baked service agents to 
> custom-created principals.
> However, Google 
> [warns](https://cloud.google.com/iam/docs/service-agents)
> against granting service agent roles to any principals except service agents.
> Changing agent roles is also a [bad idea](https://youtu.be/xXk1YlkKW_k), 
> because some services may stop working correctly. 

### Create a Service Account 

We use the 
[already defined](#global_variables)
shell env variables with the following `gcloud` command:

```Bash
gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME  \
    --description="A custom service account for ${PROJECT_ID} Vertex AI training" \
    --display-name="${PROJECT_ID} Vertex AI Training"
```
A more thorough description on how to create a service account can be found 
in the Google Cloud
[documentation](https://cloud.google.com/iam/docs/service-accounts-create#creating).

### Grant necessary roles to the Service Account

Next, we need to assign roles to the service account. Once the service account has been
created, Google Cloud provides an email for it constructed from the `${SERVICE_ACCOUNT_NAME}`
and `${PROJECT_ID}`. We replicate this nomenclature from 
[global env variables](#global_variables) 

```Bash
export SERVICE_ACCOUNT="${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
```

There are two 
[recommended roles](https://github.com/GoogleCloudPlatform/vertex-ai-samples/blob/main/notebooks/community/model_garden/model_garden_gemma_fine_tuning_batch_deployment_on_rov.ipynb),
`aiplatform.user` and `storage.admin`, that we bind to the AI training service account:

```Bash
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member=serviceAccount:${SERVICE_ACCOUNT} \
    --role="roles/aiplatform.user"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
   --member=serviceAccount:${SERVICE_ACCOUNT} \
   --role="roles/storage.admin"
```

### **More On Service Account IAM Roles** {collapsible="true"}
These roles were recommended in a community notebook
found in 
[`GoogleCloudPlatform/vertex-ai-samples`](https://github.com/GoogleCloudPlatform/vertex-ai-samples) repo.

The `aiplatform.user` role is insufficient because it does not allow spawning of 
a Ray cluster. Using Google Cloud python sdk, we start a cluster with
[`vertex_ray.create_ray_cluster`]()

In that case, we deviate from the `minimum privilege`
security policy, and promote the service account to
`aiplatform.admin`. The only instance when we needed to upgrade the
aiplatform role was when the project quota is at infancy (low 
number of CPUs, 0 or 1 GPUs), and some of the services would intermittently
work. Even in that case, whatever works in dev due to the `admin` role,
should most likely be scraped and made worksure it works inder 

## Build a Docker Image for AI Training

To build an image we need to enable the correct service (done) and we need
to authenticate it.

### Authenticate with Docker

Access to Docker registry needs authentication. Google 
[recommends](https://cloud.google.com/artifact-registry/docs/docker/authentication)
using a service account to access Artifact Registry Docker repositories. 
The available 
[authentication methods](https://cloud.google.com/artifact-registry/docs/docker/authentication#methods)
are 
* [`gcloud CLI credential helper`](https://cloud.google.com/artifact-registry/docs/docker/authentication#gcloud-helper) - 
  this is the simplest method to configure Artifact Registry credentials for use 
  with Docker directly in  `gcloud` CLI.  
* [`Standalone Docker credential helper`](https://cloud.google.com/artifact-registry/docs/docker/authentication#standalone-helper) - 
  suitable when `gcloud` is not available.
* [`Access token`](https://cloud.google.com/artifact-registry/docs/docker/authentication#token) - 
   generate a short-lived access token for service account and then use the token
   for password authentication. 
* [`Service account key`](https://cloud.google.com/artifact-registry/docs/docker/authentication#json-key) -
  a long-lived, user-managed key-pair that can be uses as a credential for a service account, 
  the least secure option.

The first one is the simplest, the second one
can be used in the absence of `gcloud` CLI. 
`Access token`s are short-lived for a service account access 
and are valid for only 60 minutes. They are safer than `service
account key`.

Google recommends using
[`access token`](https://cloud.google.com/artifact-registry/docs/docker/authentication#token)
to authenticate with Artifact Registry.

The service account needs to have 
[permissions](https://cloud.google.com/artifact-registry/docs/access-control#permissions) 
to access Artifact Registry. Since we would like to be able to create and manage
repositories and artifacts on `pkg.dev` domain, we prefer the
[`roles/artifactregistry.admin`](https://cloud.google.com/artifact-registry/docs/access-control#roles)
role. However, that may not be necessary.
Although not explained in the documentation, 
`roles/aiplatform.user` may have access to
Artifact Registry through `aiplatform.artifacts.*` permissions: 
* `aiplatform.artifacts.create` 
* `aiplatform.artifacts.delete` 
* `aiplatform.artifacts.get` 
* `aiplatform.artifacts.list` 
* `aiplatform.artifacts.update`
The 
[documentation does mention](https://cloud.google.com/vertex-ai/docs/open-source/ray-on-vertex-ai/create-cluster#custom-image)
that
"If you are using an Artifact Registry image from the same Google Cloud project where
you're using Vertex AI, then there's no further need to configure permissions."

In order to generate a short-lived access token for the service account and authenticate,
we actually 
[impersonate](https://cloud.google.com/docs/authentication/use-service-account-impersonation) 
the **service account** from the **user account**.

To obtain your user account, use the following command
```Bash

gcloud auth list
                       Credentialed Accounts
ACTIVE  ACCOUNT
*       username@domain.com
        ray-whisper-ai-training@ray-whisper.iam.gserviceaccount.com

To set the active account, run:
    $ gcloud config set account `ACCOUNT`
```
Your *user account* is the one prepended with an asterisk `*`. 
You *user account* needs to have the `Service Account Token Creator` role
[`roles/iam.serviceAccountTokenCreator`](https://cloud.google.com/iam/docs/understanding-roles#iam.serviceAccountTokenCreator)
enabled which contains 
`iam.serviceAccounts.getAccessToken` permission, which is required to 
be able to generate auth tokens.
You can add this policy to your user account with
```Bash
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
   --member=user:username@domain.com \
   --role="roles/iam.serviceAccountTokenCreator"
```

> Note: 
> If the IAM situation worsens, let your service 
> account impersonate your service account. In that case, replaced the key `user`, 
> with `serviceAccount`. Use at your own risk. 

When the principal, in this case the AI training service account, does not have the 
permissions to accomplish a task, or when using a service account in a development 
environment, we can use service account impersonation.

Service account impersonation starts with an authenticated principal (i.e. your
*user account*) request for short-lived credentials for the service account that has 
the authorization that our use case requires. 

Service account impersonation is more secure than using a service account key 
because service account impersonation requires a prior authenticated identity, 
and the credentials that are created by using impersonation do not persist. 
In comparison, authenticating with a service account key requires no prior 
authentication, and the persistent key is a high risk credential if exposed.

We can then generate an access token for the service account and 
[authenticate](https://cloud.google.com/artifact-registry/docs/docker/authentication#token)
it with
the following `gcloud` command:

```Bash
$ gcloud auth print-access-token \
    --impersonate-service-account $SERVICE_ACCOUNT | docker login \
    -u oauth2accesstoken \
    --password-stdin https://${REGION}-docker.pkg.dev
WARNING: This command is using service account impersonation. All API calls will be executed as [ray-whisper-ai-training@ray-whisper.iam.gserviceaccount.com].
Login Succeeded
```
The access token is valid for 60 minutes. You can change the token lifetime by issuing 
the `--lifetime=LIFETIME` flag, which by default is 3600. You can set this flag to under
an hour without any problem, but in order to extend it to up to 12 hours, you need to 
have the org policy constraint `constraints/iam.allowServiceAccountCredentialLifetimeExtension`
enabled. Because I am happy to avoid any additional security policy that stands in 
my way to AI training, I figure I am quite happy with 60 minutes, at least for now.

#### More on SecOps: Security risks with service accounts {collapsible="true"}

Service accounts do not have passwords. Instead, service
accounts use RSA keys for authentication. Having access to the private key
is similar to knowing a user's password.
Anyone with access to a valid private key for a service account will be able 
to access resources through the service account. 

> Note: The lifecycle of the service account key's access is independent of the 
> lifecycle of the user who has downloaded the key.

In dev scenarios having to worry about an expiring resource may be an
unnecessary distraction. 
[`Service account key`](https://cloud.google.com/artifact-registry/docs/docker/authentication#json-key)
provides access to GCP cloud assets without time limitation. Service account keys can 
become security risk if not managed carefully. The main 
[threats](https://cloud.google.com/iam/docs/best-practices-for-managing-service-account-keys)
related to using 
service account keys are: 
* **Credential leakage** - keys end up being committed to a public repo.
* **Privilege escalation** - with a poorly configured key, malicious users can escalate their privileges. 
* **Informal disclosure** - may inadvertently disclose confidential metadata.
* **Non-repudiation** - a whodunit scenario where malicious users can hide their identity.

Google provides 
[`iam.disableServiceAccountKeyCreation`](https://cloud.google.com/resource-manager/docs/secure-by-default-organizations#organization_policies_enforced_on_organization_resources)
organization policy constraint which prevents users from creating persistent keys 
for service accounts. For organizations created after 2024-05-32, this constraint
is 
[enforced by default](https://cloud.google.com/resource-manager/docs/organization-policy/restricting-service-accounts#disable_service_account_key_creation).
It seems there is not an easy way to avoid security policy, there is always a tradeoff 
that stands in the way of AI training.

Can all these complications be avoided with 
[`gcloud auth configure-docker`](https://cloud.google.com/sdk/gcloud/reference/auth/configure-docker)?
 ```Bash
gcloud auth configure-docker ${REGION}-docker.pkg.dev
```
Yes, but that requires using the simplest authentication method with
`gcloud CLI credential helper` to 
[authenticate](https://cloud.google.com/artifact-registry/docs/docker/authentication#gcloud-helper)
using a `service account key`, which we know is a big no-no.
This command then modifies `${HOME}/.docker/config.json`.

Solution: use service accounts with short-lived auth tokens.

### Create a Docker image repository

We use the following `gcloud` command to 
[create](https://cloud.google.com/sdk/gcloud/reference/artifacts/repositories/create) 
a new Artifact Registry as a Docker image repository: 
```Bash
gcloud artifacts repositories create $REPO \
  --repository-format=docker \
  --location=${REGION} \
  --description="Whisper fine-tuning"
```
Only the first flag `--repository-format` is required. The 
allowed options for repository format are `apt`, `go`, `kfp`,
`maven`, `npm`, `python` and `yum`.

```Bash
apt
   APT package format.
docker
   Docker build format.
go
   Go module format.
kfp
   KFP package format.
maven
   Maven package format.
npm
   NPM package format.
python
   Python package format.
yum
   YUM package format.
```
{collapsible="true" collapsed-title="Allowed artifact repository format"}

### Build the AI training image

Within the working repo dir, we place two files in a subdir named `image`: 
`Dockerfile` and `requirements.txt`:
```Bash
$ ls -1 build/
Dockerfile
requirements.txt
```

`Dockerfile` contains the following:
```Bash
FROM us-docker.pkg.dev/vertex-ai/training/ray-gpu.2-9.py310:latest
ENV PIP_ROOT_USER_ACTION=ignore
COPY requirements.txt .
RUN pip install -r requirements.txt
```
{collapsible="true" collapsed-title="Dockerfile"}
We build from an image based on Ray version 2.9, python 3.10 and GPUs enabled.

In the container we install python packages from `requirements.txt`:
```Bash
--extra-index-url https://download.pytorch.org/whl/cu118
ipython==8.22.2
torch==2.2.1
torchaudio==2.2.1
ray==2.10.0
ray[data]==2.10.0
ray[train]==2.10.0
datasets==2.17.0
transformers==4.39.0
evaluate==0.4.1
jiwer==3.0.0
accelerate==0.28.0
deepspeed==0.14.0
soundfile==0.12.1
librosa==0.10.0
pyarrow==15.0.2
fsspec==2023.10.0
gcsfs==2023.10.0
etils==1.7.0
```
{collapsible="true" collapsed-title="requirements.txt"}

The content of both files is provided in the 
[Medium article](https://medium.com/google-cloud/whisper-goes-to-wall-street-scaling-speech-to-text-with-ray-on-vertex-ai-part-i-04c8ceb180c0)

The article also suggests using a specific build machine. The 
[options](https://cloud.google.com/sdk/gcloud/reference/builds/submit#--machine-type)
are `e2-highcpu-32`, `e2-highcpu-8`, `e2-medium`, `n1-highcpu-32` and `n1-highcpu-8`.
Whatever you use, with the SeqOps policy in action counts against the CPU quota.
In case your policy allows you max 20 vCPUs, you cannot build the training image with 
a 32-core virtual machine. It is best not to specify the build machine type.

We then
[submit a build](https://cloud.google.com/sdk/gcloud/reference/builds/submit)
using `Cloud Build`, the API for which we have already enabled:
```Bash
pushd build
gcloud builds submit \
  --region=${REGION} \
  --tag=${TRAIN_IMAGE} \
  --timeout=3600
 popd
```
We briefly changed to the `image` directory, started the build from that directory, 
and on completion, went back to our repo root.

It takes about 20 minutes to build the image and the build log is quite extensive.
```Bash
$ gcloud builds submit \
  --region=${REGION} \
  --tag=$TAG
  
Creating temporary archive of 2 file(s) totalling 514 bytes before compression.
Uploading tarball of [.] to [gs://ray-whisper_cloudbuild/source/1719884360.752966-84217a4da65c4998bde9f438bc87865a.tgz]
Created [https://cloudbuild.googleapis.com/v1/projects/ray-whisper/locations/us-central1/builds/ec618557-0a2b-4538-bee8-f06f5b3b9938].
Logs are available at [ https://console.cloud.google.com/cloud-build/builds;region=us-central1/ec618557-0a2b-4538-bee8-f06f5b3b9938?project=702949608356 ].
Waiting for build to complete. Polling interval: 1 second(s).
----------------------------- REMOTE BUILD OUTPUT ------------------------------
starting build "ec618557-0a2b-4538-bee8-f06f5b3b9938"

FETCHSOURCE
Fetching storage object: gs://ray-whisper_cloudbuild/source/1719884360.752966-84217a4da65c4998bde9f438bc87865a.tgz#1719884361439854
Copying gs://ray-whisper_cloudbuild/source/1719884360.752966-84217a4da65c4998bde9f438bc87865a.tgz#1719884361439854...
/ [1 files][  574.0 B/  574.0 B]
Operation completed over 1 objects/574.0 B.
BUILD
Already have build (with digest): gcr.io/cloud-builders/docker
Sending build context to Docker daemon  3.072kB
Step 1/4 : FROM us-docker.pkg.dev/vertex-ai/training/ray-gpu.2-9.py310:latest
latest: Pulling from vertex-ai/training/ray-gpu.2-9.py310
aece8493d397: Pulling fs layer
...
394e1e1055b6: Pulling fs layer
4cda774ad2ec: Waiting
...
93db8dad1ca7: Waiting
5e3b7ee77381: Verifying Checksum
...
cfe505ae48e4: Pull complete
93db8dad1ca7: Pulling fs layer
...
394e1e1055b6: Pulling fs layer
4cda774ad2ec: Waiting
...
cfe505ae48e4: Waiting
93db8dad1ca7: Waiting
5e3b7ee77381: Verifying Checksum
5e3b7ee77381: Download complete
...
394e1e1055b6: Pull complete
Digest: sha256:2c9c988191087e4e8b10fbaaa4d76577626edec9cd50ebc187b333af62cb6984
Status: Downloaded newer build for us-docker.pkg.dev/vertex-ai/training/ray-gpu.2-9.py310:latest
 ---> a090a09093b8
Step 2/4 : ENV PIP_ROOT_USER_ACTION=ignore
 ---> Running in fc16d6525a3d
Removing intermediate container fc16d6525a3d
 ---> 8a9923804616
Step 3/4 : COPY requirements.txt .
 ---> b9092ad6c4f3
Step 4/4 : RUN pip install -r requirements.txt
 ---> Running in b3a8ade1164a
Looking in indexes: https://pypi.org/simple, https://download.pytorch.org/whl/cu118
Collecting ipython==8.22.2 (from -r requirements.txt (line 2))
  Downloading ipython-8.22.2-py3-none-any.whl.metadata (4.8 kB)
...
Building wheels for collected packages: deepspeed
  Building wheel for deepspeed (setup.py): started
  Building wheel for deepspeed (setup.py): finished with status 'done'
  Created wheel for deepspeed: filename=deepspeed-0.14.0-py3-none-any.whl size=1400345 sha256=32abb3fd90e55f3b8affbf724018cde5b2c66db3eed6e669a6b655aa999d5fee
  Stored in directory: /root/.cache/pip/wheels/23/96/24/bab20c3b4e2af15e195b339afaec373eca7072cf90620432e5
Successfully built deepspeed
Installing collected packages: py-cpuinfo, pure-eval, ptyprocess, ninja, mpmath, hjson, xxhash, triton, traitlets, threadpoolctl, sympy, soxr, safetensors, regex, rapidfuzz, pynvml, pyarrow-hotfix, pyarrow, prompt-toolkit, pexpect, parso, nvidia-nvtx-cu11, nvidia-nccl-cu11, nvidia-cusparse-cu11, nvidia-curand-cu11, nvidia-cufft-cu11, nvidia-cuda-runtime-cu11, nvidia-cuda-nvrtc-cu11, nvidia-cuda-cupti-cu11, nvidia-cublas-cu11, MarkupSafe, llvmlite, joblib, fsspec, executing, etils, dill, audioread, asttokens, stack-data, soundfile, scikit-learn, responses, pooch, nvidia-cusolver-cu11, nvidia-cudnn-cu11, numba, multiprocess, matplotlib-inline, jiwer, jinja2, jedi, huggingface-hub, torch, tokenizers, librosa, ipython, transformers, torchaudio, ray, deepspeed, datasets, accelerate, evaluate, gcsfs
  Attempting uninstall: pyarrow
    Found existing installation: pyarrow 15.0.0
    Uninstalling pyarrow-15.0.0:
      Successfully uninstalled pyarrow-15.0.0
  Attempting uninstall: fsspec
    Found existing installation: fsspec 2024.2.0
    Uninstalling fsspec-2024.2.0:
      Successfully uninstalled fsspec-2024.2.0
  Attempting uninstall: ray
    Found existing installation: ray 2.9.3
    Uninstalling ray-2.9.3:
      Successfully uninstalled ray-2.9.3
  Attempting uninstall: gcsfs
    Found existing installation: gcsfs 2024.2.0
    Uninstalling gcsfs-2024.2.0:
      Successfully uninstalled gcsfs-2024.2.0
Successfully installed MarkupSafe-2.1.5 accelerate-0.28.0 asttokens-2.4.1 audioread-3.0.1 datasets-2.17.0 deepspeed-0.14.0 dill-0.3.8 etils-1.7.0 evaluate-0.4.1 executing-2.0.1 fsspec-2023.10.0 gcsfs-2023.10.0 hjson-3.1.0 huggingface-hub-0.23.4 ipython-8.22.2 jedi-0.19.1 jinja2-3.1.4 jiwer-3.0.0 joblib-1.4.2 librosa-0.10.0 llvmlite-0.43.0 matplotlib-inline-0.1.7 mpmath-1.3.0 multiprocess-0.70.16 ninja-1.11.1.1 numba-0.60.0 nvidia-cublas-cu11-11.11.3.6 nvidia-cuda-cupti-cu11-11.8.87 nvidia-cuda-nvrtc-cu11-11.8.89 nvidia-cuda-runtime-cu11-11.8.89 nvidia-cudnn-cu11-8.7.0.84 nvidia-cufft-cu11-10.9.0.58 nvidia-curand-cu11-10.3.0.86 nvidia-cusolver-cu11-11.4.1.48 nvidia-cusparse-cu11-11.7.5.86 nvidia-nccl-cu11-2.19.3 nvidia-nvtx-cu11-11.8.86 parso-0.8.4 pexpect-4.9.0 pooch-1.8.2 prompt-toolkit-3.0.47 ptyprocess-0.7.0 pure-eval-0.2.2 py-cpuinfo-9.0.0 pyarrow-15.0.2 pyarrow-hotfix-0.6 pynvml-11.5.0 rapidfuzz-2.13.7 ray-2.10.0 regex-2024.5.15 responses-0.18.0 safetensors-0.4.3 scikit-learn-1.5.0 soundfile-0.12.1 soxr-0.3.7 stack-data-0.6.3 sympy-1.12.1 threadpoolctl-3.5.0 tokenizers-0.15.2 torch-2.2.1+cu118 torchaudio-2.2.1+cu118 traitlets-5.14.3 transformers-4.39.0 triton-2.2.0 xxhash-3.4.1
Removing intermediate container b3a8ade1164a
 ---> f2a9a327fa33
Successfully built f2a9a327fa33
Successfully tagged us-central1-docker.pkg.dev/ray-whisper/ray-whisper/train:latest
PUSH
Pushing us-central1-docker.pkg.dev/ray-whisper/ray-whisper/train
The push refers to repository [us-central1-docker.pkg.dev/ray-whisper/ray-whisper/train]
197f7f36c28d: Preparing
...
256d88da4185: Preparing
a14e8e0160f2: Waiting
...
197f7f36c28d: Pushed
latest: digest: sha256:b24ec34f23dbdb4b6047b58d1c0f2c199f42e5b574600ffaa2cc77669b8f7280 size: 7865
DONE
--------------------------------------------------------------------------------
ID                                    CREATE_TIME                DURATION  SOURCE                                                                                     IMAGES                                                              STATUS
ec618557-0a2b-4538-bee8-f06f5b3b9938  2024-07-02T01:39:21+00:00  17M36S    gs://ray-whisper_cloudbuild/source/1719884360.752966-84217a4da65c4998bde9f438bc87865a.tgz  us-central1-docker.pkg.dev/ray-whisper/ray-whisper/train (+1 more)  SUCCESS
[18:57] $
```
{collapsible="true" collapsed-title="gcloud builds submit --region=${REGION} --tag=${TAG}"}

In the last 3 lines we are looking for `DONE`, time how long it took to build the image 
(17M36S), and finally `STATUS`: `SUCCESS`.

Now we are ready to launch a Ray cluster. We will use this custom image for both the head
node and the workers.

<seealso>
<!--Give some related links to how-to articles-->
</seealso>
