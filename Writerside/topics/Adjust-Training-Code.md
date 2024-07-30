# Adjust Training Code

Training code does not run out of the box but requires significant 
work.

We provided a separate class to contain training configuration.
In addition, we use the Whisper fine-tunning example
[Hugging Face](https://huggingface.co/blog/fine-tune-whisper),
which we wrapped into a function `train_func`
for which we include a wrapper `task.py` which is then
used by the Ray training client.

The wrapper `task.py` defines three configurations for training on 
Ray: 
* [`ScalingConfig`](https://docs.ray.io/en/latest/train/api/doc/ray.train.ScalingConfig.html) - 
  Configuration for scaling training.
* [`RunConfig`](https://docs.ray.io/en/latest/train/api/doc/ray.train.RunConfig.html#ray.train.RunConfig)` - 
  Runtime configuration for training and tuning runs.
* [`CheckpointConfig`](https://docs.ray.io/en/latest/train/api/doc/ray.train.CheckpointConfig.html) - 
  Configurable parameters for defining the checkpointing strategy.

The wrapper also contains `main()` with command-line
argument parser, used in submitting the training job to the Ray client.

## `train.py`

Training code.

```python
import os
from pathlib import Path

import torch

import evaluate

from dataclasses import dataclass
from typing import Any, Dict, List, Union

from datasets import Audio, load_dataset, DatasetDict
from transformers import WhisperForConditionalGeneration, WhisperProcessor
from transformers import Seq2SeqTrainingArguments, Seq2SeqTrainer
from transformers.models.whisper.english_normalizer import BasicTextNormalizer

import ray.train.huggingface.transformers as ray_transformers

# Note: Heavily borrowed from
# https://github.com/huggingface/community-events/blob/main/whisper-fine-tuning-event/fine-tune-whisper-non-streaming.ipynb

MODEL_SIZES = [
    # 39M    74M      244M     769M
    'tiny', 'base', 'small', 'medium',  # english-only
    # 1550M    1550M      1550M
    'large', 'large-v2', 'large-v3'
]

REMOVE_COLUMNS = ["accent", "age", "client_id", "down_votes", "gender", "locale", "path", "segment", "up_votes"]


def now():
    import datetime
    return datetime.datetime.now().strftime('%Y-%m-%d-%H%M%S')


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


@dataclass
class DataCollatorSpeechSeq2SeqWithPadding:
    processor: Any

    def __call__(self, features: List[Dict[str, Union[List[int], torch.Tensor]]]) -> Dict[str, torch.Tensor]:
        # split inputs and labels since they have to be of different lengths and need different padding methods
        # first treat the audio inputs by simply returning torch tensors
        input_features = [{"input_features": feature["input_features"]} for feature in features]
        batch = self.processor.feature_extractor.pad(input_features, return_tensors="pt")

        # get the tokenized label sequences
        label_features = [{"input_ids": feature["labels"]} for feature in features]
        # pad the labels to max length
        labels_batch = self.processor.tokenizer.pad(label_features, return_tensors="pt")

        # replace padding with -100 to ignore loss correctly
        labels = labels_batch["input_ids"].masked_fill(labels_batch.attention_mask.ne(1), -100)

        # if bos token is appended in previous tokenization step,
        # cut bos token here as it's append later anyway
        if (labels[:, 0] == self.processor.tokenizer.bos_token_id).all().cpu().item():
            labels = labels[:, 1:]

        batch["labels"] = labels

        return batch


def train_func(params: dict):
    """
    Based on https://huggingface.co/blog/fine-tune-whisper
    :param params: dict to initialize TrainConfig
    :return: None
    """

    config = TrainConfig(**params)
    processor = WhisperProcessor.from_pretrained(
        config.model_name,
        language=config.language,
        task="transcribe"
    )
    normalizer = BasicTextNormalizer()

    def compute_metrics(pred):
        pred_ids = pred.predictions
        label_ids = pred.label_ids

        # replace -100 with the pad_token_id
        label_ids[label_ids == -100] = processor.tokenizer.pad_token_id

        # we do not want to group tokens when computing the metrics
        pred_str = processor.tokenizer.batch_decode(pred_ids, skip_special_tokens=True)
        label_str = processor.tokenizer.batch_decode(label_ids, skip_special_tokens=True)

        if config.do_normalize_eval:
            pred_str = [normalizer(pred) for pred in pred_str]
            label_str = [normalizer(label) for label in label_str]

        wer = 100 * metric.compute(predictions=pred_str, references=label_str)

        return {"wer": wer}

    def is_audio_in_length_range(length):
        return length < config.max_input_length

    def prepare_dataset(batch):
        # load and (possibly) resample audio data to 16kHz
        audio = batch["audio"]

        # compute log-Mel input features from input audio array
        batch["input_features"] = processor.feature_extractor(
            audio["array"],
            sampling_rate=audio["sampling_rate"]
        ).input_features[0]
        # compute input length of audio sample in seconds
        batch["input_length"] = len(audio["array"]) / audio["sampling_rate"]

        # optional pre-processing steps
        transcription = batch["sentence"]
        if config.do_lower_case:
            transcription = transcription.lower()
        if config.do_remove_punctuation:
            transcription = normalizer(transcription).strip()

        # encode target text to label ids
        batch["labels"] = processor.tokenizer(transcription).input_ids
        return batch

    # Load data
    common_voice = DatasetDict()
    common_voice["train"] = load_dataset(
        config.model_path,
        config.name,
        split="train+validation",
        cache_dir=config.cache_dir,
        trust_remote_code=True
    )
    common_voice["test"] = load_dataset(
        config.model_path,
        config.name,
        split="test",
        cache_dir=config.cache_dir,
        trust_remote_code=True
    )

    common_voice = common_voice.remove_columns(REMOVE_COLUMNS)
    common_voice = common_voice.cast_column("audio", Audio(sampling_rate=16000))
    common_voice = common_voice.map(
        prepare_dataset,
        remove_columns=common_voice.column_names["train"],
        num_proc=config.num_proc
    )
    common_voice["train"] = common_voice["train"].filter(
        is_audio_in_length_range,
        input_columns=["input_length"],
    )

    data_collator = DataCollatorSpeechSeq2SeqWithPadding(processor=processor)
    metric = evaluate.load(config.metric_for_best_model)

    model = WhisperForConditionalGeneration.from_pretrained(config.model_name)
    model.config.forced_decoder_ids = None
    model.config.suppress_tokens = []

    model.config.use_cache = False

    training_args = Seq2SeqTrainingArguments(
        output_dir=config.output_dir,
        per_device_train_batch_size=config.per_device_train_batch_size,
        gradient_accumulation_steps=config.gradient_accumulation_steps,
        learning_rate=config.learning_rate,
        warmup_steps=config.warmup_steps,
        max_steps=config.max_steps,
        gradient_checkpointing=config.gradient_checkpointing,
        deepspeed=config.deepspeed,
        # fp16=config.fp16,
        evaluation_strategy=config.evaluation_strategy,
        per_device_eval_batch_size=config.per_device_eval_batch_size,
        predict_with_generate=config.predict_with_generate,
        generation_max_length=config.generation_max_length,
        save_steps=config.save_steps,
        eval_steps=config.eval_steps,
        logging_steps=config.logging_steps,
        report_to=["tensorboard"],
        load_best_model_at_end=config.load_best_model_at_end,
        metric_for_best_model=config.metric_for_best_model,
        greater_is_better=config.greater_is_better,
        push_to_hub=config.push_to_hub
    )

    trainer = Seq2SeqTrainer(
        args=training_args,
        model=model,
        train_dataset=common_voice["train"],
        eval_dataset=common_voice["test"],
        data_collator=data_collator,
        compute_metrics=compute_metrics,
        tokenizer=processor.feature_extractor,
    )

    processor.save_pretrained(training_args.output_dir)

    # From Medium article
    # https://medium.com/google-cloud/whisper-goes-to-wall-street-scaling-speech-to-text-with-ray-on-vertex-ai-part-i-04c8ceb180c0
    callback = ray_transformers.RayTrainReportCallback()
    trainer.add_callback(callback)
    trainer = ray_transformers.prepare_trainer(trainer)

    trainer.train()
```
{collapsible="true" collapsed-title="train.py"}


## `task.py`

Training infrastructure utilization code.

```python
# libraries
import argparse

# training libraries
from train import train_func, TrainConfig, MODEL_SIZES

# ray libraries
import ray
import ray.train.huggingface.transformers
from ray.train import ScalingConfig, RunConfig, CheckpointConfig
from ray.train.torch import TorchTrainer


# helpers
def get_args():
    url = 'https://huggingface.co/datasets/mozilla-foundation/common_voice_11_0'
    parser = argparse.ArgumentParser(description='Fine-tuning Whisper on Ray on Vertex AI')

    parser.add_argument('--model_size', type=str, default='mini', choices=MODEL_SIZES, help='whisper model size')
    parser.add_argument('--name', type=str, default='sr', help=f'language config name; see {url}')
    parser.add_argument('--language', type=str, default='Serbian', help=f'language, see {url}')

    # some gemma parameters
    parser.add_argument("--per_device_train_batch_size", type=int, default=64, help="train batch size")
    parser.add_argument("--gradient_accumulation_steps", type=int, default=1,
                        help="gradient accumulation steps; increase by 2x for every 2x decrease in batch size")
    parser.add_argument("--learning_rate", type=float, default=1e-5, help="learning rate")
    parser.add_argument("--max_steps", type=int, default=5000, help="max steps")
    parser.add_argument("--per_device_eval_batch_size", type=int, default=8, help="eval batch size")
    parser.add_argument("--save_steps", type=int, default=1000, help="save steps")
    parser.add_argument("--logging_steps", type=int, default=25, help="logging steps")

    # ray parameters
    parser.add_argument('--num-workers', dest='num_workers', type=int, default=3, help='Number of workers')
    parser.add_argument('--use-gpu', dest='use_gpu', action='store_true', default=False, help='Use GPU')

    args = parser.parse_args()
    return args


def main():
    args = get_args()
    params = vars(args)
    print(f'params = {params}')

    config = TrainConfig(**params)

    print(f'config = {config!s}')

    # initialize ray session
    ray.init()

    # training config
    train_loop_config = {
        "per_device_train_batch_size": config.per_device_train_batch_size,
        "per_device_eval_batch_size": config.per_device_eval_batch_size,
        "gradient_accumulation_steps": config.gradient_accumulation_steps,
        "learning_rate": config.learning_rate,
        "max_steps": config.max_steps,
        "save_steps": config.save_steps,
        "logging_steps": config.logging_steps,
    }
    scaling_config = ScalingConfig(
        num_workers=config.num_workers,
        use_gpu=config.use_gpu
    )
    run_config = RunConfig(
        checkpoint_config=CheckpointConfig(
            num_to_keep=5,
            checkpoint_score_attribute="loss",
            checkpoint_score_order="min"),
        storage_path=config.logging_dir,
        name=config.train_experiment_name)

    trainer = TorchTrainer(
        train_loop_per_worker=train_func,
        train_loop_config=train_loop_config,
        run_config=run_config,
        scaling_config=scaling_config
    )
    # train
    trainer.fit()


if __name__ == '__main__':
    main()

```
{collapsible="true" collapsed-title="task.py"}
