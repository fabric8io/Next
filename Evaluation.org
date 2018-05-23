### A framework for evaluating services for Build.Next

**NOTE**: _This is a living document, which I'm fixing as I see holes in the
argument and all your comments and feedback. Just leave a note below if I missed
anything and I'll act on it._

We need a set of parameters and critereon to evaluate each of the canidates in
an unbiased way. I'm hoping to come up with a minimal list in _at least_ 3
categories - **Build**, **Orchestration**, **Deploy**. The goal is to answer the
same set of questions per tool so that the final comparison is easy and unless
we split the roles, we will never find a solution that really fits our needs in
all verticals.

We might need a 4th component, an API gateway to fit it all in place, so I'll
start with that.

#### Notes

1. Kubernetes support instead of just openshift is still something that needs to
   be decided.

#### API Gateway

This will be something of our own, so there is no need for an evaluation per se.

This service will act as a cushion b/w the services and rest of osio. The reason
to have it here as the first section is serve as a reminder that a service we
own is available b/w the service and osio to do some necessary translations if
necessary. This _might_ make working with certain tools (eg; a good build
service with an odd auth mechanism) much easier.

The major goals are,

1. Tie everything together and present a consistent API to the service
   irrespective of the underlying tools we use. This will be useful to make the
   component APIs consistent (buildNumber vs build_id), provide a way to keep
   the API same while changing provider implementations etc (current jenkins ->
   something new and sane).

2. RBAC. It might be easier to handle authentication and authorization in a
   consistent fashion at the API g/w than make each service understand the osio
   way of doing things.

3. Anything else that might make sense like load balancing, rate limiting etc

#### Build

In the simplest sense, build converts source to artifacts. We could think of it
as series of commands to execute on the source to get a docker image, jar or a
tarball. Users could optionally run their tests before or after artifact
creation.

In the simplest sense, the service will be asked to run a series of bash
commands in a specific container which shares the disk with certain other
containers in the same pod.

Potential Candidates: Circle, Concourse, Drone, Jenkins, JenkinsX, Maybe Prow,
Travis, Semaphore.

1. Multi tenancy.

   This is a hard problem to solve correctly and safely, so leaving this as the
   first requirement. With my limited understanding of k8s/os, this is one area
   in which they differ a lot as well so be clear on the differences as well.

   This blog [Hard Multi-Tenancy in Kubernetes](https://blog.jessfraz.com/post/hard-multi-tenancy-in-kubernetes/)
   should be good read before digging more into this.

2. Explain the basic architecture?

   At the core of it, is the tool cloud/container native and how well can it run
   on openshift? How does a simple bash command in a container work? Ideally I'm
   looking for paragraph explaining the basic architecture, flow, the complexity
   in the process involved. The equivalent of `kubectl exec centos date` with
   this tool.

   Ideally scaling this should not involve anything more than throwing more
   nodes into the os cluster.

3. Is the tool language/framework/tool agnostic?

4. How are jobs configured?

5. How does the k8s/os integration work?

   Is this a core feature or plugin? Link to code, explain the simplest
   workflow. A build tool which doesn't work well with os (like Jenkins) is not
   worth considering.

6. Build a container from Dockerfile or s2i

   How does the tool build docker containers? How does it deal with the root
   privilege problem?

7. How are secrets handled?

   We need this work with os secrets, so a POC using a secret from os would be
   nice. We will probably need this to be configurable because there is a good
   chance that our customers might use something external like Vault or AWS
   secret manager.

8. Build a language package and push to a repository.

   To clarify, this is not about building npm or maven support, but being able
   to do that with the same commands a js or java developer would use. This is
   more of a usability/sanity testing against the service to discover a lot of
   obvious flaws.

   1. Understand the requirements in terms of CPU, memory, IO, time etc and make
      sure your tool can build a sufficiently large app. We could start with our
      own quick starts obviously.

   2. Use secrets from previous step and push to a private registry

9. Authentication and Authorization. This could be done at the service level or
   the proxy mentioned in the previous section.

10. How are the logs, build info like status, start time etc stored?

    Is it configurable? We might need to send logs to a Gluster PV and other
    insights to PostgreSQL for example.

11. The usual engineering sanity bits.

    1. What sort of APIs does the tool expose?
    2. Are there enough tests?
    3. What languages/frameworks does it use?
    4. Is this something we can maintain if the upstream just disappears?
    5. Ideally we should not pick a tool before a few of our patches get
       upstream.

#### Orchestration

1. Define a language that the service understands. This DSL should be the
   boundary of what we could do with the service.

2. Translate this language into whatever build service we use. The less the
   differences, the better. For example, Changing an arbitrary Jenkinsfile to
   drone.yml might be borderline impossible.

3. Apply the right kind of rate limiting, resource capping etc. Implement fair
   scheduling b/w the tenants and among the jobs of a tenant. This might be done
   by the build service as well, but that is an an implementation detail

#### Deployment

Deployment is tricky and our requirements might be too specific to use a tool
out of the box. These are some of the things we have to do.

1. Deploy an application into a development namespace using the artifact
   produced by the build step.
2. Have the ability to promote an app from development namespace to staging or
   prod. This could be a different cluster altogether.
3. Support automating the process with Canary/AB deployments.

Some of these tools came up during the F2F, so I'm leaving it here as
references:

1. [Introducing Kayenta: An open automated canary analysis tool from Google and Netflix](https://cloudplatform.googleblog.com/2018/04/introducing-Kayenta-an-open-automated-canary-analysis-tool-from-Google-and-Netflix.html)
2. [Using Spinnaker for automated canary analysis](https://www.spinnaker.io/guides/user/canary)
3. [Spinaker, Continuous Delivery for Enterprise](https://www.spinnaker.io)
