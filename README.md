# EASY CI (cloned from [cloudfoundry/mega-ci](https://github.com/cloudfoundry/mega-ci))

This repo contains all public scripts and templates used to provision AWS resources and
deploy [BOSH](http://bosh.io/) and [Concourse CI](http://concourse.ci/). It should
be used in concert with a private repository that contains all necessary configuration
and secret information for your planned deployments.

#### Contents

1. [General Requirements](#general-requirements)
2. [Provisioning AWS for BOSH and Concourse, and Deploying BOSH](#provisioning-aws-for-bosh-and-concourse-and-deploying-bosh)
3. [Deploying Concourse](#deploying-concourse)

<a name="general-requirements"></a>
## General Requirements

* An AWS account for your deployments. It doesn't need to be empty as
  we can contain everything inside a VPC.

* The `aws` command line tool. This can be installed by running `brew install awscli`
  if you have Homebrew installed. You should run `aws configure` after
  installation to authenticate the CLI.

* The `bosh` command line tool.  This can be installed by running `gem install bosh_cli`

* The `bosh-init` command line tool. Instructions for installation can be found
  [here][bosh-init-docs].

* The `jq` command line tool. This can be installed by running `brew install jq`
  if you have Homebrew installed.

* The `spiff` command line tool. The latest release can be found [here]
  [spiff-releases].

* The `openssl` command line tool.

<a name="provisioning-aws-for-bosh-and-concourse-and-deploying-bosh"></a>
## Provisioning AWS for BOSH and Concourse, and Deploying BOSH

#### Usage

```bash
./scripts/setup_aws_bosh_for_concourse PATH_TO_DEPLOYMENT_DIR
```

See Requirements below for an explanation of the DEPLOYMENT_DIR.

This script will apply an AWS CloudFormation template, creating a stack in your AWS
account. It will then create a BOSH instance. The default username/password is for
your BOSH director is admin/admin. You are **strongly advised** to change these by
targetting the director and running:

```bash
bosh create user USERNAME PASSWORD
```

When you're done, create a file called `bosh_environment` at the root of your
deployment directory (see Requirements below) that looks like this:

```bash
export BOSH_USER=REPLACE_ME
export BOSH_PASSWORD=REPLACE_ME
export BOSH_DIRECTOR=REPLACE_ME_WITH_BOSH_DIRECTOR_IP
```

This file will be sourced by the script which deploys Concourse.

#### Requirements

The argument passed to the script must be the path to a "deployment directory".
For this script, it must have at least the following minimal structure.

```
my_deployment_dir/
|- aws_environment
|- certs/
|  |- (concourse.pem, optional)
|  |- (concourse.key, optional
|  |- (concourse_chain.pem, optional)
|- cloud_formation/
|  |- (properties.json, optional)
|- stubs/
   |- bosh/
      |- (bosh_passwords.yml, optional)
```

The `aws_environment` file should look like this:

```bash
export AWS_DEFAULT_REGION=REPLACE_ME # e.g. us-east-1
export AWS_ACCESS_KEY_ID=REPLACE_ME
export AWS_SECRET_ACCESS_KEY=REPLACE_ME
```

#### Optional Configuration

The `stubs/bosh/bosh_passwords.yml` contains internal BOSH passwords. If you do not provide one, one will be
generated with random passwords. The file should look like this:

```yaml
bosh_credentials:
  agent_password: REPLACE_WITH_PASSWORD
  director_password: REPLACE_WITH_PASSWORD
  mbus_password: REPLACE_WITH_PASSWORD
  nats_password: REPLACE_WITH_PASSWORD
  postgres_password: REPLACE_WITH_PASSWORD
  registry_password: REPLACE_WITH_PASSWORD
```

An SSL certificate for the domain where Concourse will be accessible is required.
If you do not provide a certificate, one will be created for you, with the Common Name
coming from the `ELBRecordSetName` parameter in the `cloud_formation/properties.json`
file (see below). The key and pem file must exist at `certs/concourse.key` and `certs/concourse.pem`.
If there is a certificate chain, it should exist at `certs/concourse_chain.pem`.
You can generate a self signed cert if needed:

* `openssl genrsa -out concourse.key 2048`
* `openssl req -new -key concourse.key -out concourse.csr` For the Common Name, you must enter your self signed domain.
* `openssl x509 -req -in concourse.csr -signkey concourse.key -out concourse.pem`
* Copy `concourse.pem` and `concourse.key` into the `certs` directory.

The optional `cloud_formation/properties.json` file should look like this:

```json
[
  {
    "ParameterKey": "ConcourseHostedZoneName",
    "ParameterValue": "REPLACE_WITH_HOSTED_ZONE_NAME"
  },
  {
    "ParameterKey": "ELBRecordSetName",
    "ParameterValue": "REPLACE_WITH_HOST_NAME"
  }
]

```
If both `ConcourseHostedZoneName` and `ELBRecordSetName` are provided, a Route 53 hosted zone will be created with the given
`ConcourseHostedZoneName` name, and a DNS entry pointing at the new ELB will be created with the given
`ELBRecordSetName` name.  Note that if you do not provide `certs/concourse.key` and `certs/concourse.pem`, then you must provide this file as the `ELBRecordSetName` is used as the Common Name for generating certs.

#### Output

The script generates several artifacts in your deployment directory:

* `artifacts/deployments/bosh.yml`: the deployment manifest for your BOSH instance
* `artifacts/deployments/bosh-state.json`: an implementation detail of `bosh-init`;
  used to determine things like whether it is deploying a new BOSH or updating an
  existing one
* `artifacts/certs/rootCA.[key,pem,srl]`: The root signing key used for
  creating the BOSH SSL certificate.
* `artifacts/certs/bosh.[crt,key]`: The SSL certificate and key used for BOSH.
  The certificate Common Name is the IP address of the BOSH director.
* `artifacts/keypair/id_rsa_bosh`: the private key created in your AWS
  account that will be used for all deployments; you will need this if you ever
  want to ssh into the BOSH instance or any of the VMs deployed by BOSH.
* `generated-stubs/pipeline/bosh-director-uuid.yml`: If you plan to use the BOSH director deployed here for Concourse to also deploy other things, you will need the Director UUID in this stub.
* `generated-stubs/pipeline/cf-resources.yml`: If you plan to use the BOSH director deployed here for Concourse to also deploy other things, you may need the data about AWS resources contained in this stub.
* (`stubs/bosh/bosh_passwords.yml`): If you do not provide this file, it will be generated for you.
* (`certs/concourse.key`): If you do not provide this file, it will be generated for you.
* (`certs/concourse.pem`): If you do not provide this file, it will be generated for you.

The script will also print the IP of the BOSH director. Target your director by running:
```bash
bosh target DIRECTOR_IP
```

<a name="deploying-concourse"></a>
## Deploying Concourse

#### Usage

Run:

```bash
./scripts/deploy_concourse PATH_TO_DEPLOYMENT_DIR
```

The script will deploy Concourse.

#### Requirements

The argument passed to the script must be the path to a "deployment directory".
For this script, it must have at least the following minimal structure.

```
my_deployment_dir/
|- aws_environment
|- bosh_environment
|- certs
|- stubs/
   |- concourse/
   |  |- (atc_credentials.yml, optional)
   |  |- binary_urls.json
   |- datadog/
   |  |- (datadog_stub.yml, optional)
   |- syslog/
      |- (syslog_stub.yml, optional)

```

The `aws_environment` file should look like this:

```bash
export AWS_DEFAULT_REGION=REPLACE_ME # e.g. us-east-1
export AWS_ACCESS_KEY_ID=REPLACE_ME
export AWS_SECRET_ACCESS_KEY=REPLACE_ME
```

The `bosh_environment` file should provide address and credentials of the BOSH director you wish to use to deploy Concourse:

```bash
export BOSH_USER=REPLACE_ME
export BOSH_PASSWORD=REPLACE_ME
export BOSH_DIRECTOR=REPLACE_ME_WITH_BOSH_DIRECTOR_IP
```

Finally, the `stubs/concourse/binary_urls.json` should look something like this:

```json
{
  "stemcell": "https://d26ekeud912fhb.cloudfront.net/bosh-stemcell/aws/light-bosh-stemcell-3087-aws-xen-hvm-ubuntu-trusty-go_agent.tgz",
  "concourse_release": "https://bosh.io/d/github.com/concourse/concourse?v=0.63.0",
  "garden_release": "https://bosh.io/d/github.com/cloudfoundry-incubator/garden-linux-release?v=0.305.0"
}
```

You can find the latest stemcells [here][bosh-stemcells]. Concourse (and associated garden releases) can be found [here][concourse-releases].

#### Optional Configuration

The `stubs/concourse/atc_credentials.yml` contains the basic auth credentials you will need to access the Concourse
web interface. If you do not provide a file, one will be generated with random passwords. The file should look like this:
```yaml
atc_credentials:
  basic_auth_username: REPLACE_ME
  basic_auth_password: REPLACE_ME
  db_name: REPLACE_ME
  db_user: REPLACE_ME
  db_password: REPLACE_ME
```

Concourse can optionally be configured to send metrics to datadog by adding your
datadog API key to datadog_stub with this format:

```yaml
---
datadog_properties:
  api_key: YOUR_DATADOG_API_KEY
```

Additionally, you can configure syslog on concourse to use an external endpoint
with the syslog_stub (i.e. papertrail):

```yaml
---
syslog_properties:
  address: logs3.papertrailapp.com:YOUR_PAPERTRAIL_PORT
```

#### Output

This script generates one additional artifact in your deployment directory:

* `artifacts/deployments/concourse.yml`: the deployment manifest of your Concourse

The script will also print the Concourse load balancer hostname at the end. This can be
used to create the `CNAME` for your DNS entry in Route53 so that you can have a nice
URL where you access your Concourse.

[concourse-releases]: https://github.com/concourse/concourse/releases
[bosh-init-docs]: https://bosh.io/docs/install-bosh-init.html
[bosh-stemcells]: http://bosh.io/stemcells
[spiff-releases]: https://github.com/cloudfoundry-incubator/spiff/releases
