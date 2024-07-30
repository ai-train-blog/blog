# Create a Ray Cluster on Vertex AI

[Ray](https://docs.ray.io/en/latest/ray-overview/index.html)
is an open-source framework for scaling AI and Python applications. 
Ray provides the infrastructure to perform distributed computing and parallel 
processing for machine learning (ML) workflows.

In order to use train with Ray on Vertex AI, we need to be able to authenticate 
with Cloud Client Libraries, set up the Ray cluster according to available resources 
(CPU/GPU availability) in a particular region, start the cluster and connect to it.
Once we connect to the cluster, we can start with AI training.

## Authenticate Service Account with Application Default Credentials

So far we have authenticated the AI Training service account for access to Docker only.
In order to run python SDK need to set up a local Application Default Credentials
([ADC](https://cloud.google.com/docs/authentication/application-default-credentials)) 
file. ADC is a strategy used by the authentication libraries to automatically 
find credentials based on the application environment. The authentication libraries 
make those credentials available to Cloud Client Libraries and Google API Client 
Libraries. With ADC, the code can run in either a development or production environment 
without changing how the application authenticates to Google Cloud services and APIs.

ADC searches for credentials in the following locations:

1. **GOOGLE_APPLICATION_CREDENTIALS** environment variable that points to the location of a 
credential JSON file. One of the options for the JSON file is a service account key.
2. **User account credentials** set up by using the Google Cloud CLI with
`gcloud auth application-default login`.
   ```Bash
   $ gcloud auth application-default login
   Your browser has been opened to visit:
   
       https://accounts.google.com/o/oauth2/auth?response_type=code&client_id=764086051850-6qr4p6gpi6hn506pt8ejuq83di341hur.apps.googleusercontent.com&redirect_uri=http%3A%2F%2Flocalhost%3A8085%2F&scope=openid+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.email+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcloud-platform+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fsqlservice.login&state=LBnB7qGqOYzUQE8S1bmiTcLjtBfugE&access_type=offline&code_challenge=QAEBWbqQde9OVuzvhRIxrj5AaJCazknbzmleX2KRrWE&code_challenge_method=S256
   
   
   Credentials saved to file: [${HOME}/.config/gcloud/application_default_credentials.json]
   
   These credentials will be used by any library that requests Application Default Credentials (ADC).
   
   Quota project "ray-whisper" was added to ADC which can be used by Google client libraries for billing and quota. Note that some services may still bill the project owning the resource. 
   ```
   {collapsible="true" collapsed-title="gcloud auth application-default"}

   It creates `~/.config/gcloud/application_default_credentials.json` with credentials.
3. **Service account credentials** using 
[service account impersonation](https://cloud.google.com/docs/authentication/provide-credentials-adc#sa-impersonation).

   Many Google Cloud services let you attach a service account that can be used to 
   provide credentials for accessing Google Cloud APIs. If ADC does not find credentials as
   already discussed in the previous two paragraphs, it uses the 
   [metadata server](https://cloud.google.com/compute/docs/metadata/overview) 
   to get credentials for the service where the code is running.
   We use the following `gcloud` command for service account impersonation to create
   a local ADC file
      ```Bash
      $ gcloud auth application-default login --impersonate-service-account ${SERVICE_ACCOUNT}
      Your browser has been opened to visit:
      
          https://accounts.google.com/o/oauth2/auth?response_type=code&client_id=764086051850-6qr4p6gpi6hn506pt8ejuq83di341hur.apps.googleusercontent.com&redirect_uri=http%3A%2F%2Flocalhost%3A8085%2F&scope=openid+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.email+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcloud-platform+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fsqlservice.login&state=KAB4OPVV83HTbcQSvEHvPSIH2gFn6k&access_type=offline&code_challenge=mSFNnHvv1euMc1wCjhwJ4gyB0dJImhFW3l3YANiMwTI&code_challenge_method=S256
      
      
      Credentials saved to file: [${HOME}/.config/gcloud/application_default_credentials.json]
      
      These credentials will be used by any library that requests Application Default Credentials (ADC).
      ```
      {collapsible="true" collapsed-title="gcloud auth service account impersonation"}
      It is now possible to use client libraries using the supported languages (`C#`, `Go`, `Java`,
      `Node.js` and `Python`) the same way you would after setting up a local ADC file with user 
      credentials. Credentials are automatically found by the authentication libraries. This is the 
      preferred method of authenticating the service account with ADC.

   The credential json file contains  `client_id`, `client_secret` and `refresh_token` for
   the impersonated service account.
   ```json
   {
     "delegates": [],
     "service_account_impersonation_url": "https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/ray-whisper-ai-training@ray-whisper.iam.gserviceaccount.com:generateAccessToken",
     "source_credentials": {
       "account": "",
       "client_id": "764086051850-6qr4p6gpi6hn302pt8ejuq83di341hur.apps.googleusercontent.com",
       "client_secret": "d-FL95Q19q7MQmFpd7hHD0Ty",
       "refresh_token": "1//06JcIfzK2WfBUCgYIARAAGAYSNgF-L9IrVmO0x4s0RLzzR-dxZWYWQ3LVhzhUdzrtOWLf6qLgQgVmotpmSXhfxaPm-uX7qy8IDA",
       "type": "authorized_user",
       "universe_domain": "googleapis.com"
     },
     "type": "impersonated_service_account"
   }
   ```
   {collapsible="true" collapsed-title="Service Account credentials"}

## Create a Ray Cluster

Assuming we have already created a development environment, prepared a development jupyter
kernel, and pip installed `google-cloud-aiplatform[ray]` into the virtual environment as 
previously described, we can start a jupyter notebook from which we can launch a Ray
cluster.

> Note:
> We installed `pandas`, `ray` and `vertex_ray` as dependencies to `google-cloud-aiplatform[ray]`.

We have developed several utilities to streamline Ray cluster provision. The code is provided in
`./create_cluster.py`. The two major components of this source file are the function `infra_config`
and the class RayClusterConfig`.

The auxiliary function `infra_config` returns available 
machine-accelerator pairs, and provides the associated cost per hour.
```python
def infra_config(n_core: int = 4, nvidia_gpu: Union[str, None] = None):
    """
    N1 Series machines are n1-standard-4, n1-standard-8 and n1-standard-16. They have 4, 8 and 16 vCPUs and
    15GB, 30GB and 60GB of memory, respectively.
    The available accelerators are T4, P4, P100 and V100. Note that a single V100 must have at most 12 vCPUs

    Available accelerator types can be found in google.cloud.aiplatform.gapic.AcceleratorType

    class AcceleratorType(proto.Enum):
        ...
        ACCELERATOR_TYPE_UNSPECIFIED = 0
        NVIDIA_TESLA_K80 = 1
        NVIDIA_TESLA_P100 = 2
        NVIDIA_TESLA_V100 = 3
        NVIDIA_TESLA_P4 = 4
        NVIDIA_TESLA_T4 = 5
        NVIDIA_TESLA_A100 = 8
        NVIDIA_A100_80GB = 9
        NVIDIA_L4 = 11
        NVIDIA_H100_80GB = 13
        TPU_V2 = 6
        TPU_V3 = 7
        TPU_V4_POD = 10
        TPU_V5_LITEPOD = 12

        Ray does not support TPU machines.

        Also, check out https://cloud.google.com/vertex-ai/docs/training/configure-compute#specifying_gpus

    :param n_core: int; 4, 8 or 16
    :param nvidia_gpu: str or None; use 'T4', 'P4', 'P100', 'V100' or None
    :return: (str, str, float), 3-tuple with machine_type, accelerator_type and cost per hour
    """

    n_core_choices = [4, 8, 16]
    assert n_core in n_core_choices, f'Invalid number of cores {n_core}. Choose from {n_core_choices}.'

    idx = n_core_choices.index(n_core)

    available_machines = ['n1-standard-4', 'n1-standard-8', 'n1-standard-16']
    available_accelerators = {
        None: {None: [.13, .27, .53]},
        'T4': {'NVIDIA_TESLA_T4': [.38, .51, .78]},
        'P4': {'NVIDIA_TESLA_P4': [.55, .69, .95]},
        'P100': {'NVIDIA_TESLA_P100': [1.16, 1.29, 1.56]},
        # Instances with 1 NVIDIA V100 must have at most 12 vCPU cores.
        'V100': {'NVIDIA_TESLA_V100': [1.87, 2.00, None]}
    }

    assert nvidia_gpu in list(available_accelerators.keys()), \
        f'Invalid nvidia_gpu {nvidia_gpu!r}. Choose from {list(available_accelerators.keys())}.'

    machine_type = available_machines[idx]
    gpu_cost = available_accelerators[nvidia_gpu]
    accelerator_type = list(gpu_cost.keys()).pop()
    cost_per_hr = list(gpu_cost.values()).pop()[idx]

    return machine_type, accelerator_type, cost_per_hr
```
{collapsible="true" collapsed-title="infra_config: returns machine_type, accelerator_type, cost_per_hr"}

The auxilliary class `RayClusterConfig` leverages `infra_config` to set up
a Ray cluster and estimate the total cost for the provisioned resources.
```python
class RayClusterConfig:

    def __init__(self, head_n_cores=4, worker_n_cores=4, worker_gpu='T4',
                 worker_node_count=3, accelerator_count=1):
        self.head_n_cores = head_n_cores
        self.worker_n_cores = worker_n_cores
        self.worker_gpu = worker_gpu
        self.worker_node_count = worker_node_count
        self.accelerator_count = accelerator_count

        self.head_machine_type, _, self.head_machine_cost_per_hr = infra_config(n_core=head_n_cores, nvidia_gpu=None)
        self.worker_machine_type, self.accelerator_type, self.worker_machine_cost_per_hr = \
            infra_config(n_core=worker_n_cores, nvidia_gpu=worker_gpu)

        self._cluster_name = None
        self._address = None

        # Argument consistency for no GPUs
        if self.worker_gpu is None or self.accelerator_count == 0:
            self.accelerator_count = 0
            self.accelerator_type = None
            self.worker_gpu = None

        # This is supposed to croak if the env variables are undefined.
        self.project = os.environ['PROJECT_ID']
        self.project_number = os.environ['PROJECT_NUMBER']
        self.region = os.environ['REGION']
        self.repo = os.environ['REPO']
        self.service_account = os.environ['SERVICE_ACCOUNT']
        self.staging_bucket = os.environ['BUCKET_URI']

    def __repr__(self):
        args_attributes = ['head_n_cores', 'worker_n_cores', 'worker_gpu', 'worker_node_count', 'accelerator_count']
        args_lst = []
        for k in args_attributes:
            v = getattr(self, k)
            if k == 'worker_gpu':
                v = str(v)
            else:
                v = int(v)
            args_lst.append(f'{k}={v!r}')
        ret = self.__class__.__name__ + '('
        ret += ', '.join(args_lst)
        ret += ')'
        return ret

    def __str__(self):
        attributes = [k for k in self.__dict__ if not k.startswith('_')]
        properties = [p for p in dir(self.__class__) if isinstance(getattr(self.__class__, p), property)]

        ret = self.__class__.__name__ + ":\n"
        ret += '  Attributes:\n'
        for v in sorted(attributes):
            ret += f'    {v} = '
            ret += f'{getattr(self, v)!r}'
            ret += '\n'

        ret += '  Properties:\n'
        for v in sorted(properties):
            ret += f'    {v} = '
            ret += f'{getattr(self, v)!r}'
            ret += '\n'

        return ret

    def cost_per_hr(self, hrs: float):
        """
        Calculate hourly cost of running the ray cluster
        :param hrs: float. If working with seconds, divide by 3600 before passing
        :return: flot, runtime cost of the ray cluster
        """
        cost_per_hr = self.head_machine_cost_per_hr
        cost_per_hr += self.worker_node_count * self.accelerator_count * self.worker_machine_cost_per_hr
        return hrs * cost_per_hr

    @property
    def address(self):
        if self._address is None:
            self._address = f'projects/{self.project_number}/locations/{self.region}'
            self._address += f'/persistentResources/{self.cluster_name}'
        return self._address

    @property
    def cluster_cost_per_hour(self):
        return self.cost_per_hr(hrs=1.)

    @property
    def cluster_name(self):
        if self._cluster_name is None:
            self._cluster_name = '-'.join(
                [
                    self.head_node_desc,
                    self.worker_node_desc,
                    timestamped_unique_name()
                ]
            )

            # From vertex_ray.create_ray_cluster docstring
            # cluster_name: This value may be up to 63 characters, and valid
            #             characters are `[a-z0-9_-]`. The first character cannot be a number
            #             or hyphen.
            self._cluster_name = self._cluster_name[:63]
        return self._cluster_name

    @property
    def custom_images(self):
        return NodeImages(
            head=self.train_image,
            worker=self.train_image
        )

    @property
    def gpu(self):
        if self.worker_gpu is None:
            _gpu = f'nogpus'
        else:
            _gpu = f'{self.accelerator_count}x{self.worker_gpu.lower()}'

        return _gpu

    @property
    def head_node_desc(self):
        return f'h{self.head_n_cores}vcpu'

    @property
    def head_node_type(self):
        return Resources(
            machine_type=self.head_machine_type,
            node_count=1
        )

    @property
    def train_image(self):
        # return f'{self.region}-docker.pkg.dev/{self.project}/{self.repo}/train'
        return os.environ['TRAIN_IMAGE']

    @property
    def worker_node_desc(self):
        return f'w{self.worker_node_count}x{self.worker_n_cores}vcpu{self.gpu}'

    @property
    def worker_node_types(self):
        return [
            Resources(
                machine_type=self.worker_machine_type,
                node_count=self.worker_node_count,
                accelerator_type=self.accelerator_type,
                accelerator_count=self.accelerator_count
            )
        ]
```
{collapsible="true" collapsed-title="RayClusterConfig"}

We start a jupyter notebook named `create_cluster.ipynb` with the following imports:

```python
import time          # To time how long it takes to spawn a cluster

import vertex_ray    # To start a cluster
import vertexai      # To connect to a project

# Code from './create_cluster.py' to assist with Ray cluster config and launch
import create_cluster
```

Next, we config a Ray cluster on which we intend to fine tune Whisper. The head node
must be at least an 8-core 
[`n1-standard-8`](https://cloud.google.com/compute/docs/general-purpose-machines#n1_machines)
machine with 30 GB of memory and without accelerators. We also provision 3 workers, 
`n1-standard-8` each with a single Nvidia T4 accelerator 

```python
config = create_cluster.RayClusterConfig(
    head_n_cores=8,
    worker_n_cores=8,
    worker_gpu='T4',
    worker_node_count=3,
    accelerator_count=1
)
```
The provided class `RayClusterConfig` then creates attributes and properties
which annotate the Ray cluster, and can be passed to `vertex_ray.create_ray_cluster`

```python
RayClusterConfig:
  Attributes:
    accelerator_count = 1
    accelerator_type = 'NVIDIA_TESLA_T4'
    head_machine_cost_per_hr = 0.27
    head_machine_type = 'n1-standard-8'
    head_n_cores = 8
    project = 'ray-whisper'
    project_number = '702949608356'
    region = 'us-central1'
    repo = 'ray-whisper'
    service_account = 'ray-whisper-ai-training@ray-whisper.iam.gserviceaccount.com'
    staging_bucket = 'gs://ray-whisper-bucket'
    worker_gpu = 'T4'
    worker_machine_cost_per_hr = 0.51
    worker_machine_type = 'n1-standard-8'
    worker_n_cores = 8
    worker_node_count = 3
  Properties:
    address = 'projects/702949608356/locations/us-central1/persistentResources/h8vcpu-w3x8vcpu1xt4-2024-07-15-17-50-21-0b3a5'
    cluster_cost_per_hour = 1.8
    cluster_name = 'h8vcpu-w3x8vcpu1xt4-2024-07-15-17-50-21-0b3a5'
    custom_images = NodeImages(head='us-central1-docker.pkg.dev/ray-whisper/ray-whisper/train', worker='us-central1-docker.pkg.dev/ray-whisper/ray-whisper/train')
    gpu = '1xt4'
    head_node_desc = 'h8vcpu'
    head_node_type = Resources(machine_type='n1-standard-8', node_count=1, accelerator_type=None, accelerator_count=0, boot_disk_type='pd-ssd', boot_disk_size_gb=100, custom_image=None)
    train_image = 'us-central1-docker.pkg.dev/ray-whisper/ray-whisper/train'
    worker_node_desc = 'w3x8vcpu1xt4'
    worker_node_types = [Resources(machine_type='n1-standard-8', node_count=3, accelerator_type='NVIDIA_TESLA_T4', accelerator_count=1, boot_disk_type='pd-ssd', boot_disk_size_gb=100, custom_image=None)]
```
{collapsible="true" collapsed-title="RayClusterConfig attributes"}

The we initialize the project

```python
vertexai.init(
    project=config.project, 
    location=config.region,
    service_account=config.service_account,
    staging_bucket=config.staging_bucket
)
```

We start a Ray cluster with the following command in the jupyter notebook:
```python
%%time
ray_cluster_name = vertex_ray.create_ray_cluster(
    head_node_type=config.head_node_type,
    python_version='3.10',
    ray_version='2.9',
    service_account=config.service_account,
    cluster_name=config.cluster_name,
    worker_node_types=config.worker_node_types,
    custom_images=config.custom_images,
    enable_metrics_collection=True
)
```
This method returns a string `ray_cluster_name` which we need
access the Ray cluster and instantiate the training client.

It takes about a little bit under 15 minutes to start a Ray cluster with 3 Nvidia T4-enabled 
workers:
```Bash
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 1; sleeping for 0:02:30 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 2; sleeping for 0:01:54.750000 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 3; sleeping for 0:01:27.783750 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 4; sleeping for 0:01:07.154569 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 5; sleeping for 0:00:51.373245 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 6; sleeping for 0:00:39.300532 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 7; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 8; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 9; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 10; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 11; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 12; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 13; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 14; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 15; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 16; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 17; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 18; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.RUNNING
delta_t = 895.1032409667969
CPU times: user 903 ms, sys: 685 ms, total: 1.59 s
Wall time: 14min 55s
```
{collapsible="true" collapsed-title="vertex_ray.create_ray_cluster 3 T$ GPUs output"}

Note: Starting a cluster can be a hit or miss. In extreme cases in take up to
a couple of hours to start a ray cluster:
```Bash
%%time
ray_cluster_name = vertex_ray.create_ray_cluster(
    head_node_type=config.head_node_type,
    python_version='3.10',
    ray_version='2.9',
    service_account=config.service_account,
    cluster_name=config.cluster_name,
    worker_node_types=config.worker_node_types,
    custom_images=config.custom_images,
    enable_metrics_collection=True
)

[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 1; sleeping for 0:02:30 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 2; sleeping for 0:01:54.750000 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 3; sleeping for 0:01:27.783750 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 4; sleeping for 0:01:07.154569 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 5; sleeping for 0:00:51.373245 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 6; sleeping for 0:00:39.300532 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 7; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 8; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 9; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 10; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 11; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 12; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 13; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 14; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 15; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 16; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 17; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 18; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 19; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 20; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 21; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 22; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 23; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 24; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 25; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 26; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 27; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 28; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 29; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 30; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 31; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 32; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 33; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 34; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 35; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 36; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 37; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 38; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 39; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 40; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 41; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 42; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 43; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 44; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 45; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 46; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 47; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 48; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 49; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 50; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 51; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 52; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 53; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 54; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 55; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 56; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 57; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 58; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 59; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 60; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 61; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 62; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 63; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 64; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 65; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 66; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 67; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 68; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 69; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 70; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 71; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 72; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 73; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 74; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 75; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 76; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 77; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 78; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 79; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 80; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 81; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 82; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 83; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 84; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 85; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 86; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 87; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 88; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 89; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 90; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 91; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 92; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 93; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 94; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 95; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 96; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 97; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 98; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 99; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 100; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 101; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 102; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 103; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 104; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 105; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 106; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 107; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 108; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 109; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 110; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 111; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 112; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 113; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 114; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 115; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 116; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 117; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 118; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 119; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 120; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 121; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 122; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 123; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 124; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 125; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 126; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 127; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 128; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 129; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 130; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 131; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 132; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 133; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 134; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 135; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 136; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 137; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 138; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 139; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 140; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 141; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 142; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 143; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 144; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 145; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 146; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 147; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 148; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 149; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 150; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 151; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 152; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 153; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 154; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 155; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 156; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 157; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 158; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 159; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 160; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 161; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 162; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 163; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 164; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 165; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 166; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 167; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 168; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 169; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 170; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 171; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 172; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 173; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 174; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 175; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 176; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 177; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 178; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 179; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 180; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 181; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 182; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 183; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 184; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 185; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 186; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 187; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 188; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 189; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 190; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 191; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 192; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 193; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 194; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 195; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 196; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 197; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 198; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 199; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 200; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 201; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 202; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 203; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 204; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 205; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 206; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 207; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 208; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 209; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 210; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 211; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 212; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 213; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 214; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 215; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 216; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 217; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 218; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 219; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 220; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 221; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 222; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 223; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 224; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 225; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 226; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 227; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 228; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 229; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 230; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 231; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 232; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 233; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 234; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 235; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 236; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 237; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 238; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 239; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 240; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 241; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 242; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 243; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 244; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 245; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 246; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 247; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 248; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 249; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 250; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 251; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 252; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 253; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 254; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 255; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 256; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 257; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 258; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 259; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.RUNNING
CPU times: user 8.34 s, sys: 7.99 s, total: 16.3 s
Wall time: 2h 17min 56s
```
{collapsible="true" collapsed-title="vertex_ray.create_ray_cluster long startup"}


> **Important**
>
> If you have already started creating a Ray cluster,
> you can neither stop nor terminate the process. You have
> to wait for the cluster to spawn before you can terminate it.
{style="note"} 
> 
Incidentally, it took over 18 minutes to start a cluster without GPUs. We know that
GPUs speed shit up, but this is definitely a new use case.
```Bash
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 1; sleeping for 0:02:30 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 2; sleeping for 0:01:54.750000 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 3; sleeping for 0:01:27.783750 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 4; sleeping for 0:01:07.154569 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 5; sleeping for 0:00:51.373245 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 6; sleeping for 0:00:39.300532 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 7; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 8; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 9; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 10; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 11; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 12; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 13; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 14; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 15; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 16; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 17; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 18; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 19; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 20; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 21; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 22; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 23; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 24; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 25; sleeping for 0:00:30.064907 seconds
[Ray on Vertex AI]: Cluster State = State.RUNNING
delta_t = 1096.0307490825653
CPU times: user 922 ms, sys: 922 ms, total: 1.84 s
Wall time: 18min 16s
```
{collapsible="true" collapsed-title="vertex_ray.create_ray_cluster No GPUs output"}


At soon as you see in the jupyter notebook
```python
[Ray on Vertex AI]: Cluster State = State.RUNNING
```
you will have had the training infrastructure ready.

Google Cloud console will also confirm that you have a running Ray cluster.
![](Open_Ray_Cluster_in_Colab_Enterprise.png){width="450" style="block"}
Once the cluster spawns, in the console you are presented with an option of 
accessing the cluster from Enterprise Colab. 

Note: Ray clusters 
[remain available](https://cloud.google.com/vertex-ai/docs/open-source/ray-on-vertex-ai/overview#overview) 
until deleted.

## Access Ray Cluster for AI Training

The next step is to establish a connection with the Ray cluster. We have at
our disposal a head node with 8 CPUs (`h8vcpu`) and 3 workers with 
identical CPU configuration and a single Nvidia T4 GPU (w3x8vcpu1xt4)
each:

```python
[19]: ray_cluster = vertex_ray.get_ray_cluster(ray_cluster_name)
[20]: ray_cluster
      Cluster(
         cluster_resource_name='projects/702949608356/locations/us-central1/persistentResources/h8vcpu-w3x8vcpu1xt4-2024-07-19-21-16-51-43ac7', 
         network='', 
         service_account='ray-ai-training@ray-whisper.iam.gserviceaccount.com', 
         state='RUNNING', 
         python_version=None, 
         ray_version=None, 
         head_node_type=Resources(
            machine_type='n1-standard-8', 
            node_count=1, 
            accelerator_type=None, 
            accelerator_count=0, 
            boot_disk_type='pd-ssd', 
            boot_disk_size_gb=100, 
            custom_image='us-central1-docker.pkg.dev/ray-whisper/ray-whisper/train'
         ), 
         worker_node_types=[
            Resources(
               machine_type='n1-standard-8',
               node_count=3, 
               accelerator_type='NVIDIA_TESLA_T4', 
               accelerator_count=1, 
               boot_disk_type='pd-ssd', 
               boot_disk_size_gb=100, 
               custom_image='us-central1-docker.pkg.dev/ray-whisper/ray-whisper/train'
            )
         ], 
         dashboard_address='a01762c924611669-dot-us-central1.aiplatform-training.googleusercontent.com', 
         labels={}
      )
```

From `ray_cluster` object we need the `dashboard_address` attribute to 
instantiate 
a 
[client](https://docs.ray.io/en/latest/cluster/running-applications/job-submission/doc/ray.job_submission.JobSubmissionClient.html)
through which we 
[submit a training job](https://docs.ray.io/en/latest/cluster/running-applications/job-submission/doc/ray.job_submission.JobSubmissionClient.submit_job.html#ray.job_submission.JobSubmissionClient.submit_job)
for asynchronous execution on the Ray cluster.

```python
from ray.job_submission import JobSubmissionClient
client = JobSubmissionClient(address=f'vertex_ray://{ray_cluster.dashboard_address}')
```

> Important: 
> 
> Accessing the Ray training client **does not** work.
> 
> I tried Enterprise Colab using the default service agent. Next I 
> enhanced the default agent with added `roles/aiplatform.admin`;
> My own account with `owner` permissions, with `roles/aiplatform.admin`;
> Service account impersonation with `roles/aiplatform.admin`, using
> recommended (temporary tokens) and not-recommended auth
> (service account key with no expiration date) methods, 
> all to to no avail.
> 
> I also tried following Google Kubernetes Engine (GKE) auth recommended
> by GCP, but Ray cluster do not run on GKE.
> 
> It is possible to start a Ray cluster within a VPC network. In that case
> connection with Google Cloud Storage bucket get affected and requires
> additional configuration.
> 
> Current status: impossible to access Ray cluster, in communication
> with GCP about resolution.
{style="warning"}

The connection times out.
```python
---------------------------------------------------------------------------
HTTPError                                 Traceback (most recent call last)
Cell In[10], line 1
----> 1 client = JobSubmissionClient(address=f'vertex_ray://{ray_cluster.cluster_resource_name}')

File ~/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages/ray/dashboard/modules/job/sdk.py:109, in JobSubmissionClient.__init__(self, address, create_cluster_if_needed, cookies, metadata, headers, verify)
     99 api_server_url = get_address_for_submission_client(address)
    101 super().__init__(
    102     address=api_server_url,
    103     create_cluster_if_needed=create_cluster_if_needed,
   (...)
    107     verify=verify,
    108 )
--> 109 self._check_connection_and_version(
    110     min_version="1.9",
    111     version_error_message="Jobs API is not supported on the Ray "
    112     "cluster. Please ensure the cluster is "
    113     "running Ray 1.9 or higher.",
    114 )
    116 # In ray>=2.0, the client sends the new kwarg `submission_id` to the server
    117 # upon every job submission, which causes servers with ray<2.0 to error.
    118 if packaging.version.parse(self._client_ray_version) > packaging.version.parse(
    119     "2.0"
    120 ):

File ~/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages/ray/dashboard/modules/dashboard_sdk.py:248, in SubmissionClient._check_connection_and_version(self, min_version, version_error_message)
    245 def _check_connection_and_version(
    246     self, min_version: str = "1.9", version_error_message: str = None
    247 ):
--> 248     self._check_connection_and_version_with_url(min_version, version_error_message)

File ~/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages/ray/dashboard/modules/dashboard_sdk.py:267, in SubmissionClient._check_connection_and_version_with_url(self, min_version, version_error_message, url)
    263 if r.status_code == 404:
    264     raise RuntimeError(
    265         "Version check returned 404. " + version_error_message
    266     )
--> 267 r.raise_for_status()
    269 running_ray_version = r.json()["ray_version"]
    270 if packaging.version.parse(running_ray_version) < packaging.version.parse(
    271     min_version
    272 ):

File ~/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages/requests/models.py:1024, in Response.raise_for_status(self)
   1019     http_error_msg = (
   1020         f"{self.status_code} Server Error: {reason} for url: {self.url}"
   1021     )
   1023 if http_error_msg:
-> 1024     raise HTTPError(http_error_msg, response=self)

HTTPError: 524 Server Error: status code 524 for url: https://a01762c924611669-dot-us-central1.aiplatform-training.googleusercontent.com/api/version
```
{collapsible="true" collapsed-title="Ray Client connection 524 Server Error"}

