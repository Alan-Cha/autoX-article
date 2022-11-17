# Automated Experiments with ArgoCD and Iter8

Say you are building a cloud app. Clearly, you will unit test the app during development to ensure that it behaves the way you expect. And no matter how you develop your app, your favorite unit testing tool will make it easy to author, execute and obtain results from your tests.

What about testing your app when you deploy it in a Kubernetes cluster (test/dev/staging/prod)? Does the app handle realistic load conditions with acceptable performance? Does the new version of the app improve business metrics relative to the earlier version? Is it resilient?

[Iter8](https://iter8.tools) is an open-source Kubernetes release optimizer that can help you get started with performance testing in seconds. With Iter8, you can perform various kinds of experiments, such as SLO validation, canary tests, A/B(/n) tests, and chaos injection tests, and use these experiments to discover which of your services is performing the best. These experiments are composed of various discrete tasks, which can perform different functions such as waiting for services to start, collecting metrics from services, and validating those metrics against SLOs, so you can customize experiments to fit your exact use case. All in all, Iter8 can help you get the most out of your Kubernetes apps and ML-models quickly and easily.

Iter8 is now introducing a new feature: AutoX. AutoX, short for automatic experimentation, allows Iter8 to detect new versions of your services and automatically trigger new experiments, allowing you to tests your services as soon as you push out a new version. Under the covers, AutoX is leveraging [Argo CD](https://argo-cd.readthedocs.io), a popular GitOps continuous delivery tool, to start these automatic experiments. Thus, AutoX greatly expands on functionality that Iter8 already provides.

In this article, we will explore automatically launching performance testing experiments for an HTTP service deployed in Kubernetes. At the end of this article, you should have everything you need to try out AutoX on your own!

This tutorial will assume that you have some prior knowledge about Iter8. If you would like to see a basic tutorial about Iter8, see [here](https://knative.dev/blog/articles/performance-test-with-slos/).

### AutoX

![AutoX](images/autox.png)

Earlier, we described AutoX as detecting new versions of your services and automatically triggering experiments. To be more exact, AutoX detects changes to a particular Kubernetes resource and will execute Iter8 experiments based on its labels. For example, AutoX can detect changes to a particular deployment and when a new version is pushed out, AutoX can spin up a new SLO test to see if the new version satisifies basic requirements.

The input to AutoX is a trigger and a set of Helm charts. A trigger describes the Kubernetes resource object that Iter8 should watch and is a combination of name, namespace, and GVR (group, version, resource). When a matching resource is created, updated and delete, **and** has an `iter8.tools/autox-group` label (referred to as the AutoX label for the remainder of the article) then AutoX will begin to install or delete the Helm charts.

These Helm charts come from [Iter8 Hub](https://github.com/iter8-tools/hub). Iter8 Hub primarily contains Iter8 experiments but it also contains other experiments such as Litmus-based chaos injection experiments.

As mentioned previously, AutoX is utilizing Argo CD. Iter8 uses Argo CD to install these Helm charts. The Helm charts could contain an Iter8 experiment, which would allow automatic experimentation to occur. However, the Helm charts are not limited to just Iter8 experiments and could contain other things such as a Litmus Chaos chaos experiment.

Different behaviors will occur based on what kind of change has occured on the watched Kubernetes resource object. If a change is made on a Kubernetes resource object that does not have the AutoX label, then nothing will happen. If a Kubernetes resource object with an AutoX label has been created or if a Kubernetes resource object has been modified to add an AutoX label, then the Helm charts will be installed. If a Kubernetes resource object with an AutoX label has been changed, then the Helm charts will be deleted and reinstalled. However, if part of the change is that the AutoX label was removed, then the charts will not be reinstalled. Lastly, if the Kubernetes resource object with an AutoX label is deleted, then the Helm charts will also be deleted.

![Flow chart](images/flowchart.png)

In the following tutorial, we will deploy an HTTP service and set up AutoX such that whenever the HTTP service is modified, AutoX will start a new HTTP SLO validation test that will check if the HTTP service meets minimum latency and error-related requirements.

##### Setup Kubernetes cluster with ArgoCD

```bash
kubectl apply -n default -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

See [here](https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-cd) for more information.

##### Download Iter8 CLI

```bash
brew tap iter8-tools/iter8
brew install iter8@0.11
```

See [here](https://iter8.tools/0.11/getting-started/install/) for alternate methods of installation.

##### Setup Kubernetes cluster with Iter8 AutoX

```bash
helm repo add iter8 https://iter8-tools.github.io/hub/
helm install httpbin iter8/autox \
--set 'chartGroups.httpbin.trigger.name=httpbin' \
--set 'chartGroups.httpbin.trigger.namespace=default' \
--set 'chartGroups.httpbin.trigger.group=apps' \
--set 'chartGroups.httpbin.trigger.version=v1' \
--set 'chartGroups.httpbin.trigger.resource=deployment' \
--set 'chartGroups.httpbin.releaseSpec[0].name=iter8' \
--set 'chartGroups.httpbin.releaseSpec[0].values.tasks={ready,http,assess}' \
--set 'chartGroups.httpbin.releaseSpec[0].values.ready.deploy=httpbin' \
--set 'chartGroups.httpbin.releaseSpec[0].values.ready.service=httpbin' \
--set 'chartGroups.httpbin.releaseSpec[0].values.ready.timeout=60s' \
--set 'chartGroups.httpbin.releaseSpec[0].values.http.url=http://httpbin.default/get' \
--set 'chartGroups.httpbin.releaseSpec[0].values.assess.SLOs.upper.http/latency-mean=50' \
--set 'chartGroups.httpbin.releaseSpec[0].values.assess.SLOs.upper.http/error-count=0' \
--set 'chartGroups.httpbin.releaseSpec[0].values.runner=job' \
--set 'chartGroups.httpbin.releaseSpec[0].version=0.12.2'
```

As mentioned previously, the input to AutoX is a trigger and a set of Helm charts. Here, the trigger is a Kubernetes resource object with the name `httpbin`, namespace `default`, and GVR `apps`, `deployment`, and `v1`, respectively, meaning that AutoX will watch for any Kubernetes resource object that fits the description **and** also has an AutoX label. When such a Kubernetes resource object (with an AutoX label) is created or when such a Kubernetes resource object is modified (and has the AutoX label), then AutoX will install the specified Helm charts.

In this case, there is only one Helm chart to install. The specified Helm chart is pointing to an Iter8 experiment, specifically an HTTP SLO validation test on the `httpbin` service. 

This experiment is composed of three tasks, `ready`, `http`, and `assess`. The `ready` task will ensure that the `httpbin` deployment and service are running. The `http` task will make requests to the specified URL and will collect latency and error-related metrics. Lastly, the `assess` task will ensure that the mean latency is less than 50 milliseconds and the error count is 0. In addition, the runner is set to job as this will be a [single-loop experiment](https://iter8.tools/0.11/getting-started/concepts/#iter8-experiment).

##### Create application

Now, we will create the `httpbin` deployment and service.

```bash
kubectl create deployment httpbin --image=kennethreitz/httpbin --port=80
kubectl expose deployment httpbin --port=80
```

##### Apply labels

In the previous step, we created an `apps/v1` deployment with the name `httpbin` in the `default` namespace, which matches the trigger that we configured for AutoX. However, to enable AutoX for the Kubernetes resource object, we need to assign it the AutoX label.

```bash
kubectl label deployment httpbin iter8.tools/autox-group=httpbin
```

The value of the AutoX label is just an identifier. It does not necessarily need to match the name of the trigger Kubernetes resource object.

##### Observe automatic experiment

After you have assigned the AutoX label, an experiment should start.

The following command allows you to check the status of the experiment.

```bash
iter8 k assert -c nofailure -c slos
```

<!-- TODO: describe output -->

And the following command allows you to check the results of the experiment.

```bash
iter8 k report
```

<!-- TODO: describe output -->

##### Push new version of the application

Now that AutoX is watching the `httpbin` deployment, any change that we make to deployment will cause AutoX to trigger a new experiment.

We will trigger a new experiment by simply adding a new label to the `httpbin` deployment.

```bash
kubectl label deployment httpbin hello=world
```

As long as the `httpbin` deployment continues to have the AutoX label, then AutoX will watch it for changes. If the label is removed or if the entire deployment is deleted, then AutoX will also delete the corresponding Helm charts.

##### Observe new automatic experiment

Check to see if a new experiment should have started.

The following command allows you to check the status of the experiment.

```bash
iter8 k assert -c nofailure -c slos
```

And the following command allows you to check the results of the experiment.

```bash
iter8 k report
```

##### Discussion

You can continue to modify the `httpbin` deployment to trigger new experiments. In this example, we only changed a label but a more realistic change would be updating the image. With every update, AutoX will run the HTTP SLO validation test and inform you if the latest version is meeting basic requirements.

It is also possible to use AutoX with more complex experiments. Iter8 experiments are composed of discrete tasks so you can compose an experiment that best fits your use case. You can also run other tests like A/B(/n) tests, to compare how your development version compares to the production version, or chaos injection tests, to see how resilient your service is.

Note that if you delete the AutoX label or if you delete the `httpbin` development, then AutoX will also delete the respective Helm charts.

### Other variations

A/B(/n) experiment?

With chaos injection test? Multiple tests?

### Next steps

Now that you have tried out AutoX, try it out on your own apps!

### Takeaways

AutoX, auto experimentation, is a powerful new feature of Iter8 that can let you automatically run performance tests on your Kubernetes apps and ML-models.

After trying out the tutorial, consider trying it out on your own services. If you need any help, you can find us on [Slack](https://join.slack.com/t/iter8-tools/shared_invite/zt-awl2se8i-L0pZCpuHntpPejxzLicbmw) where we are happy to answer any questions.