Creating a Recipe

You need a recipe to create a custom workflow for the backup and restore process.

The recipe custom resource has three basic specification elements:
Groups: A group defines a set of resources or PVCs that are processed together within a backup or restore. For example, a group can be a subset of specific resources that are specified with an exclude statement or an include statement.
Hooks: A hook can be used to start external scripts before and after snapshots, scale deployments up and down, or wait for certain conditions. For example, pods starting up and so on.
Workflows: A workflow defines the sequence of steps for a backup or restore operation, specifically the order in which the groups of resources and PVCs must be processed and any hooks that need to be applied.
For example:
apiVersion: spp-data-protection.isf.ibm.com/v1alpha1
kind: Recipe
metadata:
  name: wp-recipe
  namespace: wordpress
spec:
  appType: wordpress
  groups:
    - name: mysql_data
      type: volume
      labelSelector: app=wordpress
    - name: frontend
      labelSelector: tier in (frontend),app=wordpress
    - name: backend
      type: resource
      labelSelector: tier notin (frontend)
  hooks:
  - name: demoexechook
    type: exec
    namespace: ${GROUP.wp-app.namespace}
    nameSelector: wordpress-mysql.*
    ops:
    - name: pre
      command: >
        ["/bin/bash", "-c", "echo 'This is pretest' > /tmp/cfg_map_pre.txt"]
      container: mysql
    - name: post
      command: >
        ["/bin/bash", "-c", "echo 'This is posttest' > /tmp/cfg_map_post.txt"]
      container: mysql
  - name: demoscalehook
    type: scale  
    namespace: ${GROUP.wp-app.namespace}
    selectResource: statefulset
    nameSelector: wordpress$
  workflows:
  - name: backup
    sequence:
    - hook: demoscalehook/down
    - group: frontend
    - group: backend
    - hook: demoscalehook/sync
    - hook: demoexechook/pre
    - group: mysql_data
    - hook: demoexechook/post
    - hook: demoscalehook/up
  - name: restore
    sequence:
    - group: mysql_data
    - group: backend
    - group: frontend

Specifying a group
Groups that are defined in the recipe refine the scope of the parent group, which is completely defined by the application CR. For more information about specifying groups in the application CR, see Assigning a Recipe CR with an application CR.
The following fields are mandatory for creating a group:
name: name of the group that is specified in the workflow.
type: type of group, either volume or resource.
The following fields are optional for creating a group:
backupRef: Reference to a group used in a backup workflow. It is helpful for creating a subset of a backup group for a restore workflow.
includeResourceTypes: list of resource type names to include.
It applies to resource groups only.
For example, deployments.

If this field is not specified, then all resource types are included.
excludeResourceTypes: list of resource type names to exclude.
It applies to resource groups only.
For example, deployments.

excludeResourceTypes has precedence over includeResourceTypes.
essential: determines whether the group represents volumes or resources, which are essential to ensuring that the backup can produce a successful restore (true or false).
Groups specified as essential (default) need to be processed successfully. Otherwise, the overall backup operation fails.
If the backup operation fails, a rollback is initiated, and the workflow is not processed any further.
For non-essential groups, an unsuccessful operation of the group is tolerated; the processing of the workflow continues, and the overall backup is reported as successful while only this operation is marked as failed.
The flag is only considered whether the workflow is configured with fail-on=essential-error. See the workflow section for details.
labelSelector: select volumes and resources by label. For more information about on specifying labels, see Labels and Selectors.
nameSelector: select volume or associated workloads by object name.
This field applies to volume groups only.
When combined with labelSelector or logic applies.
selectResource: select which resource types the fields labelSelector and nameSelector must apply when you are selecting PVCs directly or indirectly through the workloads that the PVCs are associated with.
This field applies to volume groups only.
includeClusterResources: determines how cluster-scoped resources are processed (true, false, or not specified).
not specified: include only cluster-scoped resources that are associated with the namespace-scoped resources that are included in the groups.
false: do not include any cluster-scoped resources.
true: include all cluster-scoped resources (still considering other clauses and label selectors).
This field applies to only volume groups.
includedNamespaces: specify list of namespaces that are included.
Note: Use includedNamespaces and excludedNamespaces when you have applications that can stretch multiple namespaces. If the application contains only one namespace, then the namespace that is used in the recipe is obtained from the application (applications.application.isf.ibm.com) resource. In this case, make sure that your recipe is more portable by avoiding including the namespace declaration explicitly.
excludedNamespaces: specify a list of namespaces that are excluded.
This parameter has precedence over includedNamespaces.
Note: Use includedNamespaces and excludedNamespaces when you have applications that can stretch multiple namespaces. If the application contains only one namespace, then the namespace that is used in the recipe is obtained from the application (applications.application.isf.ibm.com) resource. In this case, make sure that your recipe is more portable by avoiding including the namespace declaration explicitly.
restoreOverwriteResources: Specify whether to overwrite resources during restore. By default, the value is false. If this field is set, then you must specify backupRef.
Specifying a hook
Hooks can be used to start external scripts before and after snapshots, scale deployments up and down, or wait for certain conditions (such as pods starting up). The following types of hooks are supported:
exec: Start arbitrary commands and scripts within target containers.
scale: scale workload resources up and down.
check: check for conditions on workload resources (such as readiness, replicaCount and so on).
The following fields are mandatory for creating a hook:
name: the name of the hook that is specified in the workflow.
The name must be unique within the Recipe CR.
namespace: the namespace to which the hook applies.
Namespace can be either specified directly (not recommended) or indirectly by the effective namespaces of a group ${GROUP..namespace} or using a variable from the application CR ${VARIABLE_NAME. In either case, you need to resolve to a single namespace.
type : the type of hook, either exec, scale, or check.
selectResource: workload resource type to that a hook applies. Specify the resource by using the fully qualified resource name. For example, orchestrator.aiops.ibm.com/v1alpha1/installations. The exception to this rule is that you can use the short names for commonly used resources pod, deployment, or statefulset (fully qualified resource name not required for these resources).
For exec hooks, defaults to pod.
This field is only required for hook types of scale or check.
If you add a workload resource type that is specified by using the fully qualified resource name, add additional read permissions for the resource by updating the the transaction manager clusterroles (ibm-backup-restore namespace). For example, if you plan to add the the custom resource orchestrator.aiops.ibm.com/v1alpha1/installations, update the transaction manager clusterroles with the following command:
oc edit clusterrole transaction-manager-ibm-backup-restore

Add the following fields to the clusterrole:

- verbs:
      - get
      - list
    apiGroups:
      - orchestrator.aiops.ibm.com
    resources:
      - installations

Note: If you specify a resource of type pod, deployment, or statefulset, you do not have to update the transaction manager clusterroles.
labelSelector: if specified, then the workload resource needs to match this label selector.
Either labelSelector or nameSelector is required. If both are specified, or logic applies.
nameSelector: if specified, then the workload resource needs to match this expression.
Either labelSelector or nameSelector is required. If both are specified, or logic applies.
The following fields are optional for creating a hook:
singlePodOnly: flag (true or false) that indicates whether to run a command on a single pod or on all pods that match the selector (when selectResource=pod). The hook is run on one of the matching pods only, but, which one is arbitrary.
For deployments and statefulsets (selectResource=deployment or statefulset), the option applies to each replicaset of the selected workload resources individually. For example, when two deployments are selected, the hook is run on one of the pods for each deployments replicaset, which pod is arbitrary.
This option applies only to exec hooks.
The default value is false.
essential: flag (true or false) that indicates whether the hook is essential for a successful backup. Hooks that are defined as essential (which is the default) need to be processed successfully. Otherwise, the overall backup operation is considered as failed. In this case, a rollback is initiated, and the workflow is not processed any further.
For nonessential hooks, an unsuccessful execution is tolerated. The processing of the workflow continues, and the backup is reported as successful.
The flag is only considered whether the workflow is configured with fail-on=essential-error.
onError: Determines the default behavior (fail or continue) if there is failures when an operation applies to multiple resources such as multiple pods. The option fail cancels the operation on the first occurrence of a failure, while continue continues processing of the remaining resources. In either case, the overall result of the operation is considered a failure.
timeout: The default timeout (an integer value specified in seconds) applies to custom and built-in operations. If not specified, the default is 300 seconds.
Ops specifications (optional)

ops: set of operations that the hook can be started for.
Exec hooks provide you to specify any number of custom operations.
Other hook types have built-in operations.
ops/name: name of the operation. This field needs to be unique within the hook.
ops/container: the container where the command must be run.
The default is to pick an arbitrary container within the pod.
ops/command: the command to run.
If you need multiple arguments, specify the command as a JSON array as single string, such as ["/usr/bin/uname", "-a"].
Note: If you want to save embracing quotation marks so that you can use it within the command, use > to start a block scalar.
Variable substitution is applied, both intrinsic group namespace variables (${GROUP...}) and variables from application CR can be used.
For intrinsic group namespace variables, if there are multiple namespaces it yields a comma-separated list without spaces.
ops/onError: specify fail or continue to provide the option to overwrite the identical option of the owning hook just for this operation. Defaults to the setting of the owning hook.
ops/timeout: timeout (an integer value specified in seconds) applied to the specified operation. If specified, it overrides the timeout that is specified on the hook level. The hook is considered in error if the command exceeds the timeout.
Defaults to timeout set on hook level.
ops/inverseOp: for rollback scenarios, it gives you the ability to rollback an operation by providing the name of another operation that reverses the effect of this operation.
For example, resume a database would be the revert operation for a failure on a suspend a database operation.

Chks specifications (optional)

chks: set of checks that the hook can apply to the target workload. Check that hooks provide an option to specify any number of custom checks based on conditions that are formulated as JsonPath like expressions.
chks/name: name of the check. The name must be unique within the hook.
chks/condition: the condition that needs to be true to release the hook. Conditions are specified as JsonPath like expressions.
The expression is expected to be a Boolean.
chks/onError: specify fail or continue to overwrite the identical option of the owning hook just for this operation. Defaults to the setting of the owning hook.
chks/timeout: timeout (an integer value specified in seconds) applied to the specified operation. If specified, it overrides the timeout that is specified on the hook level. The hook is considered in error if the command exceeds the timeout.
Examples of specifying conditions as JsonPath expressions:
Check to see if the number of available replicas matches the specified number of replicas.
condition: "{$.spec.replicas} == {$.status.readyReplicas}"

Check if a replica is ready.
condition: "{$.spec.replicas} == {$.status.readyReplicas}"

condition: "{$.status.containerStatuses[0].ready} == {True}"

Specifying a workflow
A workflow defines the sequence of steps for both backup and restore operations. At least one backup workflow (named "backup") and one restore workflow (named "restore") must be specified in the recipe CR. More or alternative workflows can be specified that can be referenced in on-demand backup or restore requests (overriding the default ones "backup" and "restore").

The following fields are mandatory for creating a workflow:
name: name of workflow.
The names "backup" and "restore" are reserved and implicitly used by default for backup or restore.
sequence: sequence of steps to be performed.
Sequence can refer to a series of groups, resources, and hooks in a specific order. The actions are processed sequentially in the specified order.
The following fields are optional for creating a workflow:

sequence/step/group: refer to a group specified.
sequence/step/hook: refer to a hook and one of its operations.
failOn: determines the behavior if failures when processing groups or hooks. The following is one of these values:
any-error: If this value is specified, the operation fails and performs defined rollback operations if any step of the workflow fails.
essential-error: if this value is specified, the operation fails and performs defined rollback operations if any step of the workflow that is defined as essential fails.
full-error: if this value is specified, the entire workflow is attempted. The workflow is only considered a complete failure if all essential steps fail.