+++
date = "2019-12-20T10:42:00+02:00"
draft = true
tags = [ "sops", "aws", "kms", "multi", "profile" ]
title = "sops with AWS KMS multi profile"
slug = "sops-aws-kms-multi-profile"
+++

For a while, I always wanted to store credentials in git repository. But security first, I have never done it. 
Recently, we have started using Airflow to manage all the different workflows in our teams. (ETL, scripts..). I'll write about in a few weeks.

Anyway, to manage Airflow variables and connections, which are sensitives files, I was looking for a simple but robust solution. We were using AWS secret manager, and python code to get/store our secret. But it's not free, and requires code.

We have thought about using Vault from Hashicorp, but it's overkill for our needs.


# Git-crypt ?

First attempt was to use git-crypt. Works well, but only with GPG, and not so simple to manage teams access.

# Sops

Then, looking for another solution on Github, I discovered [sops](https://github.com/mozilla/sops) on Mozilla's Github which seemed a perfect solution for our need. It's used by Mozilla to keep theirs secret.

`sops is an editor of encrypted files that supports YAML, JSON, ENV, INI and BINARY formats and encrypts with AWS KMS, GCP KMS, Azure Key Vault and PGP. `

AWS KMS ? Perfect, we use AWS Cloud, and already good fan of their KMS. 

### Sops, simple usage

Installation:

`brew install sops`

Encrypt a file in place:

`sops --kms arn:aws:kms:us-west-2:xxx:key/xxx -e -i secret.yaml`

Decrypt a file in place:

`sops --kms arn:aws:kms:us-west-2:xxx:key/xxx -d -i secret.yaml`

Edit a file:

`sops secret.yaml`

You can also use kms encryption with an iam role if you want to restrain to certain users.

To avoid repeating kms / iam role flag, you can create a `.sops.yaml` file:

```
$ cat .sops.yaml
creation_rules:
    - path_regex: .*/production/.*
      kms: arn:aws:kms:eu-west-2:xxx:key/xxx++arn:aws:iam::xxx:role/xxx

    - path_regex: .*/integration/.*
      kms: arn:aws:kms:eu-west-2:xxx:key/xxx

    - kms: arn:aws:kms:eu-west-2:xxx:key/xxx
```

You can specify an iam role with your kms key using this syntax:

```
<KMS ARN>+<ROLE ARN>
arn:aws:kms:eu-west-2:xxx:key/xxx+arn:aws:iam::xxx:role/xxx
```


# AWS KMS with assume-role / multi aws account / multi role

At work, we have multi teams using differents iam role (developers, data-scientists...). 
Iam users are created in AWS Organization and users use assume-role feature and profile to switch from one environment to another.

Different teams have access to same credentials file, so we don't specify an iam role when using sops.

All teams can have access to crendentials in lab environment.
Only Circleci and administrators can access to other environment (access managed by iam policies). There is a little trick to use sops with our needs.

Our `.sops.yaml` file looks like this :

```
creation_rules:
    - path_regex: .*/production/.*
      kms: arn:aws:kms:eu-west-2:xxx:key/xxx

    - path_regex: .*/integration/.*
      kms: arn:aws:kms:eu-west-2:xxx:key/xxx

    - kms: arn:aws:kms:eu-west-2:xxx:key/xxx
```

I can not use sops with `aws_profile`, an issue is open [https://github.com/mozilla/sops/issues/439](https://github.com/mozilla/sops/issues/439)

So when your aws credentials file contains multi profiles you need to export two environments variables before sops command.

```
AWS_SDK_LOAD_CONFIG=1 AWS_DEFAULT_PROFILE=sandbox sops -e -i settings/lab/connections.sh
#AWS_PROFILE : forces sops to use different profile from default
#AWS_SDK_LOAD_CONFIG=1 : AWS_PROFILE is only read if AWS_SDK_LOAD_CONFIG is enabled
```


# Conclusion

Sops is a nice find and already integrated in our stack. I only show you how to use it with AWS, but you can so much with.  Read their documentation if you are interested : [https://github.com/mozilla/sops](https://github.com/mozilla/sops)

