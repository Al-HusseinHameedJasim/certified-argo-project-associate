# Certified Argo Project Associate (CAPA)

<img src="https://training.linuxfoundation.org/wp-content/uploads/2023/11/Training_Badge_CAPA-300x300.png" width="100">

My study notes as I prepared for the Certified Argo Project Associate (CAPA) exam.

According to [The Linux Foundation](https://training.linuxfoundation.org/certification/certified-argo-project-associate-capa):
> The Certified Argo Project Associate (CAPA) exam demonstrates a user’s ability to navigate the Argo Project ecosystem, including when and why to use each tool.

## Domains & Competencies
The Certified Argo Project Associate (CAPA) exam curriculum can be found [here](https://github.com/cncf/curriculum/blob/master/CAPA_Curriculum.pdf).

## Argo Workflows 36%
Argo Workflows is a lightweight, scalable, and cloud agnostic workflow engine implemented as a Kubernetes CRD for orchestrating parallel jobs on Kubernetes where each step is a container.

### Understand Argo Workflow Fundamentals
-	The Workflow is the most important resource in Argo Workflows and serves two important functions, defining the workflow, and storing the state.
-	Argo Workflows features include UI, artifact support (S3, Azure Blob Storage, Artifactory, etc.), templating, loops, parameterization, conditionals, retry, Prometheus metrics, Java/Golang/Python SDKs, SSO, CLI, capturing dependencies between tasks using a directed acyclic graph (DAG).
-	Use cases include ML pipelines, data and batch processing, infrastructure automation, and CI/CD.
-	Argo Workflows build machine learning pipelines.
-	The argo-server defaults to client authentication. Server authentication mode `--auth-mode=server` bypasses the UI login.
-	Volumes are useful to move large amounts of data from one step in a workflow to another. emptyDir volume is needed for output artifacts/parameters. Use cheap and fast emptyDir volumes instead of PVCs to share data between steps.
-	Alternative methods of building containers include Kaniko or Buildkit.
-	Workflows and DAGs use arguments while templates use inputs.
-	parameters and artifacts both share the name field, but one uses value and the other uses from.
-	To handle parameters with quotes, embed an expr expression in the conditional. For example: `when: "{{=inputs.parameters['may-contain-quotes'] == 'example'}}"`
-	A `LifecycleHook` triggers an action based on a conditional expression or on completion of a step or template.
-	Synchronization enables users to limit the parallel execution of certain workflows or templates within a workflow without having to restrict others.
-	Memoization is an optimization technique used primarily to speed up computer programs by storing the results of expensive function calls. Memoization is set at the template level. The cached data is stored in config-maps.
- In case of errors like `error creating cache entry: ConfigMap \"reuse-task\" is invalid: []: Too long: must have at most 1048576 characters`, this is due to the 1MB limit placed on the size of ConfigMap. Here are a couple of ways that might help resolve this: 
  -	Delete the existing ConfigMap cache or switch to use a different cache.
  -	Reduce the size of the output parameters for the nodes that are being memoized.
  -	Split your cache into different memoization keys and cache names so that each cache entry is small.
- A step isn't getting memoized, why not? If you are running workflows <3.5 ensure that you have specified at least one output on the step.
-	The resume, stop, and retry Argo CLI and API commands support a `--node-field-selector` parameter to allow users to select a subset of nodes to apply the command to.
-	Argo Workflows provides an indication of how much resource your workflow has used and saves this information. 
-	An attempt will be made to label the Workflow with the user who created it when it is created via the CLI or UI.
-	The Workflow of Workflows pattern involves a parent workflow triggering one or more child workflows, managing them, and acting on their results.
-	To set default Workflow values, use a configMap.
-	A workflow that utilizes work avoidance is simply a workflow containing steps that do not run if the work has already been done.
- UI Features:
  -	Artifact Visualization. Use cases include:
    -	Comparing ML pipeline runs from generated charts.
    -	Visualizing end results of ML pipeline runs.
    -	Debugging workflows where visual artifacts are the most helpful.
  -	Widgets are intended to be embedded into other applications using inline frames (iframe).
  -	Intermediate Parameters: Argo workflows has supported input parameters from UI. This feature is achieved via suspend template.
  -	Title and Description.
- Debugging tools
  -	Workflow Events.
  -	Debug Pause makes it possible to pause individual workflow steps for debugging. To pause a container env variables are used: `ARGO_DEBUG_PAUSE_AFTER` (or `BEFORE`).
-	Create a unique service account for each client. Create a Kubernetes secret to hold the token.
-	To create a Workflow from another use curl.
-	Plugins are disabled by default. To enable them, start the controller with `ARGO_EXECUTOR_PLUGINS=true`
-	By default, all workflow pods run as root. Run workflow pods more securely by configuring the security context for workflow pod.
-	Workflow will error in case of Pod deletion. A pod disruption budget can reduce the likelihood of this happening, but it cannot entirely prevent it. To retry pods that were deleted, set `retryStrategy.retryPolicy: OnError`.
- There are several strategies to reduce Kubernetes API overwhelming:
  -	Use the Emissary executor.
  -	Limit the number of concurrent workflows using parallelism.
  -	Rate-limit pod creation configuration.
  -	Set `DEFAULT_REQUEUE_TIME=1m`.
-	To reduce database overwhelming, reduce the number of records created by setting `DEFAULT_REQUEUE_TIME=1m`.
-	To delete the Workflow and its child CRD objects, patch the Workflow to remove the finalizer, set `forceFinalizerRemoval` to true, or use Argo CLI command `argo delete –force`.
- Argo CLI
  ```bash
  argo submit hello-world.yaml --watch # Submit a workflow. Use --watch flag to observe the workflow as it runs.
  argo list # List current workflows.
  argo get hello-world-xxx # Get info about a specific workflow @latest.
  argo logs hello-world-xxx # Print the logs from a workflow.
  argo delete hello-world-xxx # Delete workflow.
  argo submit arguments-parameters.yaml -p message="goodbye world" # Override parameter or --parameter-file params.yaml
  argo suspend WORKFLOW # Suspend a workflow
  argo resume WORKFLOW # Resume a suspended workflow
  argo template create template.yaml # Create a template
  argo cluster-template create clustertemplates.yaml # Create a ClusterWorkflowTemplate
  argo cron create cron.yaml # Create a cron
  argo submit --serviceaccount <name> # Specify which ServiceAccount Argo uses using when submitting Workflows
  ```

### Generating and Consuming Artifacts
-	Artifacts are packaged as Tarballs and gzipped by default. It can be customized by specifying an archive strategy.
-	There is built-in support for git, HTTP, GCS, and S3 artifacts.
-	For large artifacts, set `podSpecPatch` to increase the resource request for container.
-	Consider parameterizing S3 keys by `{{workflow.uid}}`.
-	Artifacts can be deleted `OnWorkflowCompletion` or `OnWorkflowDeletion`.
-	If Garbage Collection fails, view the logs of Pod named `<wfName>-artgc-*`.
-	A key-only artifact is an artifact where it is possible to only specify the key, omitting the bucket, secrets etc. When these are omitted, the bucket/secrets from the configured artifact repository are used which allows moving the configuration of the artifact repository out of the workflow specification.
- This should probably be the default as it has the following benefits:
  -	Reduces the size of workflows (improved performance).
  -	User owned artifact repository set-up configuration (simplified management).
  -	Decouples the artifact location configuration from the workflow, allowing the re-configuration of the artifact repository without changing workflows or templates.
-	Artifact Repository Ref: Reduce duplication in templates by configuring repositories that can be accessed by any workflow. For this, create a config map in either workflows namespace or in the managed namespace. This feature gives maximum benefit when used with key-only artifacts.
-	Use `fromExpression` under a Step/DAG level output artifact and expression under a Step/DAG level output parameter to set Step/DAG level artifacts or parameters based on an expression.

### Understand Argo Workflow Templates
-	A template is a task within a Workflow or a WorkflowTemplate under the field templates.
  Templates can be thought of as "functions".
- There are 6 types of templates, divided into two different categories.
- Template Definitions: Define work to be done, usually in a Container.
  -	Container: The most common template type. Container set template allows specifying multiple containers to run within a single pod. All container set templates that have artifacts must/should have a container named main. Do not use container set when there are lopsided requests.
  -	Script: A convenience wrapper around a container. The source: field allows defining a script in-place.
  -	Resource: Performs operations on Kubernetes cluster resources directly. The resource accepts `mergeStrategy` attribute when patching. `mergeStrategy` can be strategic (default), merge, or json. CRs cannot be patched with strategic.
  -	Suspend: Suspends a workflow execution either for a duration or until it is resumed manually.
- Template Invocators: Call other templates and provide execution control.
  -	Steps: Allows defining tasks in a series of steps.
  -	DAG: Allows defining tasks as a graph of dependencies.
- Other template types:
  -	HTTP Template can execute HTTP Requests. HTTP Templates use the Argo Agent, which executes the requests independently of the controller. The Agent and the Workflow Controller communicate through the `WorkflowTaskSet` CRD.
  -	Data Sourcing and Transformations templates can best be understood by looking at a common data sourcing and transformation operation in bash: `find -r . | grep ".pdf" | sed "s/foo/foo.ready/"`. Such operations consist of two main parts:
    -	A "source" of data: `find -r`.
    -	A series of "transformations" which transform the output of the source serially: `| grep ".pdf" | sed "s/foo/foo.ready/"`
  - Inline Templates can inline other templates within DAG and steps.
-	A WorkflowTemplate is a definition of a Workflow that lives in Kubernetes cluster.
-	Any existing Workflow can be converted to a WorkflowTemplate by substituting kind: Workflow to kind: WorkflowTemplate (except for priority) spec.
-	WorkflowTemplate in v2.4 - v2.6 must not have entrypoint field.
-	To automatically add labels and/or annotations to Workflows created from `WorkflowTemplates`, use `workflowMetadata`.
-	It is possible to create a Workflow from `WorkflowTemplate` spec using `workflowTemplateRef`. 
-	Users can specify options under enum when submitting WorkflowTemplates from the UI.
-	`ClusterWorkflowTemplates` are cluster scoped `WorkflowTemplates`.
-	Output parameters provide a way to use the result of a step as a parameter. For example: `{{tasks.generate-parameter.outputs.parameters.hello-param}}`.
-	The result output parameter captures standard output. It is accessible from the `outputs` map: `outputs.result`.
- There are two basic ways of running a template multiple time.
  -	`withItems` takes a list of things to work on. `'{{item}}'` is a JSON object where each element in the object can be addressed by it's key as `'{{item.key}}'`.
  -	`withParam`: A JSON array of items. The JSON can be generated in another step (creating a dynamic workflow) which is very powerful.
-	Templates can recursively invoke each other.
-	Conditional execution syntax is implemented by govaluate.
-	`retryStrategy` dictates how failed or errored steps are retried.
-	An empty `retryStrategy` (i.e. `retryStrategy: {}`) will cause a container to retry until completion.
-	Exit handler always executes at the end of the workflow. Some common use cases of exit handlers include cleaning up, sending notifications, submitting a workflow.
-	To delete a resource when the workflow is deleted, use Kubernetes garbage collection with the workflow resource as an owner reference.
-	daemon containers run in the background and get destroyed when the workflow exits the template scope in which the daemon was invoked. Daemon containers are useful for testing. The advantage of daemons over sidecar containers is that their existence can persist across multiple steps. They can be set like this: daemon: true.
-	Argo validates and resolves only the variable that starts with an Argo allowed prefix {"item", "steps", "inputs", "outputs", "workflow", "tasks"}
- There are two kinds of template tag:
  -	simple the default, e.g. `{{workflow.name}}`
  -	expression where {{ is immediately followed by =, e.g. `{{=workflow.name}}`.
- `TemplateDefaults` feature enables the user to configure the default template values in workflow spec level that will apply to all the templates in the workflow.

### Understand the Argo Workflow Spec
- The core structure of a Workflow spec is a list of templates and an entrypoint. The basic structure of a workflow spec:
  -	Kubernetes header.
  -	Spec body includes entrypoint and template definitions.
  -	Template definition includes name, input, output, and container invocation.
- The entrypoint field defines what the "main" function will be – that is, the template that will be executed first.
-	The values set in the `spec.arguments.parameters` are globally scoped and can be accessed via `{{workflow.parameters.parameter_name}}`.
-	`activeDeadlineSeconds` limits the elapsed time for a workflow.
-	`CronWorkflow` run on a preset schedule.
-	Cron Backfill are useful to re-run cron workflows for historical days.
-	`startingDeadlineSeconds` can be set to specify the maximum number of seconds past the last successful run of a CronWorkflow during which a missed run will still be executed.

### Work with DAG (Directed-Acyclic Graphs)
-	DAG is an alternative to specifying sequences of steps.
-	DAGs fail fast unless failFast is set to false. When one task fails, no new tasks will be scheduled.
-	Enhanced Depends is a feature for more complicated, conditional dependencies.
-	depends: "task || task-2.Failed"<br>
  is equivalent to:<br>
  depends: (task.Succeeded || task.Skipped || task.Daemoned) || task-2.Failed<br>
  dependencies: ["A", "B", "C"]<br>
  is equivalent to:<br>
  depends: "A && B && C"<br>
-	In DAG templates, the tasks prefix is used instead of steps.

### Run Data Processing Jobs with Argo Workflows
-	Running a Data Replication Pipeline on Kubernetes with Argo and Singer.io key takeaways.
  - Set up a Kubernetes cluster with Argo and MinIO storage for artifacts and config files.
  - Tell Argo where the default artifact repository is:<br>
    ```bash
    wget -o patch.yml https://raw.githubusercontent.com/stkbailey/data-replication-on-kubernetes/master/argo/argo-artifact-patch.yml
    kubectl patch configmap workflow-controller-configmap \
    -n argo \
    --patch “$(cat patch.yml)”
    ```
-	Deploy a Singer tap-to-target workflow.
- Reasons why to use Argo Workflows:
  -	Cheaper than an enterprise service.
  -	Familiarity with containerized applications.
  -	Kubernetes fits with the rest of the company’s infrastructure, making collaboration easier.

## Argo CD 34%
Argo CD is a declarative, GitOps continuous delivery controller for Kubernetes. Argo CD acts as the software agent that automatically pulls the desired state from a specific source, and continuously reconciles it with the live state of the cluster.

### Understand Argo CD Fundamentals
- Argo CD Components
  - API Server is a gRPC/REST server which exposes the API. It has the following responsibilities:
    - Application management and status reporting.
    - Credential management and authentication.
  - Repository Server is an internal service which maintains a local cache of the Git repository holding the application manifests.
  - Application Controller is implemented as a Kubernetes controller which continuously monitors running applications and compares the current, live state against the desired target state specified in the Git repo.
-	Argo CD features include automated deployment to multiple clusters, support for Kustomize, Helm, etc., SSO Integration, Multi-tenancy and RBAC, rollback, health status, UI, CLI, access tokens for automation, webhook integration, PreSync, Sync, PostSync hooks to support blue/green & canary upgrades, audit trails, Prometheus metrics, Parameter overrides for overriding helm parameters in Git.
-	Argo CD has two types of installations: multi-tenant and core.
-	Two types of installation manifests are provided: Non-High Availability and High Availability.
-	The Argo CD Core installation is primarily used to deploy Argo CD in headless mode. The bundle does not include the API server, RBAC model and authentication, or UI, and installs the lightweight (non-HA) version of each component. Argo CD Core use cases include relying on Kubernetes RBAC and API only. The Argo CD controller uses Redis as an important caching mechanism reducing the load on Kube API and in Git. To use Argo CD CLI in core mode, it is required to pass the --core flag.
-	Application is a group of Kubernetes resources as defined by a manifest. This is a Custom Resource Definition (CRD).
-	Application source type means which tool is used to build the application.
-	Target state is the desired state of an application as specified in Git repository.
-	Live state is the of application, i.e., what pods etc are deployed. A deployed application whose live state deviates from the target state is considered OutOfSync.
-	Sync status determines whether or not the live state matches the target state.
-	Sync status answers whether the observed application resources state deviates from the resources state stored in the Git repository. The logic behind deviation calculation is equivalent to the logic of the kubectl diff command.
-	Sync is the process of making an application move to its target state.
-	Sync operation status determines whether or not a sync succeeded.
-	The possible values of a sync status are in-sync and out-of-sync.
-	Refresh Compare the latest code in Git with the live state. Figure out what is different.
-	Health The health of the application, is it running correctly? Can it serve requests?
-	Tool A tool to create manifests from a directory of files. E.g., Kustomize. 
-	Configuration management tool See Tool.
-	Configuration management plugin A custom tool.
-	Register an external cluster's credentials to Argo CD to deploy apps to.
-	`https://kubernetes.default.svc` is equivalent to in-cluster for cluster name as a destination.
-	Ensure label part-of: argocd exists for ConfigMaps. Annotate ConfigMap resources using the label app.kubernetes.io/part-of: argocd, otherwise Argo CD will not be able to use them.
-	It is not possible to have the same application name for two different applications.
- Application CRD is the Kubernetes resource object representing a deployed application instance in an environment. It is defined by two key pieces of information:
  -	source reference to the desired state in Git (repository, revision, path, environment).
  -	destination reference to the target cluster and namespace.
- Without the resources-finalizer.argocd.argoproj.io finalizer, deleting an application will not delete the resources it manages.
-	App of Apps means creating an app that creates other apps, which in turn can create other apps.
-	The AppProject CRD is the Kubernetes resource object representing a logical grouping of applications.
-	A Project provides a logical grouping of Applications, isolates teams from each other, and allows for fine-tuning access control in each Project.
- In order for an application to be managed and reconciled outside the Argo CD's control plane namespace, two prerequisites must match:
  -	The Application's namespace must be explicitly enabled using the --application-namespaces parameter.
  -	The AppProject referenced by the .spec.project field of the Application must have the namespace listed in its .spec.sourceNamespaces field.
-	Do not add user controlled namespaces in the .spec.sourceNamespaces field of any privileged AppProject like the default project.
-	Set up network policies to prevent direct access to Argo CD components (Redis and the repo-server). 
-	Run Argo CD alone on its own cluster.
-	Enable password authentication on the Redis instance.
-	Deep links allow users to quickly redirect to third-party systems, such as Splunk, Datadog, etc. from the Argo CD user interface.
-	The argocd admin notifications is a CLI command group that helps to configure the controller settings and troubleshoot issues.
-	Failed to parse new settings means there is an error in converting YAML to JSON, i.e., YAML syntax is incorrect.
-	service type 'xxxx' is not supported due to incompatible argocd-notifications controller version.
-	notification service 'xxxx' is not supported means 'xxxx' is not defined in argocd-notifications-cm or to fail to parse settings.
-	The ApplicationSet controller makes it possible to deploy to multiple clusters. The ApplicationSet controller provides the ability to target multiple Kubernetes clusters and improved support for monorepos.
-	ApplicationSet use cases are cluster add-ons, monorepos, and self-service of Argo CD Applications on multitenant clusters.
-	For security reasons, only admins may create/update/delete ApplicationSets.
- When To Use Parameter Overrides?
  -	A team maintains a "dev" environment, which needs to be continually updated with the latest version of their application after every build in the tip of main.
  -	A repository of Helm manifests is already publicly available (e.g. https://github.com/helm/charts).
- Use cases for hooks include:
  -	Using a PreSync hook to perform a database schema migration.
  -	Using a Sync hook to orchestrate a complex deployment.
  -	Using a PostSync hook to run integration and health checks after a deployment.
  -	Using a SyncFail hook to run clean-up or finalizer logic if a Sync operation fails.
- Best Practices
  -	Separating Config Vs. Source Code Repositories.
  -	Leaving Room for Imperativeness.
  -	Ensuring Manifests at Git Revisions Are Truly Immutable.
-	Configure Argo CD to ignore fields when differences are expected.
-	The Ingress, StatefulSet and SealedSecret types have known issues which might cause health check to return Progressing state instead of Healthy.
-	To disable admin user, add admin.enabled: "false" to the argocd-cm ConfigMap.
-	For air-gapped environments, use an Argo instance per environment. Make sure that requirements.yaml uses only internally available Helm repositories for air-gapped environment.
-	Cluster secret should have argocd.argoproj.io/secret-type: cluster label.
-	To fix Out Of Sync Even After Syncing, change the label app.kubernetes.io/instance by setting the application.instanceLabelKey value in the argocd-cm. It is recommended to use argocd.argoproj.io/instance.
-	The manifest generation in Argo CD is implemented by the argocd-repo-server component. Manifest generation requires downloading Git repository content and producing ready-to-use manifest YAML.
-	Given that it is too time-consuming to download the whole repository content every time, Argo CD caches the repository content on a local disk and uses the git fetch command to download only recent changes from the remote Git repository.
-	The reconciliation phase is implemented by the argocd-application-controller component. The controller loads the live Kubernetes cluster state, compares it with the expected manifests provided by the argocd-repo-server, and patches deviated resources.
-	The reconciliation results must be presented to end-users which is done by the argocd-server component.
-	Argo CD can deploy into the external cluster as well as into the same cluster where it is installed. The cluster where Argo CD is installed to deploy to multiple clusters is called control-plane cluster.
-	Resource hooks allow encapsulating the application synchronization logic, so we don’t have to use scripts and continuous integration tools. However, some of such use cases naturally belong to continuous integration processes, and it is still preferable to use tools like Jenkins.
-	Argo CD provides a feature called resource hooks. These hooks allow running custom scripts, typically packaged into a Pod or a Job, inside of the Kubernetes cluster during the synchronization process. The purpose is to perform some steps like setting the maintenance page, executing database migration, etc.
-	One such use case is post-deployment verification. The challenge here is that GitOps deployment is asynchronous by nature. After the commit is pushed to the Git repository, we still need to make sure that changes are propagated to the Kubernetes cluster.
-	Argo CD makes this issue trivial by providing tools that help to monitor application status. The argocd app wait command monitors the application and exits after the application reaches a synced and healthy state. As soon as the command exits, you can assume that all changes are successfully rolled out, and it is safe to start post deployment verification. The argocd app wait command is often used in conjunction with argocd app sync.
argocd app sync sample-app && argocd app wait sample-app
-	For the SSO, after performing the configuration on the OpenID Connect provider side (such as Azure AD), add the corresponding configuration to the argocd-cm ConfigMap.
-	Argo CD supports both SSO and RBAC integration for enterprise-level SSO and access control.
-	For production, it is recommended to use Autopilot, a companion project that commits all configuration to git so Argo CD can manage itself using GitOps. Argo CD comes with its own custom resources that can be stored in Git and applied to a cluster using Argo CD itself.
-	Argo CD is extensible.
-	Argo CD uses Helm only as a templating mechanism, which is the reason why helm list command doesn't return anything.
-	Argo CD UI can be made available to external traffic via a Kubernetes ingress.
-	It is possible to change the way Argo CD learns about Git changes. It can use Git webhooks (which are more efficient, with no delays) instead of polling a Git provider.
- The possible values of health status are:
  -	“Healthy” -> Resource is 100% healthy.
  -	“Progressing” -> Resource is not healthy but still has a chance to reach healthy state.
  -	“Suspended” -> Resource is suspended or paused. The typical example is a cron job.
  -	“Missing” -> Resource is not present in the cluster.
  -	“Degraded” -> Resource status indicates failure or resource could not reach healthy state in time.
  -	“Unknown” -> Health assessment failed, and actual health status is unknown.
- Working with Argo CD:
  -	Install Argo CD as a controller in the Kubernetes cluster.
  -	Store deployment manifests in Git.
  -	Create an Argo CD application by defining which Git repository to monitor and to which cluster/namespace this application should be installed.
-	argocd-vault-plugin retrieves secrets from Secret Managers and inject them into Kubernetes secrets.
-	The initial admin password of Argo CD is auto generated to be the pod name of the Argo CD API server that can be retrieved using the command below:<br>
  ```$ kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2```
- Argo CD CLI
  ```bash
  argocd app list # To see the status and configuration of the application
  argocd app get <appName> # For more details
  argocd app sync <appName> # To sync the app
  ```
### Synchronize Applications Using Argo CD
-	Automated Sync Policy
-	Automatic Pruning (automatic deletion)
-	Automatic Pruning with Allow-Empty (allows applications having empty resources)
-	Automatic Self-Healing (automatic sync when the live cluster's state deviates from the state defined in Git)
-	Rollback cannot be performed against an application with automated sync enabled.
-	Some Sync Options can be defined as annotations.
- Sync Options
  -	No Prune Resources `Prune=false`
  -	Disable Kubectl Validation `Validate=false`
  -	Skip Dry Run for new custom resource types `SkipDryRunOnMissingResource=true`
  -	Stop resources from being cleaned up during app deletion `Delete=false`
  -	Selective Sync `ApplyOutOfSyncOnly=true`
  -	Resources Prune Deletion Propagation Policy `PrunePropagationPolicy=foreground`
  -	Prune Last `PruneLast=true`
  -	Replace Resource Instead of Applying Changes `Replace=true`
  -	Server-Side Apply `ServerSideApply=true`
  -	Fail the sync if a shared resource is found `FailOnSharedResource=true`
  -	Respect ignore difference configs `RespectIgnoreDifferences=true`
  -	Create Namespace `CreateNamespace=true`
- There are 3 parameters that can be changed when defining the sync strategy:
  - Manual or automatic sync.
  - Auto-pruning of resources. This is only applicable for the automatic sync. It defines what Argo CD does when files are deleted from Git. If it is enabled, Argo CD will also remove the respective resources in the cluster as well.
  - Self-Heal of the cluster. This is only applicable for the automatic sync. It defines what Argo CD does when changes are made directly to the cluster (via kubectl, for instance). If enabled, then Argo CD will discard the extra changes and bring the cluster back to the state described in Git.
- When Argo CD starts a sync, it orders the resources in the following precedence:
  - The phase
  - The wave they are in (lower values first)
  - By kind (e.g. namespaces first and then other Kubernetes resources, followed by custom resources)
  - By name
-	Sync windows are configurable windows of time where syncs will either be blocked or allowed.
-	There are two types of synchronization strategies: "hook", which is the default value, and "apply".
- By default, the sync period is 3 minutes. To change the default period, edit argocd-cm configMap:
  ```yaml
  ...
  data:
    timeout.reconciliation: 180s
  ...
  ```
### Use Argo CD Application
-	The Application CRD is the most significant resource introduced by Argo CD. It declaratively defines the deployment process for manifests into Kubernetes.
-	The Application provides a logical grouping of Kubernetes resources and defines a resources manifest’s source and destination.
-	An application can be created in Argo CD from the UI, CLI, or by writing a Kubernetes manifest that can then be passed to kubectl to create resources.
- Creating an Argo CD application in the UI:
  - Navigate to the +NEW APP on the left-hand side of the UI.
  - Add the following to create the application:
    - General Section:
      - Application Name: <appName>
      - Project: <projectName>
      - Sync Policy: Automatic
    - Source Section:
      - Repository URL/Git: the repository URL
      - Branches: main
      - Path: <./path>
    - Destination Section:
      - Cluster URL: select the cluster URL
      - Namespace: <namespace>
  - Click Create.
- Creating an Argo CD application with the argocd CLI
  ```bash
  argocd app create <appName> \
  --project <projectName> \
  --repo <repositoryURL> \
  --path <./path> \
  --dest-namespace <namespace> \
  --dest-server <serverURL>
  ```
- The serverURL is the URL of the cluster where the application will be deployed. `https://kubernetes.default.svc` references the same cluster where Argo CD is deployed.

### Configure Argo CD with Helm and Kustomize
- The following configuration options are available for Kustomize:
  -	`namePrefix` is a prefix appended to resources for Kustomize apps.
  -	`nameSuffix` is a suffix appended to resources for Kustomize apps.
  -	`images` is a list of Kustomize image overrides.
  -	`replicas` is a list of Kustomize replica overrides.
  -	`commonLabels` is a string map of additional labels.
  -	`forceCommonLabels` is a boolean value which defines if it's allowed to override existing labels.
  -	`commonAnnotations` is a string map of additional annotations.
  -	`namespace` is a kubernetes resources namespace.
  -	`forceCommonAnnotations` is a boolean value which defines if it's allowed to override existing annotations.
  -	`commonAnnotationsEnvsubst` is a boolean value which enables env variables substitution in annotation values.
  -	`patches` are a list of Kustomize patches that supports inline updates.
-	Helm has the ability to set parameter values, which override any values in a values.yaml. For example:<br>
  `helm template . --set service.type=LoadBalancer` <br>
 	Argo CD can override values using Argo CD CLI: <br>
 	`argocd app set helm-guestbook -p service.type=LoadBalancer` <br>
 	In declarative syntax:
    ```yaml
    source:
      helm:
        parameters:
          - name: "service.type"
            value: LoadBalancer
    ```
-	To use Kustomize with an overlay, point the path to the overlay.
-	Use the `IgnoreExtraneous` compare option for generated resources.
-	Patches allow for kustomizing without kustomization file.

## Argo Rollouts 18%
Argo Rollouts is a Kubernetes controller and set of CRDs which provide advanced deployment capabilities such as blue-green, canary, canary analysis, experimentation, and progressive delivery features to Kubernetes.

### Understand Argo Rollouts Fundamentals
-	Argo Rollouts give the ability to “test” the new version by introducing a “preview” Kubernetes service, in addition to the service used for live traffic. After some time (the exact amount is configurable in Argo Rollouts), the old version is scaled down completely.
-	Argo Rollouts offers an automatic decision if the deployment should continue or not based on application metrics. Argo Rollouts supports several metric providers such as Prometheus, NewRelic, Cloudwatch, etc. or a custom URL.
-	Argo Rollouts introduces the concept of Analysis which gets deployed along with the Rollout and defines the thresholds for metrics.

### Describe AnalysisTemplate and AnalysisRun
-	Analysis is the capability to connect a Rollout to metrics provider and define specific thresholds for certain metrics that will decide if an update is successful or not.
-	For performing an analysis, Argo Rollouts includes two custom Kubernetes resources: AnalysisTemplate and AnalysisRun.
-	AnalysisTemplate contains instructions on what metrics to query. The actual result that is attached to a Rollout is the AnalysisRun custom resource.

## Argo Events12%
Argo Events is an event-based dependency manager for Kubernetes which helps defining multiple dependencies from a variety of event sources like webhook, s3, schedules, streams etc. and trigger Kubernetes objects after successful event dependencies resolution.

### Understand Argo Event Components and Architecture
- Main components of Argo Events are Event Source, Sensor, Eventbus, and Trigger.<br>

  <img src="https://argoproj.github.io/argo-events/assets/argo-events-architecture.png" width="900">

-	Event Source defines the configurations required to consume events from external sources like AWS SNS, SQS, GCP PubSub, Webhooks, etc. It further transforms the events into the cloud events and dispatches them over to the eventbus.
-	Sensor defines a set of event dependencies (inputs) and triggers (outputs). It listens to events on the eventbus and acts as an event dependency manager to resolve and execute the triggers. Event dependency is an event the sensor is waiting to happen. EventSources publish the events while the Sensors subscribe to the events to execute triggers.
-	EventBus is a Kubernetes Custom Resource which is used for event transmission from EventSources to Sensors. EventBus acts as the transport layer of Argo-Events by connecting the EventSources and Sensors. There are three implementations of the EventBus: NATS (deprecated), Jetstream, and Kafka. EventBus is namespaced. The common practice is to create an EventBus named default in the namespace. For other names or multiple EventBus in one namespace, specify eventBusName in the spec of EventSource and Sensor correspondingly.
Use Kafka in case of many events and want to horizontally scale Sensors. To get started quickly, use Jetstream. The Kafka EventBus requires one event topic and two additional topics (trigger and action) per Sensor.
-	Trigger is the resource/workload executed by the sensor once the event dependencies are resolved. Trigger types include AWS Lambda, Apache OpenWhisk, Argo Rollouts, Argo Workflows, Custom - Build Your Own, HTTP Requests - Serverless Workloads (OpenFaaS, Kubeless, KNative etc.), Kafka Messages, NATS Messages, Slack Notifications, Azure Event Hubs Messages, and create any Kubernetes Objects, Log (for debugging event bus messages).

## References
1. [Argo Workflows Documentation](https://argo-workflows.readthedocs.io/en/latest).
2. [Argo CD Documentation](https://argo-cd.readthedocs.io/en/latest).
3. [Argo Rollouts Documentation](https://argoproj.github.io/argo-rollouts).
4. [Argo Events Documentation](https://argoproj.github.io/argo-events).
