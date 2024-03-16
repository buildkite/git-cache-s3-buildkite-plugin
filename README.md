# git-cache-s3-buildkite-plugin

An experimental Buildkite plugin for caching git repositories on S3.

It's not unusual of for git repos on some projects to reach hundreds of
megabytes (or worse!), particularly when submodules are involved or projects
have a lot of history.

When agents are running on AWS this can become expensive. It's good practice to
run agents on private subnets and use a NAT gateway to connect to the Internet, but
with enough builds and/or jobs pulling the repo the AWS data transfer costs can
get expensive fast.

Transferring data from S3 is free though, and a lot faster than using the git
protocol to pull hundred of megabytes from an external source code host. This
plugin explores:

* caching the git directory for a pipeline on S3
* the first time a pipeline runs a job on an agent, download the cached git directory and seed the working directory with it
    * the agent can then fetch just the git objects that have changed since the cache was created, which should typically be Not Much Data
* every so often update the cache on S3 so the delta from the upstream repo doesn't grow very large

## Usage

Add the plugin to your steps like this:

```
steps:
  - name: ":rspec: Run"
    plugins:
      - yob/git-cache-s3#main:
          bucket: my-buildkite-git-cache
```

The cached git data will be stored in:

    s3://<bucket-name>/<buildkite-org-slug>/<buildkite-pipeline-slug>/cache-<timestamp>.git.tar

The IAM role being used by the agent should have permission to read/write `s3://<bucket-name>/<buildkite-org-slug>/<buildkite-pipeline-slug>/*`. Something like:

```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Action": [
				"s3:PutObject",
				"s3:ListBucket",
				"s3:GetObject"
			],
			"Effect": "Allow",
			"Resource": [
				"arn:aws:s3:::<bucket>/<buildkite-org-slug>/<buildkite-pipeline-slug>/*",
				"arn:aws:s3:::<bucket>"
			]
		}
	]
}
```

## Why a pluigin?

A plugin is an awkward way to do this, because it has to be explicitly added to
each step in each pipeline.yml.

I did it this way for easy prototyping, but if it works well the hooks would
make more sense as agent hooks installed on AWS hosted agents. With a bit of
tweaking they could be safe no-ops for pipelines that don't want or need
caching and otherwise on by default for any job that runs on the agents.
