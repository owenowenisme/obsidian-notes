## Table of Contents
   - [[Volcano Jobflow Enhancement#Expected Goal|Expected Goal]]
   - [[Volcano Jobflow Enhancement#Implementation Plan|Implementation Plan]]
       - [[Volcano Jobflow Enhancement#Support modifying parameters of a JobTemplate|Support modifying parameters of a JobTemplate]]
           - [[Volcano Jobflow Enhancement#Some Implementation...|Some Implementation...]]
       - [[Volcano Jobflow Enhancement#Lack of Robust Error Handling:|Lack of Robust Error Handling:]]
           - [[Volcano Jobflow Enhancement#Some Implementation...|Some Implementation...]]
           - [[Volcano Jobflow Enhancement#Expected Outcome|Expected Outcome]]
       - [[Volcano Jobflow Enhancement#Insufficient Control Flow Options|Insufficient Control Flow Options]]
   - [[Volcano Jobflow Enhancement#Conclusion|Conclusion]]

## Expected Goal
There are three main expected outcomes as written in [upstream issue](https://github.com/volcano-sh/volcano/issues/4275).
1. **Limited JobTemplate Customization:** Currently, when a JobTemplate is referenced within a JobFlow, it is used as is. Users lack the ability to make minor modifications to the JobTemplate's parameters for specific instances within the flow. This forces users to create multiple similar JobTemplates for slight variations, reducing reusability and increasing management overhead.
2. **Lack of Robust Error Handling:** When a job within a JobFlow fails, the entire flow might halt or require manual intervention. There is no built-in mechanism for automatically retrying failed jobs based on specific policies (e.g., maximum retries, backoff strategies). This reduces the robustness of JobFlow and can lead to longer overall execution times for complex workflows.
3. **Insufficient Control Flow Options:** The existing control flow primitives are basic. Supporting more advanced control structures like `if` statements, `switch` statements, and `for` loops would significantly enhance the expressiveness and capability of JobFlow, allowing users to define more intricate and dynamic job execution plans.
## Implementation Plan
### Support modifying parameters of a JobTemplate
I saw that there's a field `patch` in the [document](https://github.com/volcano-sh/volcano/tree/master/docs/design/jobflow#patch) of jobFlow, but I suppose it is not yet implemented, I don't see it exist in code anywhere.

This feature could bring us more flexibility when using jobTemplate with jobFlow, so users don't need to create new jobTemplate when there's only little diff between templates.

#### Some Implementation...
- Add patch field to Flow type
```go
type Flow struct {
	// +kubebuilder:validation:MinLength=1
	// +required
	Name string `json:"name"`
	// +optional
	DependsOn *DependsOn `json:"dependsOn,omitempty"`
	// +optional
	Patch *JobTemplateSpec `json:"jobTemplateSpec,omitempty"`
}
```
- Create a func `patchJobTemplate` in `jobflow_controller_action.go`
```go
func (jf *jobflowcontroller) patchJobTemplate(jobFlow *v1alpha1flow.JobFlow, flow v1alpha1flow.Flow) error {
	if flow.Patch == nil {
		return nil
	}

	patchBytes, err := json.Marshal(map[string]interface{}{
		"spec": flow.Patch,
	})
	if err != nil {
		return fmt.Errorf("failed to marshal patch: %v", err)
	}

	_, err = jf.vcClient.FlowV1alpha1().JobTemplates(jobFlow.Namespace).Patch(
		context.Background(),
		flow.Name,
		types.StrategicMergePatchType,
		patchBytes,
		metav1.PatchOptions{},
	)
	return 
```
- Check if need to patch before create Job 
```go
func (jf *jobflowcontroller) deployJob(jobFlow *v1alpha1flow.JobFlow) error {
	// load jobTemplate by flow and deploy it
	for _, flow := range jobFlow.Spec.Flows {
		jobName := getJobName(jobFlow.Name, flow.Name)
		if _, err := jf.jobLister.Jobs(jobFlow.Namespace).Get(jobName); err != nil {
			if errors.IsNotFound(err) {
				// If it is not distributed, judge whether the dependency of the VcJob meets the requirements
				if flow.DependsOn == nil || flow.DependsOn.Targets == nil {
					// patch the jobTemplate before creating job
					if err := jf.patchJobTemplate(jobFlow, flow); err != nil {
						return err
					}
					if err := jf.createJob(jobFlow, flow); err != nil {
						return err
					}
				} else {
					// query whether the dependencies of the job have been met
					flag, err := jf.judge(jobFlow, flow)
					if err != nil {
						return err
					}
					if flag { 
						// patch the jobTemplate before creating job
						if err := jf.patchJobTemplate(jobFlow, flow); err != nil {
							return err
						}
						if err := jf.createJob(jobFlow, flow); err != nil {
							return err
						}
					}
				}
				continue
			}
			return err
		}
	}
	return nil
}
```
- Add unit test to test patching JobTemplate of worked
### Lack of Robust Error Handling:
Many mature workflow tools provide some error Handling mechanism such as auto retry.
Take a popular workflow orchestrator - Airflow as example, Airflow provides 
- Retries: Number of automatic retry attempts
- Retry Delay: Wait time between retries
- Retry exponential backoff: Exponential increase in retry delay
These feature is basic for a mature workflow tools, and Volcano should also add these option to its Error Handling options.
#### Some Implementation...
Started with simple Retries, we can add more feature such as exponential backoff after this.
 We  provide flow retry for a more sophisticated error handling for now.
- Add field `MaxRetries` to allow defining max Retries when job failed.
```go
type Flow struct {
	// +kubebuilder:validation:MinLength=1
	// +required
	Name string `json:"name"`
	// +optional
	DependsOn *DependsOn `json:"dependsOn,omitempty"`
	// +optional
	MaxRetry int `json:"restartCount,omitempty"`
}
```
- And we can leveraging the `MaxRetry` of `Job` by propagating the MaxRetry into Job's Spec.
`jobflow_controller_action_test.go` 
Note that this can override the one defined in jobTemplate
``
```go
func (jf *jobflowcontroller) createJob(jobFlow *v1alpha1flow.JobFlow, flow v1alpha1flow.Flow) error {
	job := new(v1alpha1.Job)
	if err := jf.loadJobTemplateAndSetJob(jobFlow, flow.Name, getJobName(jobFlow.Name, flow.Name), job); err != nil {
		return err
	}
	if flow.MaxRetry > 0 { // propagate into job before creating
		job.Spec.MaxRetry = &flow.MaxRetry
	}
	if _, err := jf.vcClient.BatchV1alpha1().Jobs(jobFlow.Namespace).Create(context.Background(), job, metav1.CreateOptions{}); err != nil {
		if errors.IsAlreadyExists(err) {
			return nil
		}
		return err
	}
	jf.recorder.Eventf(jobFlow, corev1.EventTypeNormal, "Created", fmt.Sprintf("create a job named %v!", job.Name))
	return nil
}
```
- Finally, add a unit test for this to test the Job did performs retries before being marked.
#### Expected Outcome

User could specify their MaxRetry num through jobFlow yaml:
```yaml
apiVersion: flow.volcano.sh/v1alpha1
kind: JobFlow
metadata:
  name: test
  namespace: default
spec:
  jobRetainPolicy: delete
  flows:
    - name: a
    - name: b
      dependsOn:
        targets: ['a']
      maxRetry: 5
    - name: c
      dependsOn:
        targets: ['b']
      maxRetry: 5
    - name: d
      dependsOn:
        targets: ['b']
      maxRetry: 3
    - name: e
      dependsOn:
        targets: ['c','d']
	  maxRetry: 3
```
### Insufficient Control Flow Options
I'm not quite sure and don't have the context about how we are going to allow end users to use this feature.

If we are going to enable control flow in yaml the handling in controller will become too complex and will need a bunch of type check of condition, because this is not like jobDependency that you only need to check if the depended job is completed (As the logic in `judge` function). 

Therefore, I think we should further discuss about this.

## Conclusion
Through these enhancements, I believe that we could bring the functionality of Volcano JobFlow and JobTemplate to a higher level! These enhancements will transform Volcano JobFlow from a basic orchestrator into something teams can actually rely on for complex production workflows. I'm excited to contribute these improvements to the Volcano community and help make workflow orchestration on Kubernetes more accessible and robust.

This isn't just about adding features - it's about making Volcano JobFlow a tool that people actually want to use and can depend on when it matters.