# Automated Experiments with ArgoCD and Iter8

Say you are building a cloud app. Clearly, you will unit test the app during development to ensure that it behaves the way you expect. And no matter how you develop your app, your favorite unit testing tool will make it easy to author, execute and obtain results from your tests.

What about testing your app when you deploy it in a Kubernetes cluster (test/dev/staging/prod)? Does the app handle realistic load conditions with acceptable performance? Does the new version of the app improve business metrics relative to the earlier version? Is it resilient?

[Iter8](https://iter8.tools) is an open-source Kubernetes release optimizer that can help you get started with testing of Kubernetes apps in seconds. With Iter8, you can perform various kinds of [experiments](https://iter8.tools/0.13/getting-started/concepts/#iter8-experiment), such as (i) performance testing to ensure that your application can handle realistic load and satisfy SLOs, (ii) A/B(/n) tests that help you split users across app versions, collect business metrics, identify the winning version of your app that optimizes business metrics, and promote the winning version, and (iii) chaos injection tests, and more. Additionally, these experiments are composed using [discrete tasks](https://iter8.tools/0.13/user-guide/tasks/abnmetrics/), which can perform a variety of functions, so you can design an experiment to best fit your use case. All in all, Iter8 can help you get the most out of your Kubernetes services quickly and easily.

Iter8 is now introducing a new feature: [AutoX](https://iter8.tools/0.13/tutorials/autox/autox/). AutoX, short for automatic experimentation, allows you to perform the above experiments automatically, using a few simple labels on your Kubernetes resource objects. For instance, you can use the AutoX feature to automatically validate new versions of your app, as they are deployed in the cluster. Under the covers, AutoX is leveraging [Argo CD](https://argo-cd.readthedocs.io), a popular GitOps continuous delivery tool, to start these automatic experiments. Thus, AutoX greatly expands on functionality that Iter8 already provides.

In this article, we will explore automatically launching performance testing experiments for an HTTP service deployed in Kubernetes. At the end of this article, you should have everything you need to try out AutoX on your own!

Before trying out the hands-on tutorial documented in this article, you may wish to familiarize yourself with Iter8 [here](https://knative.dev/blog/articles/performance-test-with-slos/).

### AutoX

![AutoX](images/autox.png)

AutoX will detect changes to your Kubernetes resources and automatically start new experiments. To go into more detail, as a user of AutoX, you will need to specify your resources and the experiments you would like to run. Then, AutoX will watch those resources for an updates. If the resources have been changed, then AutoX will make an additional check on its labels. This label check, which we will go into greater detail later, ensures that AutoX should restart experiments for these resources and that the resources have been changed significantly enough to warrant the restart. Therefore, if the resources have changed and the label check has passed, then AutoX will restart the experiments.

To specify your resources and the experiments you would like to run, you will need to provide a *trigger* and a *set of experiment charts*, respectively. The trigger specifies the Kubernetes resources that AutoX should watch and is a combination of name, namespace, and GVR (group, version, resource). Meanwhile, the experiment charts specify the experiments that should be launched and come from [Iter8 Hub](https://github.com/iter8-tools/hub). Note that Iter8 Hub primarily contains Iter8 experiments but it also contains other experiments such as Litmus-based chaos injection experiments.

The label check is used to ensure that the experiments should be restarted. For example,  you may not want to necessarily restart an experiment just because a resource's `status` is changing and you may wish to control when AutoX will restart experiments without stopping AutoX entirely. The first check is to see if the following labels have been changed: `app.kubernetes.io/name"`, `app.kubernetes.io/version`, and `iter8.tools/track"` (referred to as the trigger labels). The trigger labels are used to determine if the resources have been changed significantly enough to warrant the restart. If none of these labels have been changed, then AutoX will not restart the experiments. The second check is to see if the resources have the `iter8.tools/autox-group` label (referred to as the AutoX label). The AutoX label is how the user can toggle AutoX on or off. If the resource does not have the label, then AutoX will not restart any experiments, regardless of all other factors. The combination of these these two checks forms the label check.

If AutoX determines that based on the changes to the resource and the label check, the experiments should be restarted, then AutoX will first try to delete any preexisting experiments and then restart them. However, if the resource does not have the AutoX label, then AutoX will try to delete any preexisting experiments.

![Flow chart](images/flowchart.png)

In the following tutorial, we will deploy an HTTP service and set up AutoX such that whenever the HTTP service is modified, AutoX will start a new HTTP SLO validation test that will check if the HTTP service meets minimum latency and error-related requirements.

##### Setup Kubernetes cluster with ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

See [here](https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-cd) for more information.

##### Download Iter8 CLI

```bash
brew tap iter8-tools/iter8
brew install iter8@0.13
```

See [here](https://iter8.tools/0.11/getting-started/install/) for alternate methods of installation.

##### Setup Kubernetes cluster with Iter8 AutoX

```bash
helm repo add iter8 https://iter8-tools.github.io/hub/
helm install autox-httpbin iter8/autox \
--set 'releaseGroupSpecs.httpbin.trigger.name=httpbin' \
--set 'releaseGroupSpecs.httpbin.trigger.namespace=default' \
--set 'releaseGroupSpecs.httpbin.trigger.group=apps' \
--set 'releaseGroupSpecs.httpbin.trigger.version=v1' \
--set 'releaseGroupSpecs.httpbin.trigger.resource=deployments' \
--set 'releaseGroupSpecs.httpbin.releaseSpecs.iter8.name=iter8' \
--set 'releaseGroupSpecs.httpbin.releaseSpecs.iter8.values.tasks={ready,http,assess}' \
--set 'releaseGroupSpecs.httpbin.releaseSpecs.iter8.values.ready.deploy=httpbin' \
--set 'releaseGroupSpecs.httpbin.releaseSpecs.iter8.values.ready.service=httpbin' \
--set 'releaseGroupSpecs.httpbin.releaseSpecs.iter8.values.ready.timeout=60s' \
--set 'releaseGroupSpecs.httpbin.releaseSpecs.iter8.values.http.url=http://httpbin.default/get' \
--set 'releaseGroupSpecs.httpbin.releaseSpecs.iter8.values.assess.SLOs.upper.http/latency-mean=50' \
--set 'releaseGroupSpecs.httpbin.releaseSpecs.iter8.values.assess.SLOs.upper.http/error-count=0' \
--set 'releaseGroupSpecs.httpbin.releaseSpecs.iter8.version=0.13.0' \
--set 'releaseGroupSpecs.httpbin.releaseSpecs.iter8.values.runner=job'
```

As mentioned previously, the input to AutoX is a trigger and a set of experiment charts. Here, the trigger is a Kubernetes resource object with the name `httpbin`, namespace `default`, and GVR `apps`, `deployments`, and `v1`, meaning that AutoX will watch any resources that meets the description. If a changed is made to the trigger labels of a matching resource, then AutoX will begin the label check and ensure that the resource's trigger labels have been changed and the resource has an AutoX label.

In this case, there is only one experiment to start, an HTTP SLO validation test on the `httpbin` service. This Iter8 experiment is composed of three tasks, `ready`, `http`, and `assess`. The `ready` task will ensure that the `httpbin` deployment and service are running. The `http` task will make requests to the specified URL and will collect latency and error-related metrics. Lastly, the `assess` task will ensure that the mean latency is less than 50 milliseconds and the error count is 0. In addition, the runner is set to job as this will be a [single-loop experiment](https://iter8.tools/0.11/getting-started/concepts/#iter8-experiment).

##### Create application

Now, we will create the `httpbin` deployment and service.

```bash
kubectl create deployment httpbin --image=kennethreitz/httpbin --port=80
kubectl expose deployment httpbin --port=80
```

##### Apply labels

In the previous step, `httpbin` deployment which is the trigger. However, regardless of what changes we make to this deployment, AutoX will not do anything unless we assign the trigger the AutoX label.

```bash
kubectl label deployment httpbin iter8.tools/autox=true
```

##### Observe automatic experiment

After you have assigned the AutoX label, an experiment should start.

You can now use `iter8` commands in order to check the status and the results of the experiment. Note that you need to specify an experiment group via the `-g` option. The experiment group for experiments started by AutoX is in the form `autox-<release group spec name>-<release spec name>`. In this case, it would be `autox-httpbin-iter8`. 

The following command allows you to check the status of the experiment. If the experiment does not immediately start, try waiting a minute.

```bash
iter8 k assert -c nofailure -c slos -g autox-httpbin-iter8
```

We can see in the sample output that the experiment has completed and all SLOs and conditions were satisfied.
```
INFO[2023-01-11 14:43:45] inited Helm config                           
INFO[2023-01-11 14:43:45] experiment has no failure                    
INFO[2023-01-11 14:43:45] SLOs are satisfied                           
INFO[2023-01-11 14:43:45] all conditions were satisfied  
```

And the following command allows you to check the results of the experiment.

```bash
iter8 k report -g autox-httpbin-iter8
```

In the sample output, we can see an experiment summary, a list of SLOs and whether they were satisfied or not, as well as any additional metrics that were collected as part of the experiment.
```
Experiment summary:
*******************

  Experiment completed: true
  No task failures: true
  Total number of tasks: 4
  Number of completed tasks: 4

Whether or not service level objectives (SLOs) are satisfied:
*************************************************************

  SLO Conditions                 | Satisfied
  --------------                 | ---------
  http/error-count <= 0          | true
  http/latency-mean (msec) <= 50 | true
  

Latest observed values for metrics:
***********************************

  Metric                     | value
  -------                    | -----
  http/error-count           | 0.00
  http/error-rate            | 0.00
  http/latency-max (msec)    | 25.11
  http/latency-mean (msec)   | 5.59
  http/latency-min (msec)    | 1.29
  http/latency-p50 (msec)    | 4.39
  http/latency-p75 (msec)    | 6.71
  http/latency-p90 (msec)    | 10.40
  http/latency-p95 (msec)    | 13.00
  http/latency-p99 (msec)    | 25.00
  http/latency-p99.9 (msec)  | 25.10
  http/latency-stddev (msec) | 4.37
  http/request-count         | 100.00
```

##### Push new version of the application

Now that AutoX is watching the `httpbin` deployment, any change that we make to its trigger labels will cause AutoX to trigger a new experiment.

We will start a new experiment by adding a new trigger label to the `httpbin` deployment.

```bash
kubectl label deployment httpbin app.kubernetes.io/version=1.0.0
```

##### Observe new automatic experiment

Check to see if a new experiment should have started. Refer to [Observe automatic experiment](#observe-automatic-experiment) for the necessary commands.

##### Discussion

You can continue to modify the trigger labels of the `httpbin` deployment to start new experiments. For example, you can continue to bump the `app.kubernetes.io/version` label to start a new HTTP SLO validation test. In practice, updates to your trigger resource should be accompanied by changes to its trigger labels.

It is possible to use AutoX to conduct more complex experiments. Iter8 experiments are composed from [discrete tasks](https://iter8.tools/0.13/user-guide/tasks/abnmetrics/) so you can design an experiment that best fits your use case. For example, instead of using the `httpbin` task, you can use `grpc` task in order to run an GRPC SLO validation test, or you can add a `slack` task so that your experiment result will be posted on Slack upon completion. You can also run experiments that are not from Iter8. For example, there is an experiment chart for a Litmus Chaos chaos experiment on Iter8 hub. Lastly, you can supply multiple experiments charts so AutoX will start a suite of experiments that will run whenever you update your trigger resources.

### Takeaways

AutoX, auto experimentation, is a powerful new feature of Iter8 that can let you automatically run performance tests on your Kubernetes services. Setting AutoX is straightforward and just requires specifying a trigger and the experiments you want to run. As you continue to iterate upon your services, you can get the latest performance statistics the moment they are updated, making testing a seamless operation.

After trying out the tutorial, consider trying it out on your own services. Please see the [Iter8 documentation](https://iter8.tools) for more information about AutoX, experiments, tasks, and much more! If you need any help, you can find us on [Slack](https://join.slack.com/t/iter8-tools/shared_invite/zt-awl2se8i-L0pZCpuHntpPejxzLicbmw) where we are happy to answer any questions.