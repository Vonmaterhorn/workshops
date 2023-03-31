# Kubernetes Fundamentals - Defining Workloads (Part 3) - WIP

We've covered Sets and Deployment, which are all great to manage stateful and stateless applications. But what if we
want a more task-oriented application, one that exits successfully after it completes its work? Using a Deployment
doesn't make sense, it's designed for applications that should be running uninterruptedly, and will attempt to restart
if they exit. We can configure the `restartPolicy` to avoid restarting the application, but then, what if we want to
ensure these executions occur only at a given time? What if the trigger is _ad-hoc_? And what about scheduled tasks?

This brings us into the `Job` and `CronJob` territory.

# Jobs

A `Job` is a Kubernetes resource that represents an application that should run to completion and exit. Be it a shell
command, or a ruby application that sifts through
# Cronjobs
