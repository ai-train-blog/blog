# Summary

According to public cloud providers, public clouds 
provide practically infinite compute, networking and storage resources with
endless optimization combinations for latency, scaling and availability.
That value proposition is especially appealing to data scientists who
would like to and should be able to pick any ML model, regardless of 
size or complexity, feed any size or type of data and train it.

Here we make a distinction between what a data scientist can control:

1. Python version (latest vs. stable);
2. Development environment;
3. AI/ML model.

Typically, the quality of an AI/ML model can be assessed in a limited way,
constrained to the practitioner's local environment, i.e. laptop. Only in that
constrained scenario data scientists have the freedom to operate.

In order to demonstrate the full value of AI/ML models, data scientists
are constrained by:

1. Financial backing of their parent company (FinOps)
2. Security practice (SecOps)
3. Infrastructure provisioning (DevOps)
4. Infrastructure utilization (MLOps)
5. Monitoring (Ops)

FinOps dictate the overall access to cloud infrastructure. 
With electricity and gas, we have unlimited supply and
we pay every month for what we use. Public cloud providers, however, are 
not utilities. Public cloud providers are more like credit card companies: 
you need to show a history of responsible behavior before you can
earn perks, like better hardware.

In this day and age, when everything in AI/ML progresses at breath-taking speed, 
having to wait to build "cloud credit history" and achieve "high cloud
credit score" with cloud providers is one of the biggest 
impediments to independent AI/ML practitioners. Only the largest companies 
with strong relationship with public cloud providers are afforded access 
to scarcely available computational resources. With such large demand for
computational resources and limited supply (just check Nvidia's market value), 
nimble AI teams are placed at considerate disadvantage. They often resort
to finding algorithms that are frugal with resource utilization (DeepSpeed),
which further widens the gap between the 

Data scientists have tenacity to push their 
models to the extreme. In order to do that, the primary dev environment
that data scientists have control over, their own laptop,
falls short on scalability. To overcome that impediment,
data scientists then assume roles of data engineers
(for data cleanup), ML engineers (for building pipelines) and cybersecurity
engineers (is it safe?). Given the abundance of available documentation,
there is no question that 
data scientist are capable of implementing the best security practices,
the best pipeline implementation practices, the best build and deployment
practices, with automated monitoring. 

Data scientists can handle all infrastructure-related requirements through 
code. Data scientists build data pipelines with code. The recommended 
implementation of production-level ML systems is to
bild a simplistic end-to-end pipeline first, deploy it with the simplest
model possible and then iterate. In this whole process, data scientist
do the least what they enjoy the most: AI training.

It is tedious but possible for data scientists to clear all operational hurdles 
in model development and training. Going through documentation and
implementing optimal resources is not difficult, but time consuming. Is it 
worth for data scientists to spend most of their time
away from the tasks they enjoy doing and should be doing in the first place?
Following documentation is not as nearly as bad as not being able to implement
what documentation specifies. It is often time the case that
data scientists have the expertise to overcome challenges they may be
facing, but do not have authority to act. What gets in their way are FinOps. 
That does not necessarily mean budgetary constraints, but also cloud providers'
imposed "cloud credit history score" which dictates what and how can be used.
It is the combination of the company budget and cloud providers' allowance
that trickles down as severely constrained authority to do anything perceived 
outside of model training under the principle of least access.

In summary, public clouds provide seemingly infinite resources for building, training
and deploying AI models. In reality, there is unreasonable demand 
imposed on data scientists to understand, develop and maintain significant
auxiliary resources just to provide the scaffolding upon which AI construction
can begin. It is very likely that the scaffolding is here to stay, just like 
medieval churches that were built for centuries.

The main question then is: how can we provide a scalable environment
for data scientists to build, train and deploy AI models without having to 
go through the rigmarole imposed by the current standard industry-wide practices? 


<seealso>
    <!--Provide links to related how-to guides, overviews, and tutorials.-->
</seealso>