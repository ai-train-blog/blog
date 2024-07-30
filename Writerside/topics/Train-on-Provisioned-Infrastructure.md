# Train on Provisioned Infrastructure

Once the infrastructure has been made available in a public cloud,
we need to connect to the head node, send the training code
to the cluster and train. The training code takes care of data loading
and transformation, as well as setting various training hyperparameters.

Before we can send the training code and 
distribute data to the cluster, we need to establish a connection
through a client. The client then can pass the command-line 
parameters to the AI training script, establish the runtime environment 
in which the code can run and connect 

<seealso>
<!--Give some related links to how-to articles-->
</seealso>
