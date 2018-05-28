# Prow for build next

This document evaluates [Prow][prow] as a candidate for Build.Next. The
[evaluation criteria][criteria] and the relevant issues [#3570][3570] and
[#3606][3606] contains more information.

### Abstract

Prow was built for testing k8s on k8s. Prow listens for events from github and a
comment of the form `/test all` starts a build, manages the complete life cycle
including a web UI and eventually reports the status back to Github. Overall
[architecture][architecture] is described fairly well in their documentation.

### 1. Multi tenancy

[Multi tenancy][multi-tenancy] is still a WIP, so this might not work for us at
all. I don't see this discussed elsewhere.

### 2. Explain basic architecture

[Official docs][architecture] cover the life cycle of a job. As said in the
abstract, the process starts with a github comment, spawning a job using the
jenkins backend and reporting the result back.

There seems to be a concept of multiple backends for running jobs, at least that
is what I understand from the [following docs][jenkins-operator].

> jenkins-operator is a controller that enables Prow to use Jenkins as a backend
> for running jobs.

I'm trying to find other backends as of now but so far I've only seen the
[Jenkins backend][jenkins-backend], which effectively talks to an external
Jenkins master to schedule builds with a HTTP client. If there are no other
backends, then Prow is not a good solution for our problems at all.

PS: This [commit][rm-jenkins] deleted most of the jenkins code outside the prow
operator from k8a-testing infra.

### 3. Is the tool language/framework/tool agnostic?

Seems to be so.

### 4. How are jobs configured?

Jobs are configured using a [config.yaml][config.yaml]. This looks very k8s
specific and might be way more complicated than what we want to build quick
start apps. It seems to expose a lot of internals (like configuring their micro
services directly) than give a high level DSL.

### 5. How does the k8s/os integration work?

Its from the same community, so I'd expect them to get this right.

### 6. Build a container from Dockerfile or s2i

As far as I understand, this happens on the backend executor which is jenkins
now. Need more reading here.

### 7. How are secrets handled?

I'd assume it to be vanilla k8s secrets, I havent seen anything else yet. I
might be wrong.

### 8. Build a language package and push to a repository.

As far as I understand, its not significantly different from what we do now.
Inject k8s secrets to Jenkins and do the same thing we do now.

### 9. Authentication and Authorization.

N/A. Might have to do it at the proxy level.

### 10. How are the logs, build info like status, start time etc stored?

Standard k8s.

---

Some interesting observations include:

### K8s jobs didn't work well for prow.

[Case study: Prow Jobs (and why Jobs are not enough for k8s testing) #43964][case-study]

As the [docs][docs-concept-job] say,

> A job creates one or more pods and ensures that a specified number of them
> successfully terminate.

A k8s job and a 'build job' looked so build for each other from far away, but
the fact that the former is expected to complete successfully while the other
could fail (user wrote some bad tests for example) leads to some impedance
mismatch. I wouldn't have expected this gap to be wide enough for Prow to invent
a new custom resource type called '[ProwJob][prowjob]' but they did just that. I
would have thought about wrapping the user code in an exception handler instead
of going down this way. [Limitations of k8s jobs][limitations] are discussed in
this issue.

### Prow runs a whole lot of tiny processes

[Sinker][sinker] for example cleans up old jobs and is less than 200 lines of
code. Seems to be very well in the spirit of k8s and makes the code a lot easier
to understand.

---

[case-study]: https://github.com/kubernetes/kubernetes/issues/43964
[docs-concept-job]: https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion
[jenkins-operator]: https://github.com/kubernetes/test-infra/blob/master/prow/cmd/jenkins-operator/README.md
[jess]: https://blog.jessfraz.com/post/hard-multi-tenancy-in-kubernetes/
[limitations]: https://github.com/kubernetes/kubernetes/issues/30243
[multi-tenancy]: https://github.com/kubernetes/test-infra/issues/7803
[prowjob]: https://github.com/kubernetes/test-infra/blob/master/prow/kube/prowjob.go
[rm-jenkins]: https://github.com/kubernetes/test-infra/commit/0b54684aa811d2caf8fbf65d63294f36a8fd8668
[sinker]: https://github.com/kubernetes/test-infra/blob/master/prow/cmd/sinker/main.go
[jenkins-backend]: https://github.com/kubernetes/test-infra/blob/c8829eef589a044126289cb5b4dc8e85db3ea22f/prow/jenkins/jenkins.go
[prow]: https://github.com/kubernetes/test-infra/tree/master/prow
[criteria]: https://github.com/fabric8io/buildnext/blob/prow/Evaluation.md
[3570]: https://github.com/openshiftio/openshift.io/issues/3570
[3606]: https://github.com/openshiftio/openshift.io/issues/3606
[architecture]: https://github.com/kubernetes/test-infra/blob/master/prow/architecture.md
[config.yaml]: https://github.com/kubernetes/test-infra/blob/master/prow/config.yaml
