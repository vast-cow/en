---
title: "Interactive SLURM Job Attachment with Bash"
description: "This script provides a streamlined method for attaching an interactive shell to a running SLURM job...."
pubDatetime: 2026-04-13T09:49:03.413Z
updatedDate: 2026-04-14T07:14:25.535Z
---

This script provides a streamlined method for attaching an interactive shell to a running SLURM job. It integrates several command-line tools to identify active jobs, optionally prompt the user for selection, and then attach a terminal session to the chosen job using `srun`.

## Overview

In high-performance computing (HPC) environments managed by SLURM, users often need to inspect or interact with running jobs. This script automates that workflow by:

* Verifying required dependencies
* Retrieving active job IDs for the current user
* Allowing interactive selection when multiple jobs exist
* Launching an interactive shell inside the selected job

## Dependency Checks

At the beginning, the script ensures that all required commands are available:

* `squeue`: Queries SLURM job information
* `jq`: Parses JSON output
* `fzf`: Provides an interactive selection interface
* `srun`: Attaches to or launches SLURM jobs

If any of these tools are missing, the script exits immediately with an error.

## Retrieving Running Jobs

The script uses `squeue` with JSON output to fetch running jobs for the current user:

```bash
squeue --user "$USER" --states Running --json
```

This output is then processed with `jq` to extract job IDs. It accounts for different JSON structures (`.jobs` or `.job_array`) and filters out empty entries. The final list is sorted and deduplicated.

## Handling Job Selection

The script adapts based on how many jobs are found:

* **No jobs**: Exits with a message indicating no running jobs.
* **One job**: Automatically selects that job.
* **Multiple jobs**: Uses `fzf` to present an interactive selection menu.

If the user cancels the selection, the script exits gracefully.

## Attaching to the Job

Once a job ID is determined, the script attaches an interactive shell using:

```bash
srun --jobid "$jobid" --overlap --pty /bin/bash --login -i
```

### Key Options Explained

* `--jobid`: Specifies the target job
* `--overlap`: Allows the new step to share resources with the existing job
* `--pty`: Allocates a pseudo-terminal
* `/bin/bash --login -i`: Starts an interactive login shell

The use of `exec` ensures that the script is replaced by the new shell process.

## Error Handling and Robustness

The script uses strict Bash settings:

```bash
set -euo pipefail
```

This enforces:

* Immediate exit on errors (`-e`)
* Undefined variable detection (`-u`)
* Proper error propagation in pipelines (`-o pipefail`)

These safeguards improve reliability, especially in production HPC environments.

## Conclusion

This script encapsulates a common operational need in SLURM-based systems: attaching to running jobs efficiently. By combining structured data parsing, interactive selection, and robust error handling, it reduces manual steps and minimizes user error in multi-job scenarios.

```bash
#!/usr/bin/env bash
set -euo pipefail

for cmd in squeue jq fzf srun column; do
  if ! command -v "$cmd" >/dev/null 2>&1; then
    echo "Error: '$cmd' not found." >&2
    exit 1
  fi
done

job_lines="$(
  squeue -u "$USER" --states Running --json \
    | jq -r '
        if .jobs then .jobs
        elif .job_array then .job_array
        else []
        end
        | map([
            (.job_id | tostring),
            (.name // "-"),
            (.partition // "-"),
            (.time_used // .run_time // "-"),
            (.nodes // .node_count // "-" | tostring),
            (.node_list // .nodelist // "-")
          ] | @tsv)
        | .[]
      '
)"

if [[ -z "$job_lines" ]]; then
  echo "No running jobs found." >&2
  exit 1
fi

job_count="$(printf '%s\n' "$job_lines" | grep -c '.')"

if (( job_count == 1 )); then
  jobid="$(printf '%s\n' "$job_lines" | cut -f1)"
else
  selected="$(
    {
      printf 'JOBID\tNAME\tPARTITION\tELAPSED\tNODES\tNODELIST\n'
      printf '%s\n' "$job_lines"
    } \
      | column -t -s $'\t' \
      | fzf \
          --prompt='Select job> ' \
          --height=50% \
          --reverse \
          --header-lines=1
  )"

  if [[ -z "${selected:-}" ]]; then
    echo "No job selected." >&2
    exit 1
  fi

  jobid="$(awk '{print $1}' <<< "$selected")"
fi

echo "Using jobid: $jobid" >&2
exec srun --jobid "$jobid" --overlap --pty /bin/bash --login -i
```
