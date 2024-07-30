# OpenAI Whisper Training on a Ray Cluster

## Intro

This project was inspired by a Medium 
[post](https://medium.com/google-cloud/whisper-goes-to-wall-street-scaling-speech-to-text-with-ray-on-vertex-ai-part-i-04c8ceb180c0) 
from 2024-06-25 on fine-tuning Whisper, an OpenAI foundational model developed for 
voice-to-text applications. 
The author discusses how to leverage a distributed infrastructure orchestrated by 
[Ray](https://www.ray.io/) 
for fine-tuning. Ray, a "unified compute framework 
that makes it easy to scale AI and Python workload" (in their own words) is suggested
as an ideal environment for this task because of the frequent `out-of-memory` errors
that the model is prone to.

Whisper is an automatic speech recognition (ASR) system 
[trained](https://cdn.openai.com/papers/whisper.pdf)
on 680,000 hours or multilingual and multitask supervised data collected from the web.
According to the Whisper
[model-card on github](https://github.com/openai/whisper/blob/main/model-card.md), 
the training data was divided as follows:
1. English audio with matched English transcripts - 438,000 hours (65%).
2. Non-English audio with English transcripts - 126,000 hours (18%).
3. Non-English audio with non-English trancripts - 117,000 hours (17%); 98 languages.

One of the benchmarks used for testing multilingual speech recognition
was [Multilingual LibriSpeech](https://www.openslr.org/94/), 
a collection of 8 languages: English, German, 
Dutch, French, Spanish, Italian, Portuguese and Polish
Whisper was shown to be the State Of The Art (SOTA) for multilingual 
speech recognition.
It was, unsurprisingly, also 
[observed](https://cdn.openai.com/papers/whisper.pdf) 
that the quality and performance
on transcription in a given language is directly correlated with the amount
of available training data. Whisper is especially suitable for 
transcription in multiple languages, and
translations from those languages to English.
While Whisper performs well transcribing and translating high-resource languages,
languages outside of Indo-European language family may be challenging.

Fine-tuning is an established method of improving pre-trained AI models. 
OpenAI 
[recommends](https://platform.openai.com/docs/guides/fine-tuning/when-to-use-fine-tuning)
first getting good results with prompt engineering before investing time and effort
into fine-tuning. Here we skip prompt engineering and get straight into fine-tuning.


## Fine Tuning Data for Whisper

Whisper's performance is not optimized for low-resource 
languages. We use the clean training data from public source provided by
[Mozilla Common Voice](https://commonvoice.mozilla.org/en/datasets).

## Whisper Model 

Whisper is a 
[transformer](https://huggingface.co/blog/fine-tune-whisper) 
based encoder-decoder model.
Whisper is a sequence-to-sequence model: it maps a sequence of 
audio spectrogram features to a sequence of text tokens. 
First, the feature extractor converts the raw audio inputs to a 
log-Mel spectrogram. The Transformer encoder then encodes the 
spectrogram to form a sequence of encoder hidden states. 
The last encoder hidden states are input to the decoder via 
 cross-attention mechanisms. The decoder autoregressively predicts 
text tokens, jointly conditional on the encoder hidden states 
and previously predicted tokens.
![](Whisper_model.png){width="650" style="block"}

The 
[Whisper models](https://github.com/openai/whisper/blob/main/model-card.md)
are capable of transcribing speech into the text the language
is spoken (automatic speech recognition, ASR) as well as into English (speech
translation).

There are 9 models of different sizes and capabilities

| Size     | Layers | Width | Heads | Parameters | English-only | Multilingual |
|:---------|:------:|:-----:|:-----:|-----------:|:------------:|:----:|
| tiny     |   4    |  384  |   6   |       39 M |      ✓       |      ✓       |
| base     |   6    | 	512  |   8   |       74 M |      ✓       |      ✓       |
| small	   |   12   |  768  |  12   |     	244 M |      ✓       |      ✓       |
| medium   |   24   | 	1024 |  16   |     769 M	 |      ✓       |      	✓      |
| large	   |   32   | 1280  |  20	  |     1550 M |      x       |      ✓       |
| large-v2 |   32   | 1280  |  20   |     1550 M |      x       |      ✓       |
| large-v3 |   32   | 1280  |  20   |     1550 M |      x       |      ✓       |

According to the 
[Medim article](https://medium.com/google-cloud/whisper-goes-to-wall-street-scaling-speech-to-text-with-ray-on-vertex-ai-part-i-04c8ceb180c0)
efficient training of large transformer models like Whisper may require a significant amount of GPU memory. 
In particular, `whisper-small`, the 244M model from the table, requires, at 
most, 36GB memory available. This memory is mainly allocated to the model 
and its training stages, which include gradient calculation (gradients), 
backward pass (activations, gradients of activations) and the optimizer 
step (for example, Adam optimizer stores not only the current gradients but
also past gradients and their squares). The author claims this may cause the 
CUDA ‘out-of-memory’ error.

The 'out-of-memory' issue may requires some form of parallelism strategies:
data parallelism, tensor parallelism, and pipeline parallelism. The 
easiest parallelism to implement is data parallelism (i.e.
[DistributedDataParallel](https://arxiv.org/pdf/2006.15704)
or DDP; in pytorch
), where each batch is processed independently by a different worker (GPU)
where model parameters are replicated. While this strategy scales well with
the number of workers, it may not solve the `out-of-memory` issue with the 
Whisper model.

## A Single GPU Training Using DeepSpeed

[DeepSpeed](https://arxiv.org/pdf/2207.00032) 
is an open-source deep learning optimization library 
developed by Microsoft for PyTorch for training massive large models 
with a single GPU. 

Although HuggingFace with DDP and Deepspeed makes distributed 
training efficient, according to the author, there is not built-in 
mechanisms for launching and managing the training processes among workers,
which is important for resource management and fault tolerance.
It is especially desirable to use a distributed processing framework
that allows scalable training across a large number of workers
that handles node failures. 


## Scalable and Fault Tolerant AI Training with Ray

OpenAI recommends prompt engineering before attempting fine-tuning.
The next step in the Whisper model optimization for low-resource languages
is fine tuning. 
Google Cloud Practice
([GCP](https://cloud.google.com))
AI/ML platform 
[Vertex AI](https://cloud.google.com/vertex-ai).
provides 
[Ray](https://www.ray.io/), 
a "unified compute framework that makes it easy to scale AI and Python workload"!
Once we start leveraging the advantages of public cloud infrastructure,
we should be able to access advantages of fault-tolerant and scalable AI
training system.

## Python Setup

AI training models are written in python. We need to provision the 
infrastructure either in GCP console through the UI, or we can provision
programmatically, also in python. Next we discuss how to use
prepare a particular version of python (3.10), how to enable and
prepare a virtual environment that contains all packages and dependencies 
required for the project. Finally, we describe how to use this
dedicated python environment in a jupyter notebook with a custom
kernel.
