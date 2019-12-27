+++
date = "2019-12-20T10:42:00+02:00"
draft = false
tags = [ "sops", "aws", "kms", "multiple", "profiles" ]
title = "sops with AWS KMS multiple profiles"
slug = "sops-aws-kms-multiple-profiles"
+++

For a while, I have been wanting to store credentials in git repository. But I have never done it because of security.
Recently, we have started using Airflow to manage all the different workflows in our teams. (ETL, scripts..). I will write about it in a few weeks.

Anyway, to manage Airflow variables and connections, which are sensitives files, I was looking for a simple but robust solution. Up recently, we were using AWS secret manager, and python code to get/store our sensitive informations. But it is not free, and requires code.

We have thought about using Vault from Hashicorp, but it's overkill for our needs.


# Git-crypt?

First attempt was to use git-crypt. It works well, but only with GPG, and not so simple to manage teams access.

# Sops

Then while looking for another solution on Github, I came across [sops](https://github.com/mozilla/sops) on Mozilla's Github which seemed a perfect solution for our needs. It is used by Mozilla to keep their secrets.

`sops is an editor of encrypted files that supports YAML, JSON, ENV, INI and BINARY formats and encrypts with AWS KMS, GCP KMS, Azure Key Vault and PGP. `

AWS KMS? Perfect, we use AWS Cloud, and are already big fan of their KMS. 

### Sops, simple usage

Installation:

`brew install sops`

Encrypt a file in place:

`sops --kms arn:aws:kms:us-west-2:xxx:key/xxx -e -i secret.yaml`

Decrypt a file in place:

`sops --kms arn:aws:kms:us-west-2:xxx:key/xxx -d -i secret.yaml`

Edit a file:

`sops secret.yaml`

You can also use kms encryption with an iam role if you want to restrict access to certain users.

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


# AWS KMS with assume-role / multiple aws accounts / multiple roles

At work, we have teams using different iam roles (developers, data-scientists...). 
Iam users are created in AWS Organization, users use assume-role feature and profile to switch from one environment to another.

Different teams have access to the same credentials file, so we do not specify an iam role when using sops.

All teams have access to crendentials in our lab environment.
Only Circleci and administrators have access to other environments (access managed by iam policies). There is a little trick to use sops with our needs.

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

So when your aws credentials file contains multiple profiles you need to export two environments variables before sops command.

```
AWS_SDK_LOAD_CONFIG=1 AWS_DEFAULT_PROFILE=sandbox sops -e -i settings/lab/connections.sh
#AWS_PROFILE : forces sops to use different profile from default
#AWS_SDK_LOAD_CONFIG=1 : AWS_PROFILE is only read if AWS_SDK_LOAD_CONFIG is enabled
```


# Conclusion

Sops is a nice find and is already integrated in our stack. I have only shown you how to use it with AWS, but you can do so much with.  Read their documentation if you are interested : [https://github.com/mozilla/sops](https://github.com/mozilla/sops)
