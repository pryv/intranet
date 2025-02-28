|         |                       |
| ------- | --------------------- |
| Author  | Kaspar Schiess (Pryv) |
| Version | 1 (10.11.2017)        |

Pryv Cluster Management
{: .doc_title} 

Options and Opportunities, Summing up our current knowledge
{: .doc_subtitle} 



# Summary

We've experimented with Nomad for two reasons: a) to supercharge Pryv updates during development, b) to allow for large-scale Pryv developments. Please find a more detailed description of the experiments below ('Experiments'). We conclude these experiments with the following findings: 

* The effort involved in deploying Pryv on a Nomad/X cluster is currently rather large. This is in connection to the fact that we're very much stateful. 

* When targeting large installations, we'll have the choice between two extremes: 

  * A: A component in Pryv manages all machines and is aware of them, health-wise. 
  * B: We integrate with a variety of cluster management solutions deeply, allowing our clients to perform the same function with their favourite cluster management. 

* Option A has become more attractive now that we've experimented with the real world systems used for cluster management. If Pryv has its own management, the following advantages would eventually 'fall out': 

  * Health checking and reporting
  * Simplified installation and extension of the cluster
  * A logical place to orchestrate and initiate replication, backup and user migration. Maybe even migrate users automatically, based on resource constraints? 

  We can start small, however we're going to invest a lot of time before this works as described here. 

In a nutshell, we have to find a new idea for automating our deploys internally. Our client deploys will eventually be automated via our own system (probably). We hope to get around to constructing this system soon, i.e. After writing the code to scale up. 

# Experiments

We've created a two node cluster within the constraints of Pryv to experiment with nomad. We did not deploy Pryv on this cluster. We were stopped by state management on the cluster and how nomad and other tools deal with this. In particular, this ticket (https://github.com/hashicorp/nomad/issues/150) shows what nomad is doing and thinking about the issue. 

What we would need: 

* A system that can deploy registries and cores on prepared nodes. 
* Once a component has launched somewhere, this node becomes its home.
* Administrative control over these nodes, restarting and updating. Rollbacks. ...

Nomad in particular would cover the deployment and the administrative control, however it fails to really address the topic of state. It provides '`ephemeral_disk`', but it seems to fail to migrate it correctly (https://github.com/hashicorp/nomad/issues/3074). 

In addition to these problems (which we might be able to resolve) we would need to integrate Pryv with consul for service discovery. 

Conclusion is that today, this work needs too much time; we cannot keep treating it like a prerequisite to the HF development. Also: See the dichotomy exposed above: Maybe Pryv needs to address these issues on its own. 