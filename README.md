## Scan Your Docker Containers For Vulnerabilities

This article will look at how to scan a Docker image for security vulnerabilities and ideas for integrating a security scan into your CI/CD platform. We will setup a demo scanner system using Clair and Klar running in pods in a Kubernetes cluster.

Before we start - let's talk vocabulary. This article talks about Docker, Dockerfiles, containers and images - what's the difference? Docker is a "container runtime", it wraps the fundamental concept of containers with tools that help humans build, run and archive containers. There are many container runtimes to help you run a container, we are going to focus on Docker in this article.

An image is a static file that is a snapshot of a container. Images are built from a blueprint (in Docker's case a "Dockerfile") and can be stored in a registry. The image produces a container when started with run. 

Docker containers are a great way to encapsulate your application alongside everything you need to run your application. Combining docker containers and an agile CI/CD system can allow developers to push, test and deploy changes to containerized applications quickly. But with this fast iterative development style how can you trust the software running in your production containers are secure? 

Security is a multilayered problem that should be considered from many angles. This article is going to consider one angle, how to scan an image for vulnerabilities. We will look at how to scan an image in a minikube Kubernetes cluster, once you have an idea of how to scan and image it should make sense how you can integrate this process into your CI/CD platform. This will allow developers or admins to configure a CI job that will set a maximum number of security vulnerabilities of a certain level and the pipeline will fail if the scan discovers more vulnerabilities than the stated maximum. Be certain what know vulnerability exist in any container you deploy to production.

Before jumping in, let's break down the problem and concepts a bit more. Most importantly, why should you care? Why scan your docker images for vulnerabilities? 

### What even is a vulnerability? 

A vulnerability or exposure in a piece of software is a weak section of code or process that allows access from a potentially malicious third party. Generally speaking, it's a bug that a hacker could take advantage of. Remember the "Heartbleed" - that was a major bug in software that many companies relied on. Not all bugs are that big, but they can still be a disaster for your organization, so people are organizations around the world agreed we need to work together to document and distribute a list of the bugs we know about to try and prevent people from putting buggy software into production.

[The MITRE corporation](https://cve.mitre.org/about/faqs.html#MITRE_role_in_cve) currently maintains the "Common Vulnerabilities and Exposures" or "CVE" which is a list of common identifiers for publicly known cybersecurity vulnerabilities. Plainly - the MITRE corporation maintains a list of these bugs. Each vulnerability in the CVE is standardized with details so developers can know how serious it is and how it works. You can learn more about how [CVE](https://cve.mitre.org/about/index.html) work, but for this article we will focus on an averaged "severity level" which ranges from low, medium, high to critical, this score is calculated by the [CVSS](https://en.wikipedia.org/wiki/Common_Vulnerability_Scoring_System) "Common Vulnerability Scoring System". 

So we have a list of knowns bugs (CVE) and we have a Docker image - how do we make sure the software in the Docker image doesn't include a known vulnerability? Scan it! 

### Clair: Vulnerability Static Analysis for Containers

We are going to use an open source vulnerability scanner for containers called Clair which was created by CoreOS. Let's get an idea of how Clair works by running a demo in Kubernetes cluster using Minikube. 

First you need to get minikube up and running - [learn more about minikube and follow the quickstart to spin up a local Kubernetes cluster](https://github.com/kubernetes/minikube). Once minikube has spun up your Kubernetes cluster you should be able to access it using `kubectl`.

Here are the Kubernetes resources we are going to deploy to scan our images:

"clairsecret" [secret](https://kubernetes.io/docs/concepts/configuration/secret/)
"clairsvc" [service](https://kubernetes.io/docs/concepts/services-networking/service/)
"clair" [ReplicationController](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)
"clair-postgres" ReplicationController
"postgres" service
"klar" [job](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/)

Clair is a process that will check each layer in your Docker image for vulnerabilities. Our Postgres database resources will be populated with the CVE list for Clair to check against. [Klar](https://github.com/optiopay/klar) is a process we will run as a job to pass an image stored in a registry to our Clair process. Think of Clair as the backend process that coordinates with the database and Klar as the front end process that coordinates getting the image from the registry and passing it to Clair.

`> kubectl create secret generic clairsecret --from-file=./config.yaml`
`> kubectl get secret`
`> kubectl create -f clair-kubernetes.yaml`

NOTE: Clair will take ~20 minutes to populate the database with the list of vulnerabilities. You can run Klar without errors, but the vulnerability count will not be correct until the database is populated. Grab some coffee or catch up on Reddit before continuing to the next step.

At this point you should have all the resources to scan an image. Let's take a look at the `klar.yaml` file. This is a job that will run a [container](https://github.com/leahnp/klar/blob/master/Dockerfile) that contains the Klar application. We will pass klar the container we want to scan in the form of an argument (we will be scanning: quay.io/samsung_cnct/fluentd-central:latest) and provide the necessary environment variables for Klar to connect to Clair and set our maximum vulnerability number (CLAIR_ADDR, CLAIR_OUTPUT, CLAIR_THRESHOLD. Before we scan our first image, let's make sure our `CLAIR_ADDR` variable is correct by running (be sure to put the name of the clair pod running in your cluster in the following command):

`> minikube service clairsvc --url`

Take the first address of the output and put it in the `klar.yaml` file as the value for the `CLAIR_ADDR` variable. Save your file and let's run the job:

`> kubectl create -f klar.yaml`

Wait until the klar pod's status is "Completed" then run:

`> kubectl logs klar`

You should see the summary of vulnerabilities found at the top and the details of each vulnerability listed after. To see the pod exit code run:

`> kubectl describe po klar`

and look for:

```
State:          Terminated
   Reason:       Completed
   Exit Code:    0
```

So, the process exited successfully with a `0`, let's modify the maximum vulnerabilities to make our klar process fail. In `klar.yaml` change the `CLAIR_THRESHOLD` variable to 1. Then cleanup the old klar job and start a new one:

`> kubectl delete job klar`
`> kubectl create -f klar.yaml`

If you take a look at the logs, you should see similar output, but if you describe the pod you will notice the exit status is 1, indicating the maximum vulnerabilities were exceeded. If you want to try scanning a different image, change the arg we pass to Klar in the `klar.yaml` file (currently it is: `quay.io/leahnp/klar:latest`).

So what level of risk is appropriate? This question really depends on your company and the details of what is running in the container and where. You should work with your team to outline a strategy and best practices approach to vulnerability levels and patterns to address fixes. 

### Now we have Secure Containers? Don't be so sure

By integrating a job like this into your CI pipeline you can guarantee a certain risk level for all your containers that go into production. But before you say you've solved your container security issues - think about this scenario. You have a long lived container that doesn't get built via CI often, maybe because it doesn't require updates or new features. Since the time it passed your CI vulnerability scan, a vulnerability was added to the CVE for one of the processes in that container. How would you ever know?

This approach scans a static image and verifies it's contents before it deploys to production - scanning running containers on regular intervals is another approach to understanding how secure your cluster is.