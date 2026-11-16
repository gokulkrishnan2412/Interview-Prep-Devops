1. Tell me about the e-commerce platform you worked on.
“It’s a Bosch e-commerce web application running on Microsoft Azure, hosted on Azure Kubernetes Service. For about two years I supported its deployment and operation.
That meant helping deploy new versions, validating workload health after releases, monitoring the application, and troubleshooting issues with the development and
operations teams. [CONFIRM: add the architecture, e.g. frontend, backend services, database, and how many services or environments.]”

3. What was your role in deployments?
“I supported the deployment process and validated it. After a release, I checked that the deployments, pods and services were healthy and that the application was available.
 When something failed, I investigated with kubectl and Azure Monitor and worked with the developers on the fix.
 [CONFIRM: say whether you triggered deployments yourself, wrote or edited YAML or Helm charts, or only validated.]”

5. How was the application deployed to AKS?
“The application runs as containerized workloads on AKS. Images are built and stored in a container registry,
 [CONFIRM: Azure Container Registry], and the Kubernetes deployments pull them from there. [CONFIRM: the tool that triggered the release, e.g. Jenkins, Azure DevOps or manual kubectl/Helm.]
After deployment, I verified rollout status and pod health.”

7. Which environments did you work with?
“[CONFIRM: e.g. dev, test/QA, staging and production.]
Changes are validated in the lower environments before going to production, following the project’s approval process.”

B. Deployment and release

5. Walk me through a typical deployment.
“First, the new version of the application is built into a container image and pushed to the registry. Second,
 the deployment is updated with the new image tag, [CONFIRM: through a pipeline, Helm, or manifests].
Third, I monitor the rollout with kubectl rollout status and check that the new pods are Running and Ready. Fourth, I check the logs and Azure Monitor for errors, and I confirm with the
team that the application works. If there’s a problem, we investigate or roll back.”

7. What deployment strategy does Kubernetes use by default?
“A rolling update. Kubernetes replaces old pods gradually with new ones, so the application stays available.
 The maxUnavailable and maxSurge settings control how many pods can be down or extra during the update. Readiness probes make sure traffic only goes to pods that are ready.”

9. How do you roll back a bad deployment?
“I check the history with kubectl rollout history deployment <name> and roll back with kubectl rollout undo deployment <name>.
 Then I verify that the pods are healthy and the application is working. I only do this with the approval of the responsible team, and afterwards we find the root cause.
 [CONFIRM: if you have actually performed a rollback, add that one real example.]”

11. What is the difference between blue-green and canary deployment?
“In blue-green, two identical environments exist. Traffic switches from the old one (blue) to the new one (green) all at once, which makes rollback fast.
In canary, a small percentage of traffic goes to the new version first, and it’s increased gradually if the metrics look good. Canary reduces risk but needs more traffic
 control and monitoring. [CONFIRM: say honestly which one your project uses; if neither, say ‘we use rolling updates’.]”

13. How do you verify that a deployment was successful?
“I don’t rely only on the pipeline showing green. I check pod status, the deployment’s available replicas, the service endpoints,
 and the container logs for errors. I also look at Azure Monitor metrics and check that the application responds correctly, ideally with a smoke test or a quick functional check by QA.”

C. Kubernetes troubleshooting

10. A pod is in CrashLoopBackOff. What do you do?
“It means the container keeps starting and crashing. I run kubectl describe pod to see events and the exit code, then kubectl logs
<pod> and kubectl logs <pod> --previous to see why it crashed. Common causes are an application error, a wrong configuration or environment variable,
 a missing secret, a failed dependency such as a database connection, or insufficient memory causing an OOMKill. I fix or escalate based on what I find.”

12. A pod is in ImagePullBackOff. What do you check?
“I check the image name and tag, whether the image exists in the registry, and whether AKS has permission to pull from it.
For Azure Container Registry, that means checking that the AKS cluster is attached to ACR or has the AcrPull role. I also check for typos in the tag and for registry networking issues.”

14. A pod is stuck in Pending.
“Usually the scheduler can’t place it. I use kubectl describe pod and look at the events. Typical causes are insufficient CPU or
memory on the nodes, node selectors or taints that don’t match, or an unbound persistent volume. The fix may be adjusting resource requests, scaling the node pool, or correcting the scheduling rules.”

16. The application is running but users can’t reach it. How do you troubleshoot?
“I work from the inside out. First, are the pods Ready? Second, does the Service have endpoints, using kubectl get endpoints,
 and do its selector labels match the pods? Third, is the Ingress configured correctly? Then I check the network rules, DNS, and the application logs. Readiness
probe failures are a common reason pods are removed from the service.”

18. What are readiness and liveness probes?
“A liveness probe tells Kubernetes whether the container is still healthy. If it fails, the container is restarted.
A readiness probe tells Kubernetes whether the container is ready to receive traffic. If it fails, the pod is taken out of the service but not restarted.
Wrongly configured probes are a common cause of restarts or traffic problems after a deployment.”

20. What are resource requests and limits?
“Requests are the amount of CPU and memory the scheduler reserves for a container. Limits are the maximum it can use.
 If a container exceeds its memory limit it gets OOMKilled, and if it exceeds its CPU limit it is throttled. Setting them properly helps scheduling and stability.”

22. How do you manage configuration and secrets?
“Non-sensitive configuration goes in ConfigMaps. Sensitive values go in Kubernetes Secrets, or better in Azure Key Vault,
so they are not stored in source code or images. Access is controlled with RBAC. [CONFIRM: what your project actually uses.]”

D. Monitoring and operations

17. How do you monitor the application on Azure?
“I use Azure Monitor and Log Analytics. For AKS, I look at node and pod CPU and memory, restart counts, and container logs.
I also use Azure Monitor alerts, which notify us when something looks abnormal, for example high resource usage or repeated restarts. This helps us catch issues before users report them.”

19. Give an example of a log query you’ve used.
“I use KQL in Log Analytics to filter container logs by namespace, pod name, time range and error keywords. For example,
I search the container logs of one deployment for ‘error’ or ‘exception’ in the last hour to find what happened around the time of a failure.
[CONFIRM: only mention queries you have really written, and practice 2-3 basic KQL queries.]”

21. What alerts did you work with, and what did you do when one fired?
“I reviewed alerts on application and infrastructure behavior, for example pod restarts or resource pressure.
When one fired, I checked its scope and severity, looked at the related logs and metrics, checked whether a recent deployment was involved,
and worked with the responsible team on the fix. [CONFIRM: add the specific alert types you handled.]”

23. How do you find the root cause of an incident?
“I build a timeline: when it started, what changed, and what the metrics and logs show. I check recent deployments, configuration
changes and dependencies. Then I separate the symptom from the cause. For example, a restarting pod might be caused by a memory limit, which might be caused by a code change.
 After the fix, we document the root cause and a preventive action.”

E. Scenarios (use your real stories, in STAR format)

21. Tell me about a deployment that failed and how you fixed it.
“[Replace with a real case.] Structure: Situation: a release to [environment] left pods in [state].
 Task: I had to find the cause and restore the service. Action: I used kubectl describe and logs and found
[e.g. a missing environment variable or wrong image tag], and coordinated the correction with the developers. Result: the deployment succeeded, and we added [a check or documentation] to prevent a repeat.”

23. Tell me about an issue found in production that you helped resolve.
“[Replace with a real case.] Use the same STAR format. If you haven’t handled a production issue directly,
 say so, and describe the part you did take, such as investigating logs or supporting the team.”

25. How do you work with developers during a failed release?
“I share evidence rather than opinions: the exact error, the logs, the pod events and what changed.
That helps the developers focus on the fix. I also keep the ticket and stakeholders updated while we work on it.”
