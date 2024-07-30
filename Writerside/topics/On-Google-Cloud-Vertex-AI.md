# On Google Cloud Vertex AI

[Vertex AI](https://cloud.google.com/vertex-ai/docs) 
from Google Cloud
is a fully-managed, unified artificial intelligence (AI) and machine learning (ML) 
development platform 
that provides the means to train and deploy ML models and AI applications. 
Vertex AI combines data engineering, data science, and ML engineering workflows,
enabling team collaboration 
using a common toolset.
Vertex AI has powerful graphic user interface,
command line interface tool and python a SDK. We present a brief overview on how to 
install and use those tools.

We first present GCP compute infrastructure with available Nvidia accelerators in the US.
Next, we discuss prerequisites necessary to start working
on Vertex AI, to authorize, authenticate, enable various roles 
and services and build a training Docker image. We present how to create a Ray cluster on 
Vertex AI, and how to connect to it.
We also mention snafus we came across along the way. Not all of the snafus are presented,
but some of the more painful deserved to be kept here for the posterity.
