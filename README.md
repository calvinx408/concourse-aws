### Auto-scaling Concourse CI on AWS with Terraform

http://www.slideshare.net/mumoshu/autoscaled-concourse-ci-on-aws-wo-bosh

## Recommended Usage: Using `concourse-aws` binary

1. Install [packer](https://github.com/mitchellh/packer) and [terraform(< 0.7.0)](https://github.com/hashicorp/terraform)
  * for now, concourse-aws supports only terraform(< 0.7.0) due to this issue: https://github.com/hashicorp/terraform/issues/7971
  
2. Create 1 VPC and 2 subnets in it

3. Clone this repository

```
git clone https://github.com/mumoshu/concourse-aws
```

4. Run `concourse-aws up`

```
cd concourse-aws

# Obtain the latest concourse-aws binary from GitHub releases
curl -L https://github.com/mumoshu/concourse-aws/releases/download/v0.0.4/concourse-aws -o concourse-aws && chmod +x concourse-aws

./build-amis.sh

./concourse-aws up
```

And then, `concourse-aws` will prompt you to provide required parameters(region, availability zone, subnet id, cidr, and vice versa)

### Upgrading Concourse workers with latest binaries

```
$ git pull --rebase origin master
$ ./build-concourse-ami.sh
$ vi cluster.yml # and update `ami_id` with the one produced by `build-concourse-ami.sh`
$ ./concourse-aws up
```

### Syncing configurations and states with S3

Users would sometime like to save and restore configurations and states on AWS resources, which is actually managed by terraform, with external data storage.  concourse-aws supports to save/restore states files with S3.  You can these operation with `save/restore` command like below:

```
# Save
# this will save configurations and states on AWS resources to S3 bucket.
./concourse-aws save --bucket <bucket_name> --bucket-region <region_of_bucket>
```

```
# restore
# this will pull configurations and states on AWS resources to S3 bucket.
./concourse-aws restore --bucket <bucket_name> --bucket-region <region_of_bucket>
```

Note: Saved/restored files
- `cluster.yml`: configuration file which can be generated by `concourse-aws` interactively.
-  SSH keys used for communicating between concourse servers.  These key files can be also automatically generated by `concourse-aws` interactively.
  - `host_key`,`host_key.pub`
  - `worker_key`,`worker_key.pub`
  - `session_signing_key`,`session_signing_key.pub`
  - `authorized_worker_keys`
- `terraform.tfstate`
  - states of AWS resources managed by terraform

## Advanced Usage: Using shell scripts and terraform directly

1. Install [packer](https://github.com/mitchellh/packer) and [terraform](https://github.com/hashicorp/terraform)

2. Create 1 VPC and 2 subnets in it

3. Set up required environment variables required by the wrapper script for terraform
   ```
   $ cat >> .envrc <<<'
   export AWS_ACCESS_KEY_ID=<YOUR ACCESS KEY>
   export AWS_SECRET_ACCESS_KEY=<YOUR SECRET ACCESS KEY>
   export CONCOURSE_IN_ACCESS_ALLOWED_CIDRS="<YOUR_PUBLIC_IP1>/32,<YOUR_PUBLIC_IP2>/32"
   export CONCOURSE_SUBNET_ID=<YOUR_SUBNET1_ID>,<YOUR_SUBNET2_ID>
   export CONCOURSE_DB_SUBNET_IDS=<YOUR_SUBNET1_ID>,<YOUR_SUBNET2_ID>
   '
   ```

   Install [direnv](https://github.com/direnv/direnv) and allow it to read `.envrc` created in the previous step.

   ```
   $ direnv allow
   ```

4. The same for optional ones
   ```
   $ export CONCOURSE_WORKER_INSTANCE_PROFILE=<YOUR INSTANCE PROFILE NAME>
   ```

5. Edit terraform variables and Run the following commands to build required AMIs and to provision a Concourse CI cluster
   ```
   $ ./build-amis.sh
   $ vi ./variables.tf
   $ ./terraform.sh get
   $ ./terraform.sh plan
   $ ./terraform.sh apply
   ```

6. Open your browser and confirm that the Concourse CI is running on AWS:
   ```
   # This will extract the public hostname for your load balancer from terraform output and open your default browser
   $ open http://$(terraform output | ruby -e 'puts STDIN.first.split(" = ").last')
   ```

7. Follow the Concourse CI tutorial and experiment as you like:
   ```
   $ export CONCOURSE_URL=http://$(terraform output | ruby -e 'puts STDIN.first.split(" = ").last')
   $ fly -t test login -c $CONCOURSE_URL
   $ fly -t test set-pipeline -p hello-world -c hello.yml
   $ fly -t test unpause-pipeline -p hello-world
   ```
   See http://concourse.ci/hello-world.html for more information and the `hello.yml` referenced in the above example.

8. Modify autoscaling groups' desired capacity to scale out/in webs or workers.

## Why did you actually created this?

[BOSH](https://github.com/cloudfoundry/bosh) looks [very promising to me according to what problems it solves](https://bosh.io/docs/problems.html).
However I was too lazy to learn it for now mainly because:

* I'm not going to use IaaS other than AWS for the time being
* learning it to JUST try Concourse CI might be too much in the short term though

## You may also find those projects useful

* [Concourse CI docker image](https://github.com/MeteoGroup/concourse-ci)
* [gregarcara/concourse-docker](https://github.com/gregarcara/concourse-docker)
* [jtarchie/concourse-docker-compose](https://github.com/jtarchie/concourse-docker-compose)
  * I wonder if I could run docker containers instead of concourse ci's standalone binaries using this
* Maybe more up-to-date than [starkandwayne/terraform-concourse](https://github.com/starkandwayne/terraform-concourse)
* [motevets/concourse-in-a-box](https://github.com/motevets/concourse-in-a-box) to quickly get concourse up-and-running on a single Ubuntu 14.04 EC2 instance

## Contributing

### Making changes

The concourse-aws binary needs to be built for every architecture and pushed to GitHub Releases manually whenever the concourse-aws binary has code changes. Every significant change to the functionality should result in a bump of the version number in the `version` file.
