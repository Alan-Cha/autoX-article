# Automated Experiments with ArgoCD and Iter8

[Iter8](https://iter8.tools) is an open-source Kubernetes release optimizer that can help you get started with performance testing in seconds. With Iter8, you can perform various kinds of experiments, such as SLO validation, A/B(/n) tests, and chaos testing, and discover which services are performing the best. These experiments are composed of various discrete tasks, which can perform functions such as waiting for services to start, collecting metrics from services, and validating those metrics against SLOs, so you can customize the experiments to fit your use case. All in all, Iter8 can help you get the most out of your Kubernetes apps and ML-models quickly and easily.

Iter8 is now introducing a new feature: AutoX. AutoX, short for automatic experimentation, allows Iter8 to detect new versions of your services and automatically trigger new experiments, allowing you to tests your services as soon as you push them. Under the covers, Iter8 is leveraging [Argo CD](https://argo-cd.readthedocs.io), a popular GitOps continuous delivery tool. With Argo CD, Iter8 is able to start not only new Iter8 experiments but also other kinds of experiments, such as Litmus Chaos chaos tests. Thus, AutoX greatly expands on the testing that Iter8 can already perform.

In this article, we will describe how you can set up Iter8 and AutoX for your own Kubernetes cluster. 

This tutorial will assume that you have some prior knowledge about Iter8. If you would like to see a basic tutorial about Iter8, see [here](https://knative.dev/blog/articles/performance-test-with-slos/).

### AutoX

To be more exact, AutoX detects changes to a particular Kubernetes resource and will install a set of Helm charts.

The input to AutoX is a trigger and a set of Helm charts. A trigger describes what Iter8 should watch for and is a combination of name, namespace, and GVR (group, version, resource). When a matching resource is created, updated and delete, **and** has an `iter8.tools/autox-group` label (referred to as the AutoX label for the remainder of the article) then AutoX will begin to install or delete the Helm charts.

As mentioned previously, AutoX is utilizing Argo CD. Iter8 uses Argo CD to install these Helm charts. The Helm charts could contain an Iter8 experiment, which would allow automatic experimentation to occur. However, the Helm charts are not limited to just Iter8 experiments and could contain other things such as a Litmus Chaos chaos experiment.

Different behaviors will occur what kind of change has occured on the GVR. If a change is made on a GVR that does not have the AutoX label, then nothing will happen. If a GVR with an AutoX label has been created or if a GVR has been modified to add an AutoX label, then the Helm charts will be installed. If a GVR with an AutoX label has been changed, then the Helm charts will be deleted and reinstalled. However, if part of the change is that the AutoX label was removed, then the charts will not be reinstalled. Lastly, if the GVR with an AutoX label is deleted, then the Helm charts will also be deleted.

<!-- TODO: Flow chart -->

In the following tutorial, we will deploy an HTTP service and set up AutoX such that whenever the HTTP service is modified, AutoX will start a new HTTP SLO validation test that will check if the HTTP service meets minimum latency and error-related requirements.

##### Install ArgoCD services and application resources

```bash
kubectl apply -n default -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

See [here](https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-cd) for more information.

##### Install Iter8

```bash
brew tap iter8-tools/iter8
brew install iter8@0.11
```

See [here](https://iter8.tools/0.11/getting-started/install/) for alternate methods of installation.

##### Install Iter8 controller

```bash
helm repo add iter8 https://iter8-tools.github.io/hub/
helm install httpbin iter8/autox \
--set 'chartGroups.httpbin.trigger.name=httpbin' \
--set 'chartGroups.httpbin.trigger.namespace=httpbin' \
--set 'chartGroups.httpbin.trigger.group=apps' \
--set 'chartGroups.httpbin.trigger.version=v1' \
--set 'chartGroups.httpbin.trigger.resource=deployment' \
--set 'chartGroups.httpbin.releaseSpec[0].repoURL=https://iter8-tools.github.io/hub/' \
--set 'chartGroups.httpbin.releaseSpec[0].name=iter8' \
--set 'chartGroups.httpbin.releaseSpec[0].values=...' \
--set 'chartGroups.httpbin.releaseSpec[0].version=0.12.2'
```

As mentioned previously, the input to AutoX is a trigger and a set of Helm charts. Here, we are defining a trigger with GVR `apps`, `deployment`, and `v1`, respectively, meaning that when an `app/v1` deployment with an AutoX label is created or an `app/v1` deployment is modified and has an AutoX label after the modification, then AutoX will install the provided Helm chart. 

In this case, the Helm chart is pointing to the Iter8 experiment chart, specifically one designed to perform an HTTP SLO validation test on the `httpbin` service.

##### Create service

```bash
kubectl create deployment httpbin --image=kennethreitz/httpbin --port=80
kubectl expose deployment httpbin --port=80
```

##### Assign service AutoX labels

By assigning the AutoX label to your service, you are informing Iter8 that you would like it to have AutoX enabled. 

TODO: Reference to Iter8 groups?

Set the value to `httpbin`

```bash
kubectl label deployment httpbin iter8.tools/autox-group=httpbin
```

##### Watch for Iter8 experiment

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

##### Create a new version of the service

We will add a new label to the deployment in order to trigger a new experiment.

```bash
kubectl label deployment httpbin hello=world
```

As long as this deployment continues to have the AutoX label, the AutoX will watch it for changes. If the label is removed or if the entire deployment is deleted, the AutoX will also delete the corresponding Helm charts.

##### Watch for new Iter8 experiment

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

You can continue to modify the `httpbin` deployment to trigger new experiments. In this example, we only changed a label but a more realistic change would be updating it to a new version. With every updated version, AutoX will run the HTTP SLO validation test and inform you if the latest version is meeting basic requirements.

It is also possible to use AutoX with more complex experiments and other types of experiments. Iter8 experiments are composed of discrete tasks so you can create an experiment that best fits your use case. You can also run other tests like A/B(/n) tests to compare how your development version compares to the production version or like chaos tests, to see how resilient your service is.

Note that if you delete the AutoX label or if you delete the `httpbin` development, then AutoX will also delete the respective Helm charts.

### Takeaways

AutoX, auto experimentation, is a powerful new feature of Iter8 that can let you automatically run performance tests on your Kubernetes apps and ML-models.

After trying out the tutorial, consider trying it out on your own services. If you need any help, you can find us on [Slack]() where we are happy to answer any questions.