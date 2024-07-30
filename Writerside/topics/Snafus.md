# Snafus

## Build image

```bash
$ gcloud builds submit \
  --region=${REGION} \
  --tag=$TAG \
  --machine-type=${BUILD_MACHINE_TYPE} \
  --timeout=3600
Creating temporary archive of 2 file(s) totalling 514 bytes before compression.
Some files were not included in the source upload.

Check the gcloud log [/Users/damir/.config/gcloud/logs/2024.07.01/17.06.30.916785.log] to see which files and the contents of the
default gcloudignore file used (see `$ gcloud topic gcloudignore` to learn
more).

Uploading tarball of [./] to [gs://ray-whisper_cloudbuild/source/1719878791.146411-8db94407a6584c97acb898a5be8a2690.tgz]
ERROR: (gcloud.builds.submit) FAILED_PRECONDITION: failed precondition: due to quota restrictions, cannot run builds of this machine type in this region, see https://cloud.google.com/build/docs/locations#restricted_regions_for_some_projects
```

Note that the link says that `us-central1` should work:

![](failed_precondition.png){width="450" style="block"}

On the second attempt I did not specify the `machine-type`. It takes less than 20 minutes 
to build the image


## On my laptop
Use python code in jupyter lab locally on my laptop

```python
ray_cluster_name = vertex_ray.create_ray_cluster(
    head_node_type=config.head_node_type,
    cluster_name=config.cluster_name,
    worker_node_types=config.worker_node_types,
    enable_metrics_collection=True,
    labels={
        'head': config.head_node_desc,
        'worker': config.worker_node_desc,
        'hourly_cost': f'${config.run_cost(1):.2f}'
    }
)

---------------------------------------------------------------------------
PermissionDenied                          Traceback (most recent call last)
Cell In[12], line 1
...
PermissionDenied: 403 Permission 'resourcemanager.projects.get' denied on resource '//cloudresourcemanager.googleapis.com/projects/ray-whisper' (or it may not exist). [reason: "IAM_PERMISSION_DENIED"
domain: "cloudresourcemanager.googleapis.com"
metadata {
  key: "resource"
  value: "projects/ray-whisper"
}
metadata {
  key: "permission"
  value: "resourcemanager.projects.get"
}
]
```
{collapsible="true" collapsed-title="vertex_ray.create_ray_cluster error"}

IAM role is overly non-permissive. According to 
[https://gcp.permissions.cloud](https://gcp.permissions.cloud/iam/resourcemanager), 
`resourcemanager.projects.get` 
is not documented, but is 
[included](https://gcp.permissions.cloud/iam/resourcemanager#resourcemanager.projects.get) 
in the predefined role `roles/aiplatform.user`

Also, checking on the command line

```bash
$ gcloud iam roles describe roles/aiplatform.user
description: Grants access to use all resource in Vertex AI
etag: AA==
includedPermissions:
- aiplatform.agentExamples.create
...
- aiplatform.tuningJobs.vertexTune
- resourcemanager.projects.get
- resourcemanager.projects.list
name: roles/aiplatform.user
stage: GA
title: Vertex AI User
```

Tried passing to `aiplatform.init` service account arg with presumably appropriate IAM 
permissions, but that did not work, either.

## On Vertex AI jupyter notebook

A single pip install required

```bash
pip install google-cloud-aiplatform[ray]
```

Used code from `create_cluster.py`

### Head 4 vCPU/15GB of RAM

#### Code generated cluster name 

Ray, or grpc, does not allow underscores in labels/names, even though
the source docstring says otherwise

```python
print(config.cluster_name)
h_4vCPUs-w_3x4vCPUs1xT4-2024-07-02-06-22-12-2ba3c

len(config.cluster_name)
49
```

Create a ray cluster with a name that contains '_':

```python
ray_cluster_name = vertex_ray.create_ray_cluster(
    head_node_type=config.head_node_type,
    cluster_name=config.cluster_name,
    worker_node_types=config.worker_node_types,
    enable_metrics_collection=True,
    labels={
        'head': config.head_node_desc,
        'worker': config.worker_node_desc,
        'hourly_cost': f'${config.run_cost(1):.2f}'
    }
)

---------------------------------------------------------------------------
_InactiveRpcError                         Traceback (most recent call last)
File /opt/conda/lib/python3.10/site-packages/google/api_core/grpc_helpers.py:65, in _wrap_unary_errors.<locals>.error_remapped_callable(*args, **kwargs)
     64 try:
---> 65     return callable_(*args, **kwargs)
     66 except grpc.RpcError as exc:

File /opt/conda/lib/python3.10/site-packages/grpc/_channel.py:1181, in _UnaryUnaryMultiCallable.__call__(self, request, timeout, metadata, credentials, wait_for_ready, compression)
   1175 (
   1176     state,
   1177     call,
   1178 ) = self._blocking(
   1179     request, timeout, metadata, credentials, wait_for_ready, compression
   1180 )
-> 1181 return _end_unary_response_blocking(state, call, False, None)

File /opt/conda/lib/python3.10/site-packages/grpc/_channel.py:1006, in _end_unary_response_blocking(state, call, with_call, deadline)
   1005 else:
-> 1006     raise _InactiveRpcError(state)

_InactiveRpcError: <_InactiveRpcError of RPC that terminated with:
	status = StatusCode.INVALID_ARGUMENT
	details = "List of found errors:	1.Field: persistent_resource_id; Message: persistent_resource_id can only contain numbers, lower case letters and hyphens. The first character must be a letter, and the last character must be a letter or number.	2.Field: persistent_resource.labels; Message: There can be no more than 64 labels attached to a single resource. Label keys and values can only contain lowercase letters, numbers, dashes and underscores. Label keys must start with a letter or number, must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8. Label values must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8.	"
	debug_error_string = "UNKNOWN:Error received from peer ipv4:172.217.214.95:443 {created_time:"2024-07-02T06:08:46.707923825+00:00", grpc_status:3, grpc_message:"List of found errors:\t1.Field: persistent_resource_id; Message: persistent_resource_id can only contain numbers, lower case letters and hyphens. The first character must be a letter, and the last character must be a letter or number.\t2.Field: persistent_resource.labels; Message: There can be no more than 64 labels attached to a single resource. Label keys and values can only contain lowercase letters, numbers, dashes and underscores. Label keys must start with a letter or number, must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8. Label values must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8.\t"}"
>

The above exception was the direct cause of the following exception:

InvalidArgument                           Traceback (most recent call last)
File /opt/conda/lib/python3.10/site-packages/google/cloud/aiplatform/vertex_ray/cluster_init.py:297, in create_ray_cluster(head_node_type, python_version, ray_version, network, service_account, cluster_name, worker_node_types, custom_images, enable_metrics_collection, labels)
    296 try:
--> 297     _ = client.create_persistent_resource(request)
    298 except Exception as e:

File /opt/conda/lib/python3.10/site-packages/google/cloud/aiplatform_v1/services/persistent_resource_service/client.py:867, in PersistentResourceServiceClient.create_persistent_resource(self, request, parent, persistent_resource, persistent_resource_id, retry, timeout, metadata)
    866 # Send the request.
--> 867 response = rpc(
    868     request,
    869     retry=retry,
    870     timeout=timeout,
    871     metadata=metadata,
    872 )
    874 # Wrap the response in an operation future.

File /opt/conda/lib/python3.10/site-packages/google/api_core/gapic_v1/method.py:113, in _GapicCallable.__call__(self, timeout, retry, *args, **kwargs)
    111     kwargs["metadata"] = metadata
--> 113 return wrapped_func(*args, **kwargs)

File /opt/conda/lib/python3.10/site-packages/google/api_core/grpc_helpers.py:67, in _wrap_unary_errors.<locals>.error_remapped_callable(*args, **kwargs)
     66 except grpc.RpcError as exc:
---> 67     raise exceptions.from_grpc_error(exc) from exc

InvalidArgument: 400 List of found errors:	1.Field: persistent_resource_id; Message: persistent_resource_id can only contain numbers, lower case letters and hyphens. The first character must be a letter, and the last character must be a letter or number.	2.Field: persistent_resource.labels; Message: There can be no more than 64 labels attached to a single resource. Label keys and values can only contain lowercase letters, numbers, dashes and underscores. Label keys must start with a letter or number, must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8. Label values must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8.	 [field_violations {
  field: "persistent_resource_id"
  description: "persistent_resource_id can only contain numbers, lower case letters and hyphens. The first character must be a letter, and the last character must be a letter or number."
}
field_violations {
  field: "persistent_resource.labels"
  description: "There can be no more than 64 labels attached to a single resource. Label keys and values can only contain lowercase letters, numbers, dashes and underscores. Label keys must start with a letter or number, must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8. Label values must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8."
}
]

The above exception was the direct cause of the following exception:

ValueError                                Traceback (most recent call last)
Cell In[14], line 1
----> 1 ray_cluster_name = vertex_ray.create_ray_cluster(
      2     head_node_type=config.head_node_type,
      3     cluster_name=config.cluster_name,
      4     worker_node_types=config.worker_node_types,
      5     enable_metrics_collection=True,
      6     labels={
      7         'head': config.head_node_desc,
      8         'worker': config.worker_node_desc,
      9         'hourly_cost': f'${config.run_cost(1):.2f}'
     10     }
     11 )

File /opt/conda/lib/python3.10/site-packages/google/cloud/aiplatform/vertex_ray/cluster_init.py:299, in create_ray_cluster(head_node_type, python_version, ray_version, network, service_account, cluster_name, worker_node_types, custom_images, enable_metrics_collection, labels)
    297     _ = client.create_persistent_resource(request)
    298 except Exception as e:
--> 299     raise ValueError("Failed in cluster creation due to: ", e) from e
    301 # Get persisent resource
    302 cluster_resource_name = f"{parent}/persistentResources/{cluster_name}"

ValueError: ('Failed in cluster creation due to: ', InvalidArgument('List of found errors:\t1.Field: persistent_resource_id; Message: persistent_resource_id can only contain numbers, lower case letters and hyphens. The first character must be a letter, and the last character must be a letter or number.\t2.Field: persistent_resource.labels; Message: There can be no more than 64 labels attached to a single resource. Label keys and values can only contain lowercase letters, numbers, dashes and underscores. Label keys must start with a letter or number, must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8. Label values must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8.\t'))
```
{collapsible="true" collapsed-title="vertex_ray.create_ray_cluster error"}

The error message is 
```text
List of found errors:
    1.Field: persistent_resource_id; 
      Message: persistent_resource_id can only contain numbers, 
      lower case letters and hyphens. The first character must be a 
      letter, and the last character must be a letter or number.
    2.Field: persistent_resource.labels; 
      Message: There can be no more than 64 labels attached to a 
      single resource. Label keys and values can only contain lowercase 
      letters, numbers, dashes and underscores. Label keys must 
      start with a letter or number, must be less than 64 
      characters in length, and must be less that 128 bytes in 
      length when encoded in UTF-8. Label values must be less than 
      64 characters in length, and must be less that 128 bytes in length
      when encoded in UTF-8.
```
There are two errors here, actually. The first are underscores in the 
cluster name. The second is that the `labels` dictionary is not accepted.

The first problem can be fixed by removing underscores from the cluster
name. The second one, passing a `labels` dictionary with keys and values
that start with a letter or number does not work.

This problem also goes away if I don't pass a custom `config.cluster_name`
and I do not specify labels.

### Code generated labels

```python
labels={
        'head': config.head_node_desc,
        'worker': config.worker_node_desc,
        'hourly_cost': f'${config.run_cost(1):.2f}'
    }

labels
{'head': 'h4vcpu', 'worker': 'w3x4vcpu1xt4', 'hourly_cost': '$1.27'}
```
Clearly there are only three labels. Passing the labels dictionary to create ray cluster raises ValueError

```python
%%time
ray_cluster_name = vertex_ray.create_ray_cluster(
    head_node_type=config.head_node_type,
    worker_node_types=config.worker_node_types,
    enable_metrics_collection=True,
    labels=labels
)
---------------------------------------------------------------------------
_InactiveRpcError                         Traceback (most recent call last)
File /opt/conda/lib/python3.10/site-packages/google/api_core/grpc_helpers.py:65, in _wrap_unary_errors.<locals>.error_remapped_callable(*args, **kwargs)
     64 try:
---> 65     return callable_(*args, **kwargs)
     66 except grpc.RpcError as exc:

File /opt/conda/lib/python3.10/site-packages/grpc/_channel.py:1181, in _UnaryUnaryMultiCallable.__call__(self, request, timeout, metadata, credentials, wait_for_ready, compression)
   1175 (
   1176     state,
   1177     call,
   1178 ) = self._blocking(
   1179     request, timeout, metadata, credentials, wait_for_ready, compression
   1180 )
-> 1181 return _end_unary_response_blocking(state, call, False, None)

File /opt/conda/lib/python3.10/site-packages/grpc/_channel.py:1006, in _end_unary_response_blocking(state, call, with_call, deadline)
   1005 else:
-> 1006     raise _InactiveRpcError(state)

_InactiveRpcError: <_InactiveRpcError of RPC that terminated with:
	status = StatusCode.INVALID_ARGUMENT
	details = "List of found errors:	1.Field: persistent_resource.labels; Message: There can be no more than 64 labels attached to a single resource. Label keys and values can only contain lowercase letters, numbers, dashes and underscores. Label keys must start with a letter or number, must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8. Label values must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8.	"
	debug_error_string = "UNKNOWN:Error received from peer ipv4:142.250.152.95:443 {created_time:"2024-07-02T16:15:30.545160262+00:00", grpc_status:3, grpc_message:"List of found errors:\t1.Field: persistent_resource.labels; Message: There can be no more than 64 labels attached to a single resource. Label keys and values can only contain lowercase letters, numbers, dashes and underscores. Label keys must start with a letter or number, must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8. Label values must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8.\t"}"
>

The above exception was the direct cause of the following exception:

InvalidArgument                           Traceback (most recent call last)
File /opt/conda/lib/python3.10/site-packages/google/cloud/aiplatform/vertex_ray/cluster_init.py:297, in create_ray_cluster(head_node_type, python_version, ray_version, network, service_account, cluster_name, worker_node_types, custom_images, enable_metrics_collection, labels)
    296 try:
--> 297     _ = client.create_persistent_resource(request)
    298 except Exception as e:

File /opt/conda/lib/python3.10/site-packages/google/cloud/aiplatform_v1/services/persistent_resource_service/client.py:867, in PersistentResourceServiceClient.create_persistent_resource(self, request, parent, persistent_resource, persistent_resource_id, retry, timeout, metadata)
    866 # Send the request.
--> 867 response = rpc(
    868     request,
    869     retry=retry,
    870     timeout=timeout,
    871     metadata=metadata,
    872 )
    874 # Wrap the response in an operation future.

File /opt/conda/lib/python3.10/site-packages/google/api_core/gapic_v1/method.py:113, in _GapicCallable.__call__(self, timeout, retry, *args, **kwargs)
    111     kwargs["metadata"] = metadata
--> 113 return wrapped_func(*args, **kwargs)

File /opt/conda/lib/python3.10/site-packages/google/api_core/grpc_helpers.py:67, in _wrap_unary_errors.<locals>.error_remapped_callable(*args, **kwargs)
     66 except grpc.RpcError as exc:
---> 67     raise exceptions.from_grpc_error(exc) from exc

InvalidArgument: 400 List of found errors:	1.Field: persistent_resource.labels; Message: There can be no more than 64 labels attached to a single resource. Label keys and values can only contain lowercase letters, numbers, dashes and underscores. Label keys must start with a letter or number, must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8. Label values must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8.	 [field_violations {
  field: "persistent_resource.labels"
  description: "There can be no more than 64 labels attached to a single resource. Label keys and values can only contain lowercase letters, numbers, dashes and underscores. Label keys must start with a letter or number, must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8. Label values must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8."
}
]

The above exception was the direct cause of the following exception:

ValueError                                Traceback (most recent call last)
File <timed exec>:1

File /opt/conda/lib/python3.10/site-packages/google/cloud/aiplatform/vertex_ray/cluster_init.py:299, in create_ray_cluster(head_node_type, python_version, ray_version, network, service_account, cluster_name, worker_node_types, custom_images, enable_metrics_collection, labels)
    297     _ = client.create_persistent_resource(request)
    298 except Exception as e:
--> 299     raise ValueError("Failed in cluster creation due to: ", e) from e
    301 # Get persisent resource
    302 cluster_resource_name = f"{parent}/persistentResources/{cluster_name}"

ValueError: ('Failed in cluster creation due to: ',  InvalidArgument('List of found errors:\t1.Field: persistent_resource.labels; Message: There can be no more than 64 labels attached to a single resource. Label keys and values can only contain lowercase letters, numbers, dashes and underscores. Label keys must start with a letter or number, must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8. Label values must be less than 64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8.\t'))
```
{collapsible="true" collapsed-title="vertex_ray.create_ray_cluster labels"}

This error message does not make any sense either - **64+ labels**?:
```text
List of found errors:
    1.Field: persistent_resource.labels; 
    Message: There can be no more than 64 labels attached to a single 
    resource. Label keys and values can only contain lowercase letters, 
    numbers, dashes and underscores. Label keys must start with a 
    letter or number, must be less than 64 characters in length, 
    and must be less that 128 bytes in length when encoded in UTF-8. 
    Label values must be less than 64 characters in length, and must 
    be less that 128 bytes in length when encoded in UTF-8.
```
This is total BS, underscores are not allowed.

To summarize:

```text
List of found errors:
    1.Field: persistent_resource_id; Message: persistent_resource_id can only contain numbers, 
    lower case letters and hyphens. The first character must be a letter, and the last character
    must be a letter or number.
    2.Field: persistent_resource.labels; Message: There can be no more than 64 labels attached 
    to a single resource. Label keys and values can only contain lowercase letters, numbers, 
    dashes and underscores. Label keys must start with a letter or number, must be less than 
    64 characters in length, and must be less that 128 bytes in length when encoded in UTF-8. 
    Label values must be less than 64 characters in length, and must be less that 128 bytes 
    in length when encoded in UTF-8.
```

This error message does not make any sense, so dropping labels.

### Min Head node size

We keep default for cluster_name and no labels. 
Use `n1-standard-4` for head

```python
config = RayClusterConfig(
        head_n_cores=4,
        worker_n_cores=4,
        worker_gpu='T4',
        worker_node_count=3,
        accelerator_count=1
    )
```

There is a min memory requirement on the head node for T4 accelerators

```python
%%time
ray_cluster_name = vertex_ray.create_ray_cluster(
    head_node_type=config.head_node_type,
    worker_node_types=config.worker_node_types,
    enable_metrics_collection=True,
)
---------------------------------------------------------------------------
_InactiveRpcError                         Traceback (most recent call last)
File /opt/conda/lib/python3.10/site-packages/google/api_core/grpc_helpers.py:65, in _wrap_unary_errors.<locals>.error_remapped_callable(*args, **kwargs)
     64 try:
---> 65     return callable_(*args, **kwargs)
     66 except grpc.RpcError as exc:

File /opt/conda/lib/python3.10/site-packages/grpc/_channel.py:1181, in _UnaryUnaryMultiCallable.__call__(self, request, timeout, metadata, credentials, wait_for_ready, compression)
   1175 (
   1176     state,
   1177     call,
   1178 ) = self._blocking(
   1179     request, timeout, metadata, credentials, wait_for_ready, compression
   1180 )
-> 1181 return _end_unary_response_blocking(state, call, False, None)

File /opt/conda/lib/python3.10/site-packages/grpc/_channel.py:1006, in _end_unary_response_blocking(state, call, with_call, deadline)
   1005 else:
-> 1006     raise _InactiveRpcError(state)

_InactiveRpcError: <_InactiveRpcError of RPC that terminated with:
	status = StatusCode.INVALID_ARGUMENT
	details = "Ray head node machine type's memory is too small. Please use a machine type with larger than 18GB of memory for the Ray cluster's head node."
	debug_error_string = "UNKNOWN:Error received from peer ipv4:64.233.181.95:443 {created_time:"2024-07-02T16:25:32.465440065+00:00", grpc_status:3, grpc_message:"Ray head node machine type\'s memory is too small. Please use a machine type with larger than 18GB of memory for the Ray cluster\'s head node."}"
>

The above exception was the direct cause of the following exception:

InvalidArgument                           Traceback (most recent call last)
File /opt/conda/lib/python3.10/site-packages/google/cloud/aiplatform/vertex_ray/cluster_init.py:297, in create_ray_cluster(head_node_type, python_version, ray_version, network, service_account, cluster_name, worker_node_types, custom_images, enable_metrics_collection, labels)
    296 try:
--> 297     _ = client.create_persistent_resource(request)
    298 except Exception as e:

File /opt/conda/lib/python3.10/site-packages/google/cloud/aiplatform_v1/services/persistent_resource_service/client.py:867, in PersistentResourceServiceClient.create_persistent_resource(self, request, parent, persistent_resource, persistent_resource_id, retry, timeout, metadata)
    866 # Send the request.
--> 867 response = rpc(
    868     request,
    869     retry=retry,
    870     timeout=timeout,
    871     metadata=metadata,
    872 )
    874 # Wrap the response in an operation future.

File /opt/conda/lib/python3.10/site-packages/google/api_core/gapic_v1/method.py:113, in _GapicCallable.__call__(self, timeout, retry, *args, **kwargs)
    111     kwargs["metadata"] = metadata
--> 113 return wrapped_func(*args, **kwargs)

File /opt/conda/lib/python3.10/site-packages/google/api_core/grpc_helpers.py:67, in _wrap_unary_errors.<locals>.error_remapped_callable(*args, **kwargs)
     66 except grpc.RpcError as exc:
---> 67     raise exceptions.from_grpc_error(exc) from exc

InvalidArgument: 400 Ray head node machine type's memory is too small. Please use a machine type with larger than 18GB of memory for the Ray cluster's head node.

The above exception was the direct cause of the following exception:

ValueError                                Traceback (most recent call last)
File <timed exec>:1

File /opt/conda/lib/python3.10/site-packages/google/cloud/aiplatform/vertex_ray/cluster_init.py:299, in create_ray_cluster(head_node_type, python_version, ray_version, network, service_account, cluster_name, worker_node_types, custom_images, enable_metrics_collection, labels)
    297     _ = client.create_persistent_resource(request)
    298 except Exception as e:
--> 299     raise ValueError("Failed in cluster creation due to: ", e) from e
    301 # Get persisent resource
    302 cluster_resource_name = f"{parent}/persistentResources/{cluster_name}"

ValueError: ('Failed in cluster creation due to: ', InvalidArgument("Ray head node machine type's memory is too small. Please use a machine type with larger than 18GB of memory for the Ray cluster's head node."))
```
{collapsible="true" collapsed-title="vertex_ray.create_ray_cluster small head error"}

The error is

```text
Ray head node machine type's memory is too small. 
Please use a machine type with larger than 18GB of memory for the 
Ray cluster's head node.
```

This is not documented.

### GPU Quota for `n1-standard-8`

Specifying a bigger node with 8 vCPUs and 30 GB of RAM, 
`n1-standard-8` can be achieved with the following config

```python
config = RayClusterConfig(
        head_n_cores=8,
        worker_n_cores=4,
        worker_gpu='T4',
        worker_node_count=3,
        accelerator_count=1
    )
```

Create a Ray cluster with the following command where initially 
everything looks promising:

```python
%%time
ray_cluster_name = vertex_ray.create_ray_cluster(
    head_node_type=config.head_node_type,
    worker_node_types=config.worker_node_types,
    enable_metrics_collection=True,
)
[Ray on Vertex AI]: Cluster State = State.PROVISIONING
Waiting for cluster provisioning; attempt 1; sleeping for 0:02:30 seconds
```

However, after several minutes of running, the following RuntimeError gets triggered

```python
ERROR:root:[Ray on Vertex AI]: The following quotas are exceeded: CustomModelTrainingT4GPUsPerProjectPerRegion
---------------------------------------------------------------------------
RuntimeError                              Traceback (most recent call last)
File <timed exec>:1

File /opt/conda/lib/python3.10/site-packages/google/cloud/aiplatform/vertex_ray/cluster_init.py:303, in create_ray_cluster(head_node_type, python_version, ray_version, network, service_account, cluster_name, worker_node_types, custom_images, enable_metrics_collection, labels)
    301 # Get persisent resource
    302 cluster_resource_name = f"{parent}/persistentResources/{cluster_name}"
--> 303 response = _gapic_utils.get_persistent_resource(
    304     persistent_resource_name=cluster_resource_name,
    305     tolerance=1,  # allow 1 retry to avoid get request before creation
    306 )
    307 return response.name

File /opt/conda/lib/python3.10/site-packages/google/cloud/aiplatform/vertex_ray/util/_gapic_utils.py:113, in get_persistent_resource(persistent_resource_name, tolerance)
    111 if response.error.message:
    112     logging.error("[Ray on Vertex AI]: %s" % response.error.message)
--> 113     raise RuntimeError("[Ray on Vertex AI]: Cluster returned an error.")
    115 print("[Ray on Vertex AI]: Cluster State =", response.state)
    116 if response.state == PersistentResource.State.RUNNING:

RuntimeError: [Ray on Vertex AI]: Cluster returned an error.
```
{collapsible="true" collapsed-title="CustomModelTrainingT4GPUsPerProjectPerRegion"}

The exception is `CustomModelTrainingT4GPUsPerProjectPerRegion`. That sounds pretty
specific and pretty important. Try googling that
![](jfgi_CustomModelTrainingT4GPUsPerProjectPerRegion.png){width="450" style="block"}
It is an understatement to declare the search results overwhelming.

Going back to the error message, it is not very useful, but the first line indicates 
there might is a GPU quota problem.

Turns out the overall quota for T4s was 1, so provisioning 3 was a bit much.

I would expect a different message, even something like: "Hey genius, you are
trying to use 3 GPUs, but your quota is only 1. WTF? Go figure it out."

![](T4_initial_quota.png){width="450" style="block"}

While I was at it, I also requested 4 P4s and 4 V100s. Google wanted my phone number, probably to check on me
![](3_quota_request.png){width="450" style="block"}

Received instantaneous response from `cloudquota@google.com` about my request
![](3_quota_request_email_conf.png){width="450" style="block"}
Seemed like I was talking to the right person, who at least listened. It also seemed
that the quota was instantaneously upped, but that was also BS. It takes a few days
to get an official email by a live person stating you are good to go. Automated,
that is AI generated, responses are totally useless because all these problems persisted.

Then another email from my favorite `#cloudquota` arrived in a rapid fire stating that 
my request was approved, that I had access to 4 TPUs anywhere in the world, not only in Iowa:
![](3_quota_request_approval.png){width="450" style="block"}
I got excited and immediately restarted the ray cluster, with the same results.

At that point I went to bed, waiting for the 15 min approval time window to kick in.

The next morning, I retried. No change.

In reality, that 15 minutes turned into a week. Once 



and then I was good to go.

### Ray cluster, no GPUs

Use the following config to establish a ray cluster w/o accelerators

```python
config = RayClusterConfig(
        head_n_cores=8,
        worker_n_cores=4,
        worker_gpu=None,
        worker_node_count=3,
        accelerator_count=1
    )
```
There is no collision b/w woker_gpu and accelerator_count, because if worker_gpu is None, accelerator_count 
internally gets set to 0.

I was able to start the cluster after 22 min:
```python
%%time
ray_cluster_name = vertex_ray.create_ray_cluster(
    head_node_type=config.head_node_type,
    worker_node_types=config.worker_node_types,
    enable_metrics_collection=True,
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
[Ray on Vertex AI]: Cluster State = State.RUNNING
CPU times: user 1.72 s, sys: 521 ms, total: 2.24 s
Wall time: 22min 6s
```
{collapsible="true" collapsed-title="Start Ray cluster, no GPUs"}

Checking the ray cluster status in the console, I was happy to see that at least one of them was 
working - the one with no GPUs

![](ray_dashboard_no_GPUs.png){width="450" style="block"}

Opening the only healthy ray cluster in Colab Enterprise, I was presented with useful code:

```python
! pip install "google-cloud-aiplatform>=1.51"

from google.cloud import aiplatform
from google.cloud.aiplatform.preview import vertex_ray

aiplatform.init(project='ray-whisper', location='us-central1')

# Initialize connection to the Ray cluster on Vertex AI.
ray.init(address='vertex_ray://projects/702949608356/locations/us-central1/persistentResources/ray-cluster-2024-07-02-15-33-50-05879')

```

The ray cluster address is the same string I got when I executed `vertex_ray.create_ray_cluster` in the first
Vertex AI jupyter notebook

```python
print(ray_cluster_name)
projects/702949608356/locations/us-central1/persistentResources/ray-cluster-2024-07-02-15-33-50-05879
```

Upon cluster init, I get the following output in the colab enterprise notebook

```python
# Initialize connection to the Ray cluster on Vertex AI.
ray.init(address='vertex_ray://projects/702949608356/locations/us-central1/persistentResources/ray-cluster-2024-07-02-15-33-50-05879')
[Ray on Vertex AI]: Cluster State = State.RUNNING
---------------------------------------------------------------------------
ConnectionError                           Traceback (most recent call last)
<ipython-input-8-1cbbda6a776a> in <cell line: 2>()
      1 # Initialize connection to the Ray cluster on Vertex AI.
----> 2 ray.init(address='vertex_ray://projects/702949608356/locations/us-central1/persistentResources/ray-cluster-2024-07-02-15-33-50-05879')

8 frames
/usr/local/lib/python3.10/dist-packages/ray/util/client/worker.py in _connect_channel(self, reconnecting)
    258                     "more information."
    259                 )
--> 260             raise ConnectionError("ray client connection timeout")
    261 
    262     def _can_reconnect(self, e: grpc.RpcError) -> bool:

ConnectionError: ray client connection timeout
```
Short and sweet exception that I cannot connect to the Ray cluster with
Google-provided boilerplate code. Fuck me.

I went back to the jupyter notebook on my laptop to init the ray connection 

```python
ray.init(address=f'vertex_ray://{ray_cluster_name}')
/opt/conda/lib/python3.10/site-packages/ray/util/client/worker.py:253: UserWarning: Ray Client connection timed out. Ensure that the Ray Client port on the head node is reachable from your local machine. See https://docs.ray.io/en/latest/cluster/ray-client.html#step-2-check-ports for more information.
  warnings.warn(
---------------------------------------------------------------------------
ConnectionError                           Traceback (most recent call last)
Cell In[32], line 1
----> 1 ray.init(address=f'vertex_ray://{ray_cluster_name}')

File /opt/conda/lib/python3.10/site-packages/ray/_private/client_mode_hook.py:103, in client_mode_hook.<locals>.wrapper(*args, **kwargs)
    101     if func.__name__ != "init" or is_client_mode_enabled_by_default:
    102         return getattr(ray, func.__name__)(*args, **kwargs)
--> 103 return func(*args, **kwargs)

File /opt/conda/lib/python3.10/site-packages/ray/_private/worker.py:1430, in init(address, num_cpus, num_gpus, resources, labels, object_store_memory, local_mode, ignore_reinit_error, include_dashboard, dashboard_host, dashboard_port, job_config, configure_logging, logging_level, logging_format, log_to_driver, namespace, runtime_env, storage, **kwargs)
   1428 passed_kwargs.update(kwargs)
   1429 builder._init_args(**passed_kwargs)
-> 1430 ctx = builder.connect()
   1431 from ray._private.usage import usage_lib
   1433 if passed_kwargs.get("allow_multiple") is True:

File /opt/conda/lib/python3.10/site-packages/google/cloud/aiplatform/vertex_ray/client_builder.py:171, in VertexRayClientBuilder.connect(self)
    166     bearer_token = _validation_utils.get_bearer_token()
    167     self._metadata = [
    168         ("authorization", "Bearer {}".format(bearer_token)),
    169         ("x-goog-user-project", "{}".format(initializer.global_config.project)),
    170     ]
--> 171 ray_client_context = super().connect()
    172 ray_head_uris = self.response.resource_runtime.access_uris
    174 # Valid resource name (reference public doc for public release):
    175 # "projects/<project_num>/locations/<region>/persistentResources/<pr_id>"

File /opt/conda/lib/python3.10/site-packages/ray/client_builder.py:173, in ClientBuilder.connect(self)
    170 if self._allow_multiple_connections:
    171     old_ray_cxt = ray.util.client.ray.set_context(None)
--> 173 client_info_dict = ray.util.client_connect.connect(
    174     self.address,
    175     job_config=self._job_config,
    176     _credentials=self._credentials,
    177     ray_init_kwargs=self._remote_init_kwargs,
    178     metadata=self._metadata,
    179 )
    181 dashboard_url = ray.util.client.ray._get_dashboard_url()
    183 cxt = ClientContext(
    184     dashboard_url=dashboard_url,
    185     python_version=client_info_dict["python_version"],
   (...)
    190     _context_to_restore=ray.util.client.ray.get_context(),
    191 )

File /opt/conda/lib/python3.10/site-packages/ray/util/client_connect.py:55, in connect(conn_str, secure, metadata, connection_retries, job_config, namespace, ignore_version, _credentials, ray_init_kwargs)
     50 _explicitly_enable_client_mode()
     52 # TODO(barakmich): https://github.com/ray-project/ray/issues/13274
     53 # for supporting things like cert_path, ca_path, etc and creating
     54 # the correct metadata
---> 55 conn = ray.connect(
     56     conn_str,
     57     job_config=job_config,
     58     secure=secure,
     59     metadata=metadata,
     60     connection_retries=connection_retries,
     61     namespace=namespace,
     62     ignore_version=ignore_version,
     63     _credentials=_credentials,
     64     ray_init_kwargs=ray_init_kwargs,
     65 )
     66 return conn

File /opt/conda/lib/python3.10/site-packages/ray/util/client/__init__.py:250, in RayAPIStub.connect(self, *args, **kw_args)
    248 def connect(self, *args, **kw_args):
    249     self.get_context()._inside_client_test = self._inside_client_test
--> 250     conn = self.get_context().connect(*args, **kw_args)
    251     global _lock, _all_contexts
    252     with _lock:

File /opt/conda/lib/python3.10/site-packages/ray/util/client/__init__.py:92, in _ClientContext.connect(self, conn_str, job_config, secure, metadata, connection_retries, namespace, ignore_version, _credentials, ray_init_kwargs)
     89 setup_logger(logging_level, logging_format)
     91 try:
---> 92     self.client_worker = Worker(
     93         conn_str,
     94         secure=secure,
     95         _credentials=_credentials,
     96         metadata=metadata,
     97         connection_retries=connection_retries,
     98     )
     99     self.api.worker = self.client_worker
    100     self.client_worker._server_init(job_config, ray_init_kwargs)

File /opt/conda/lib/python3.10/site-packages/ray/util/client/worker.py:139, in Worker.__init__(self, conn_str, secure, metadata, connection_retries, _credentials)
    136 # Set to True after initial connection succeeds
    137 self._has_connected = False
--> 139 self._connect_channel()
    140 self._has_connected = True
    142 # Has Ray been initialized on the server?

File /opt/conda/lib/python3.10/site-packages/ray/util/client/worker.py:260, in Worker._connect_channel(self, reconnecting)
    252 if log_once("ray_client_security_groups"):
    253     warnings.warn(
    254         "Ray Client connection timed out. Ensure that "
    255         "the Ray Client port on the head node is reachable "
   (...)
    258         "more information."
    259     )
--> 260 raise ConnectionError("ray client connection timeout")

ConnectionError: ray client connection timeout
```
{collapsible="true" collapsed-title="ray client connection timeout"}

I get the same ConnectionError exception, in the same line of code in `worker.py`, but this one 
gives more contex through a warning

```text

Ray Client connection timed out. Ensure that the Ray Client 
port on the head node is reachable from your local machine. 
See 
[https://docs.ray.io/en/latest/cluster/ray-client.html#step-2-check-ports](https://docs.ray.io/en/latest/cluster/ray-client.html#step-2-check-ports)
for more information.
```
That looks much better. I need to tweak ports configuration on the head node
by following
[https://docs.ray.io/en/latest/cluster/ray-client.html#step-2-check-ports](https://docs.ray.io/en/latest/cluster/ray-client.html#step-2-check-ports)
and I should be OK.

Unless the document that may potentially help does not exist:

![](ray_ports_404.png){width="450" style="block"}
I followed the tip:
`You can try to navigate to 
[the index page](https://docs.ray.io/en/latest/) of the project and use its navigation, 
or search for a similar page.` did not help either.
On the index page, I searched for `head node port ray client`
![](head_node_port_ray_client.png){width="450" style="block"}

The solution was to ignore the misleading import from the Enterprise Colab
```python
from google.cloud.aiplatform.preview import vertex_ray
```
and go straight for
```python
import vertex_ray
```
I found that by accident, going through the source code,
executing lines of code from `google-cloud-vertexai` SDK in jupyter lab,
desperately comparing code line by line. Fuck me, again.

### Start ray with 1 worker and 1 node

## Start cluster without an authorized account

Code to start the server
```python
%%time
t_start = time.time()
try:
    ray_cluster_name = vertex_ray.create_ray_cluster(
        head_node_type=config.head_node_type,
        cluster_name=config.cluster_name,
        worker_node_types=config.worker_node_types,
        enable_metrics_collection=True,
        custom_images=config.custom_images,
        # labels=labels
    )
finally:
    t_end = time.time()
    delta_t = t_end - t_start
    print(f'{delta_t = }')
```

Output
```python
WARNING:google.auth.compute_engine._metadata:Compute Engine Metadata server unavailable on attempt 1 of 3. Reason: timed out
WARNING:google.auth.compute_engine._metadata:Compute Engine Metadata server unavailable on attempt 2 of 3. Reason: timed out
WARNING:google.auth.compute_engine._metadata:Compute Engine Metadata server unavailable on attempt 3 of 3. Reason: [Errno 64] Host is down
delta_t = 6.020592927932739
---------------------------------------------------------------------------
DefaultCredentialsError                   Traceback (most recent call last)
File <timed exec>:3

File ~/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages/google/cloud/aiplatform/vertex_ray/cluster_init.py:286, in create_ray_cluster(head_node_type, python_version, ray_version, network, service_account, cluster_name, worker_node_types, custom_images, enable_metrics_collection, labels)
    284 location = initializer.global_config.location
    285 project_id = initializer.global_config.project
--> 286 project_number = resource_manager_utils.get_project_number(project_id)
    288 parent = f"projects/{project_number}/locations/{location}"
    289 request = persistent_resource_service.CreatePersistentResourceRequest(
    290     parent=parent,
    291     persistent_resource=persistent_resource,
    292     persistent_resource_id=cluster_name,
    293 )

File ~/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages/google/cloud/aiplatform/utils/resource_manager_utils.py:70, in get_project_number(project_id, credentials)
     53 def get_project_number(
     54     project_id: str,
     55     credentials: Optional[auth_credentials.Credentials] = None,
     56 ) -> str:
     57     """Gets project ID given the project number
     58 
     59     Args:
   (...)
     67 
     68     """
---> 70     credentials = credentials or initializer.global_config.credentials
     72     projects_client = resourcemanager.ProjectsClient(credentials=credentials)
     74     project = projects_client.get_project(name=f"projects/{project_id}")

File ~/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages/google/cloud/aiplatform/initializer.py:348, in _Config.credentials(self)
    346 logging_warning_filter = utils.LoggingFilter(logging.WARNING)
    347 logger.addFilter(logging_warning_filter)
--> 348 self._set_project_as_env_var_or_google_auth_default()
    349 credentials = self._credentials
    350 logger.removeFilter(logging_warning_filter)

File ~/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages/google/cloud/aiplatform/initializer.py:99, in _Config._set_project_as_env_var_or_google_auth_default(self)
     96         self._project = project
     98 if not self._credentials:
---> 99     credentials, _ = google.auth.default()
    100     self._credentials = credentials

File ~/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages/google/auth/_default.py:691, in default(scopes, request, quota_project_id, default_scopes)
    683             _LOGGER.warning(
    684                 "No project ID could be determined. Consider running "
    685                 "`gcloud config set project` or setting the %s "
    686                 "environment variable",
    687                 environment_vars.PROJECT,
    688             )
    689         return credentials, effective_project_id
--> 691 raise exceptions.DefaultCredentialsError(_CLOUD_SDK_MISSING_CREDENTIALS)

DefaultCredentialsError: Your default credentials were not found. To set up Application Default Credentials, see https://cloud.google.com/docs/authentication/external/set-up-adc for more information.
```
{collapsible="true" collapsed-title="Ray missing default credentials"}

Pretty decent error message. There is a warning that I did not set a project or 
that there is no environment variable set. Again, total BS because I did more than once
and I can prove it
```Bash
$ echo $PROJECT_ID
ray-whisper
```
I fucked with credentials for the AI Training
Service Account quite a bit, so I went back to see how I can unfuck it by reworking the
[ADC setup](https://cloud.google.com/docs/authentication/external/set-up-adc).

That was easily correctable by service account impersonation.

## You cannot kill the Ray server while it is spawning

You have to wait for the server to spin up, then you can delete it in the Vertex
AI console. 

## Enterprise Colab is slow AF, still cannot connect to Ray Cluster

Once the cluster is up, Vertex AI provides a link to Enterprise Colab

![](enterprise_colab.png){width="450" style="block"}

It takes about one 1min 30s to connect to the server and figuring out that
all dependencies have already been installed, even though Colab claims it 
takes only 18s:
![](connect_to_collab.png){width="450" style="block"}

## Potential Versioning Snafu

The 
[Medium article](https://medium.com/google-cloud/whisper-goes-to-wall-street-scaling-speech-to-text-with-ray-on-vertex-ai-part-i-04c8ceb180c0)
Docker image is built from `us-docker.pkg.dev/vertex-ai/training/ray-gpu.2-9.py310:latest`
ray 2.9. On the other hand, the `requirements.txt` file specifies
```text
ray==2.10.0
ray[data]==2.10.0
ray[train]==2.10.0
```

## Unable to start a cluster with `n1-standard-8` node

Could not find in the documentation why, but it is impossible use
`n1-standard-4` on the head node.

## Outdated documentation

The
[documentation](https://cloud.google.com/vertex-ai/docs/open-source/ray-on-vertex-ai/create-cluster#ray-on-vertex-ai-sdk)
is not up-to-date. The last version of `create_ray_cluster` does not have
`service_account`.
