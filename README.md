# Github Metrics

A metrics Action and API to record runtimes of github Actions workflow jobs. 

## Thesis

Github does not have a way of determining the aggregate runtime of jobs within workflows, 
so users are unable to know if their changes cause regressions within CI/CD pipelines.
By recording the time each job within a workflow takes to run, engineers will be able to 
understand how their changes affect everyone else's experience when running builds. 

Github Workflows have tow major pieces to them:

1. A Workflow, which typically starts when a Pull Request is created, or a merge occurs on a branch. 
Workflows consist of jobs that are pieced together in a certain order. 
2. A Job is a task to be executed inside of a Workflow. Multiple Jobs make up a workflow, 
and multiple Steps typically make up a Job. 

If we wrap a set of core Steps within a Job with Metrics Steps, 
one that gathers job data and starts a timer, and another that updates the results of the Job with 
additional metadata collected during execution of the Job by Github Actions in addition to ending the timer,
then we will be able to understand the runtime of a Job. Since Workflows are also just collections of Jobs, 
we'll also be able to understand the runtime of a given Workflow. 

Workflows typically have a Workflow Name and trigger based on what work the Workflow is supposed to accomplish. 
Here are some examples:
1. Branch CI - Perhaps you want to know before a PR is merged whether the branch is in a mergeable state.
This is where a Branch CI, or Pull Request CI Workflow would come in handy. Some of the Jobs within this Workflow
might also be shared with another type of CI Workflow, and comparisons between the two could be made. j
2. Main CI - You always want your main branch to be buildable, and to do so you run your CI tasks against it whenever
there is a new Merge event (either from PR merge or direct commit). Most of the Jobs within this Workflow are shared with
a Branch CI Workflow if one exists. 
3. Deployment or Publish - If you're building libraries or applications, typically you also deploy or publish code that is 'Released', 
for example building a library and creating a new Github Release then publishing that library to npm or maven central or publishing a
whole new version of your application to your Production Kubernetes cluster. 

The idea behind the Github Metrics Action and API is simple: 
1. Record certain metadata about a Github Repo's Workflows
2. Record runtimes of Jobs within each Workflow. 
3. Upload metadata to a central location via API call. 
4. Provide API to retrieve data for (eventual) visualization of Workflows and any Jobs that may be running slower than normal.

According to the Github Workflow documentation, the following information is available to us: 

- Workflow Name - ${{ github.workflow }} - string
- Run ID - ${{ github.run_id }} - number
- Run Number - ${{ github.run_number }} - number
- Actor - ${{ github.actor }} - string
- Repo - ${{ github.repository }} - string
- Event - ${{ github.event_name }} - string
- Commit SHA - ${{ github.sha }} - string
- Branch/Tag - ${{ github.ref }} - string
- Runner OS - ${{ runner.os }} - string
- Runner Arch - ${{ runner.arch }} - string 
- Runner Name - ${{ runner.name }} - string
- Start Time - ${{ env.START_TIME }} - timestamp / UTC datetime
- End Time - ${{ env.END_TIME }} - timestamp / UTC datetime
- Job Status - ${{ job.status }} - string
- Matrix Info - ${{ matrix }}  - json, optional
- Inputs Info - ${{ inputs }}  - json, optional 
- Additional Metadata - json, optional 

## Implementation Notes

### Github Action

Two Steps are needed, one is a simple step to record the Start Time of a Job. The other is a
Typescript implementation that gathers all the job information.

```yaml
# Record the start time into a GitHub Environment Variable
- name: Record start time
  run: echo "START_TIME=$(date +%s)" >> $GITHUB_ENV
```

The typescript implementation of the 2nd step would be a composite Action that requires a few GitHub Actions Secrets:

- A METRICS_TOKEN secret
- A METRICS_API_URL secret

This step will collect all the different variables we want to track. It will then make an HTTP POST request to the METRICS_API_URL using the METRICS_TOKEN secret for authentication

The request payload will be a JSON document with the following structure

```json
{
  "workflow_name": "${{ github.workflow }}",
  "run_id": "${{ github.run_id }}",
  "run_number": "${{ github.run_number }}",
  "actor": "${{ github.actor }}",
  "repo": "${{ github.repository }}",
  "event": "${{ github.event_name }}",
  "commit_sha": "${{ github.sha }}",
  "branch_tag": "${{ github.ref }}",
  "runner_os": "${{ runner.os }}",
  "runner_arch": "${{ runner.arch }}",
  "runner_name": "${{ runner.name }}",
  "start_time": "${{ env.START_TIME }}",
  "end_time": "${{ env.END_TIME }}",
  "job_status": "${{ job.status }}",
  "matrix_info": "${{ toJson(matrix) }}",
  "inputs_info": "${{ toJson(inputs) }}",
  "additional_metadata": {}
}
```

Example payload:

```json
{
  "workflow_name": "CI Workflow",
  "run_id": 123456789,
  "run_number": 42,
  "actor": "octocat",
  "repo": "my-org/fizzbuzz",
  "event": "push",
  "commit_sha": "1a2b3c4d5e6f7g8h9i0j",
  "branch_tag": "refs/heads/main",
  "runner_os": "ubuntu-latest",
  "runner_arch": "X64",
  "runner_name": "GitHub Actions Runner",
  "start_time": "2024-08-04T12:00:00Z",
  "end_time": "2024-08-04T12:01:40Z",
  "job_status": "success",
  "matrix_info": {
    "os": "ubuntu-latest",
    "node": 16
  },
  "inputs_info": {
    "greeting": "Hi",
    "name": "Bob"
  },
  "additional_metadata": {}
}
```

