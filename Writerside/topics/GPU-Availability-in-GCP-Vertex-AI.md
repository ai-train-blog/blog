# GPU Availability in GCP Vertex AI

Google Cloud Compute Engine resources are 
[hosted](https://cloud.google.com/compute/docs/regions-zones) 
in multiple locations worldwide. These locations are composed of regions, 
which are further subdivided into zones. In the US the regions are
`us-central1`, `us-east1`, `us-east4`, `us-east5`, `us-south1`, `us-west1`, 
`us-west2`, `us-west3`, `us-west4`.
All regions have at least three zones; `us-central1` has 4 zones.
![](gcp_regions.png){width="650" style="block"}

In general, choosing regions is important for handling failures
due to outages and network latency (the closer the resource, the lower the latency). 

These considerations are less relevant for AI training because
Nvidia accelerators are available only in specific regions on particular machine types. 
We do not discuss TPU accelerators here.

First we list the machine types within Google Cloud that have access 
to Nvidia accelerators. Next, we list regions
and zones in the US with various types of Nvidia GPUs.

## GCP Compute Instances with GPUs

GCP has made the following machines with the following GPUs 
[generally available](https://cloud.google.com/compute/docs/gpus#gpus_for_compute_workloads)

1. [A2 machine series](https://cloud.google.com/compute/docs/gpus#a100-gpus)
    * A2 Ultra: [NVIDIA A100 80GB](https://www.nvidia.com/en-us/data-center/a100/)
    * A2 Standard: NVIDIA A100 40GB
2. [A3 machine series](https://cloud.google.com/compute/docs/gpus#h100-gpus)
    * A3 Standard: [NVIDIA H100 80GB](https://www.nvidia.com/en-us/data-center/h100)
    * A3 Mega: NVIDIA H100 80GB Mega
3. [G2 machine series](https://cloud.google.com/compute/docs/gpus#l4-gpus)
    * NVIDIA [L4](https://www.nvidia.com/en-us/data-center/l4)
4. [N1 machine series](https://cloud.google.com/compute/docs/gpus#n1-gpus)
    * NVIDIA [T4](https://resources.nvidia.com/en-us-gpu-resources/t4-tensor-core-datas)
    * NVIDIA [V100](https://www.nvidia.com/en-us/data-center/v100)
    * NVIDIA [P100](https://www.nvidia.com/en-us/data-center/tesla-p100)
    * NVIDIA [P4](https://images.nvidia.com/content/pdf/tesla/184457-Tesla-P4-Datasheet-NV-Final-Letter-Web.pdf)


## GCP Regions with Accelerators

GCP has data centers distributed in different regions, which are further divided into zones.
In the US, GCP 
[offers](https://cloud.google.com/vertex-ai/docs/general/locations#accelerators) 
the following combinations of regions and accelerators:

| Region    | Location             | Accelerators                                                                     |
|:----------|:---------------------|:---------------------------------------------------------------------------------|
| us-central1| Council Bluffs, IW   | A100 40GB, A100 80GB, H100, L4, P4, P100, T4, TPU V2*, TPU V2 Pod*, TPU V3*, V100|
|us-east1| Moncks Corner, SC    | A100 40GB*, L4, P100, T4, TPU V3*, V100* |                                        
|us-east4| Ashburn, VA          | A100 80GB, H100, L4, P4, T4                                                      |
|us-west1| The Dalles, OR       |	L4, P100, T4, H100*, TPU v5e†, V100 |
|us-west2| Los Angeles, CA | P4, T4 |
|us-west4|Las Vegas, NV | L4†, T4|
Note:
* \* Cells marked with asterisks represent regions where the specified accelerator
is available for training but not for serving batch or online predictions.
* † Cells marked with daggers represent regions where the specified accelerator
is available for serving batch or online predictions but not for training.
Also note:
If your job uses multiple types of GPUs, they must all be available in a single
zone in your region. For example, you can't run a job in australia-southeast1
using NVIDIA Tesla P4 GPUs, NVIDIA Tesla T4 GPUs, and NVIDIA Tesla P100 GPUs.
While all of these GPUs are available for jobs in australia-southeast1, no single
zone in that region provides all three types of GPU. To learn more about the zone
availability of GPUs, see 
[GPU availability by regions and zones](https://cloud.google.com/compute/docs/gpus/gpu-regions-zones#view-using-table).

In the `us-central1` region we have the largest choice of NVIDIA accelerators: 
A100 40GB, A100 80GB, H100, L4, P4, P100, T4, V100,

## GCP Zones with Accelerators

The `us-central1` region has accelerators available in the following zones:

| Zones | Location           | GPU machine type               | 
| :-----|:-------------------|:-------------------------------|
|us-central1-a | Council Bluffs, IW | A3 Mega, A3 Standard, A2 Ultra, A2 Standard, G2, N1+T4, N1+P4, N1+V100 |
|us-central1-b| Council Bluffs, IW | A2 Standard, G2, N1+T4, N1+V100|
|us-central1-c| Council Bluffs, IW | A3 Mega, A3 Standard, A2 Ultra, A2 Standard, G2, N1+T4, N1+P4, N1+V100, N1+P100|
|us-central1-f| Council Bluffs, IW | A2 Standard, N1+T4, N1+V100, N1+P100 |

Alternatively, GPU zones can be obtained with the GCP CLI
```bash
$ gcloud compute accelerator-types list
$ gcloud compute accelerator-types list | grep us-central1
ct5l                   us-central1-a              ct5l
ct5lp                  us-central1-a              ct5lp
nvidia-a100-80gb       us-central1-a              NVIDIA A100 80GB
nvidia-h100-80gb       us-central1-a              NVIDIA H100 80GB
nvidia-h100-mega-80gb  us-central1-a              NVIDIA H100 80GB MEGA
nvidia-l4              us-central1-a              NVIDIA L4
nvidia-l4-vws          us-central1-a              NVIDIA L4 Virtual Workstation
nvidia-tesla-a100      us-central1-a              NVIDIA A100 40GB
nvidia-tesla-p4        us-central1-a              NVIDIA Tesla P4
nvidia-tesla-p4-vws    us-central1-a              NVIDIA Tesla P4 Virtual Workstation
nvidia-tesla-t4        us-central1-a              NVIDIA T4
nvidia-tesla-t4-vws    us-central1-a              NVIDIA Tesla T4 Virtual Workstation
nvidia-tesla-v100      us-central1-a              NVIDIA V100
nvidia-h100-80gb       us-central1-b              NVIDIA H100 80GB
nvidia-h100-mega-80gb  us-central1-b              NVIDIA H100 80GB MEGA
nvidia-l4              us-central1-b              NVIDIA L4
nvidia-l4-vws          us-central1-b              NVIDIA L4 Virtual Workstation
nvidia-tesla-a100      us-central1-b              NVIDIA A100 40GB
nvidia-tesla-t4        us-central1-b              NVIDIA T4
nvidia-tesla-t4-vws    us-central1-b              NVIDIA Tesla T4 Virtual Workstation
nvidia-tesla-v100      us-central1-b              NVIDIA V100
nvidia-a100-80gb       us-central1-c              NVIDIA A100 80GB
nvidia-h100-80gb       us-central1-c              NVIDIA H100 80GB
nvidia-h100-mega-80gb  us-central1-c              NVIDIA H100 80GB MEGA
nvidia-l4              us-central1-c              NVIDIA L4
nvidia-l4-vws          us-central1-c              NVIDIA L4 Virtual Workstation
nvidia-tesla-a100      us-central1-c              NVIDIA A100 40GB
nvidia-tesla-p100      us-central1-c              NVIDIA Tesla P100
nvidia-tesla-p100-vws  us-central1-c              NVIDIA Tesla P100 Virtual Workstation
nvidia-tesla-p4        us-central1-c              NVIDIA Tesla P4
nvidia-tesla-p4-vws    us-central1-c              NVIDIA Tesla P4 Virtual Workstation
nvidia-tesla-t4        us-central1-c              NVIDIA T4
nvidia-tesla-t4-vws    us-central1-c              NVIDIA Tesla T4 Virtual Workstation
nvidia-tesla-v100      us-central1-c              NVIDIA V100
nvidia-tesla-a100      us-central1-f              NVIDIA A100 40GB
nvidia-tesla-p100      us-central1-f              NVIDIA Tesla P100
nvidia-tesla-p100-vws  us-central1-f              NVIDIA Tesla P100 Virtual Workstation
nvidia-tesla-t4        us-central1-f              NVIDIA T4
nvidia-tesla-t4-vws    us-central1-f              NVIDIA Tesla T4 Virtual Workstation
nvidia-tesla-v100      us-central1-f              NVIDIA V100
```
{collapsible="true" collapsed-title="gcloud compute accelerator-types list"}

## Vertex AI Availability in GCP

Google Cloud allows use of any supported location when creating a dataset, 
training a custom-trained model that doesn't use a managed dataset, 
or when importing an existing model. 
The recommendation is to use the region closest to the 
physical location of development or the physical location of intended users, 
provided that the required Vertex AI features are supported in the preferred
region. There is no global location.

For operations other than creating a dataset or importing a model, 
you must use the location of the resources you are operating on. 
For example, when you create a training pipeline that uses a managed dataset, 
you must use the region where the dataset is located.

Vertex AI is 
[available](https://cloud.google.com/vertex-ai/docs/general/locations#available-regions) 
in the following regions:
- Columbus, Ohio (us-east5)
- Dallas, Texas (us-south1)
- Iowa (us-central1)
- Las Vegas, Nevada (us-west4)
- Los Angeles, California (us-west2)
- Moncks Corner, South Carolina (us-east1)
- Northern Virginia (us-east4)
- Oregon (us-west1)
- Salt Lake City, Utah (us-west3)

<seealso>
    <!--Provide links to related how-to guides, overviews, and tutorials.-->
</seealso>