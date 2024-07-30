# Train on GCP Vertex AI

## Connect to Ray Cluster
After starting a Ray cluster as previously describe, we connect
to the cluster in python with

```python
ray_cluster = vertex_ray.get_ray_cluster(ray_cluster_name)
```

From the `ray_cluster` object we can inspect the resources we have at our 
disposal. It is possible then to scale up and scale down workers.

## Instantiate Ray Client

To instantiate a Ray client, we use the dashboard Ray head address

```python
from ray.job_submission import JobSubmissionClient
client = JobSubmissionClient(address=f'vertex_ray://{ray_cluster.dashboard_address}')
```

## Prepare code

The fine-tuning code needs to be wrapped for distributed training on Ray.
We developed a `TrainConfig` class that handles all/most of the details
related to input, output, labeling and parameter configuration.

```python
@dataclass
class TrainConfig:
    model_path: str = 'mozilla-foundation/common_voice_11_0'
    model_size: str = 'tiny'
    name: str = 'sr'
    language: str = 'Serbian'
    max_input_length: int = 30
    num_proc: int = 4
    num_workers: int = 3
    use_gpu: bool = True
    do_lower_case: bool = False
    do_remove_punctuation: bool = False
    do_normalize_eval: bool = True
    per_device_train_batch_size: int = 64
    gradient_accumulation_steps: int = 1  # increase by 2x for every 2x decrease in batch size
    learning_rate: float = 1e-5
    warmup_steps: int = 500
    max_steps: int = 5000
    gradient_checkpointing: bool = True
    fp16: bool = True
    evaluation_strategy: str = "steps"
    per_device_eval_batch_size: int = 8
    predict_with_generate: bool = True
    generation_max_length: int = 225
    save_steps: int = 1000
    eval_steps: int = 1000
    logging_steps: int = 25
    load_best_model_at_end: bool = True
    metric_for_best_model: str = "wer"
    greater_is_better: bool = False
    push_to_hub: bool = False
    _train_id = None
    _timestamp = None

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

    @property
    def cache_dir(self):
        path = Path(os.path.join(os.environ['HOME'], 'data', self.model_path), 'cache_dir')
        os.makedirs(path, exist_ok=True)
        return str(path)

    @property
    def deepspeed(self):
        ds_config = os.path.join(self.src_dir, 'ds_config.json')
        return ds_config

    @property
    def experiments_folder_uri(self):
        path = Path(os.environ['BUCKET_URI'], 'experiments')
        return str(path)

    @property
    def logging_dir(self):
        return self.experiments_folder_uri

    @property
    def model_name(self):
        return f'openai/whisper-{self.model_size}'

    @property
    def models_path(self):
        path = Path(os.environ['BUCKET_URI'], 'models')
        return str(path)

    @property
    def output_dir(self):
        path = Path(os.path.join(os.environ['HOME'], 'data', self.model_path), 'output_dir')
        os.makedirs(path, exist_ok=True)
        return str(path)

    @property
    def predictions_folder_uri(self):
        path = Path(os.environ['BUCKET_URI'], 'predictions')
        return str(path)

    @property
    def src_dir(self):
        parent_dir = Path(__file__).parent.resolve()
        return str(parent_dir)

    @property
    def tensorboard_name(self):
        return f'tb-{self.train_experiment_name}-{self.timestamp}'

    @property
    def timestamp(self):
        if self._timestamp is None:
            self._timestamp = now()
        return self._timestamp

    @property
    def train_experiment_name(self):
        return f'fine-tune-whisper-{self.model_size}-{self.name}'

    @property
    def train_experiment_uri(self):
        path = Path(self.logging_dir, self.train_experiment_name)
        return str(path)

    @property
    def train_id(self):
        if self._train_id is None:
            import random
            import string
            self._train_id = ''.join(random.choices(string.ascii_lowercase + string.digits, k=3))
        return self._train_id

    @property
    def train_submission_id(self):
        return f'ray-job-{self.train_id}'
```
{collapsible="true" collapsed-title="TrainConfig"}

The config contains the default DeepSpeed settings.

We define the train entrypoint, keeping it as simple as possible considering
that `TrainConfig` does most of the heavy lifting for utilizing the 
provisioned infrastructure:

```python
train_entrypoint = f'python3 task.py --use-gpu'
```

We set up the train runtime environment 

```python
config = TrainConfig()

train_runtime_env = {
    'working_dir': config.src_dir,
    'env_vars': {
        'HF_TOKEN': os.getenv('HF_TOKEN'),
        'TORCH_NCCL_ASYNC_ERROR_HANDLING': '3'
    },
}
```
The working directory `config.src_dir` is `./src` directory in 
the repo which contains
the training code and the DeepSpeed configuration.
```Bash
$ ls -1 ./src
__init__.py
ds_config.json
task.py
train.py
```
with `task.py` the training entrypoint and `train.py` source for AI training.

## Train on Ray Cluster

The final step is to submit the entrypoint and runtime environment to 
the Ray client

```python
train_job_id = client.submit_job(
    submission_id=config.train_submission_id,
    entrypoint=train_entrypoint,
    runtime_env=train_runtime_env,
)
```

The training code should then authomatically send training artifacts to
`config.experiments_folder_uri` on Google Cloud Storage.

<seealso>
<!--Give some related links to how-to articles-->
</seealso>
