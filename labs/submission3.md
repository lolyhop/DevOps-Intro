# Task 1

## 1. Workflow Creation

I created the workflow file `github-actions-demo.yml` in the `.github/workflows` directory.

The workflow is set to trigger on push and contains a single job Explore-GitHub-Actions that runs on ubuntu-latest. It executes several simple commands to demonstrate runner information and repository status.


## 2. Workflow Execution

I pushed a commit to trigger the workflow. GitHub automatically ran the workflow because the on: [push] event was defined in the YAML file.

**Workflow run screenshot:**

![](imgs/result1.png)


## 3. Observations
- The workflow started automatically when I pushed the commit;
- Steps executed sequentially within a single job;
- Runner information (OS, workspace path) was printed using predefined GitHub contexts like `${{ runner.os }}` and `${{ github.workspace }}`;
- Listing files in the workspace showed all repository contents available on the runner.
- Job status was reported at the end using `${{ job.status }}`.


## 4. Key Concepts Learned
- **Jobs:** Units of work in a workflow. Here, Explore-GitHub-Actions is one job;
- **Steps:** Commands or actions executed inside a job;
- **Runners:** Virtual machines or pods provided by GitHub to run jobs;
- **Triggers:** Events that start workflows (push in this workflow).
- **Contexts:** Variables like `${{ github.actor }}` and `${{ runner.os }}` provide runtime information.


# Task 2