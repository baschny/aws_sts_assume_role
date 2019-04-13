## aws_sts_assume_role
This role allows for switching of AWS role-credentials mid-playbook.
It's like _sudo_ for Ansible automation of AWS resources.

The intended use (my itch) is when running playbooks from a normal user account which requires assuming other AWS roles to get things done.
By defining the assumed roles within playbooks, we do not need to rely on the command line or shell environment config to set the correct AWS credentials.

A side effect of this role is that we can be more granular with AWS permissions. 
So we can switch to an AWS-role, run some Ansible tasks, switch to another AWS-role, and then run some other tasks. 
This is useful where some infrastructure or operational tasks require elements within different AWS accounts, or with different permissions scope.

_Note 1. This is the Ansible role. 
In an unfortunate naming clash AWS also uses roles, and we use this Ansible role to switch those AWS roles. I try to be explicit and verbose about the difference._

_Note 2. To maintain ease of understanding it helps to use this role as per the examples below, in top-level playbooks. Try to avoid embedding this role in the tasks file of another role where it will be hidden._

### Credential Caching
This role caches credentials to the filesystem.
The cache path and the timeout that the role observes are set in the default vars.

```
(ansible) camus: ls -l /Users/thisdougb/.aws/
-rw-------  1 thisdougb  staff   755 10 Apr 21:13 ansible_aws_assume_role.533274721903.dev-role.cache
-rw-------  1 thisdougb  staff  1128 10 Mar 18:23 config
-rw-------  1 thisdougb  staff   238 27 Mar 10:48 credentials
```
### Requirements
```
$ aws sts get-caller-identity
{
    "UserId": "K3DEIJ7UQI5OKTHVEQODE",
    "Account": "533274721903",
    "Arn": "arn:aws:iam::533274721903:user/thisdougb"
}
```
### Role Variables
The role accepts following variables:

| Variable           | Type   | Description                              | Required | Default value                 |
|--------------------|--------|------------------------------------------|----------|-------------------------------|
| aws_account_id     | string | ID of the targeted acccount              | yes      | none                          |
| switch_to_role     | string | name of role that we would like to swich | yes      | none                          |
| aws_region         | string | AWS region to use                        | yes      | none                          |
| aws_username       | string | name of user for MFA authentication      | no       | current AWS caller username   |
| aws_mfa_account_id | string | ID of the account for MFA authentication | no       | current AWS caller account ID |
| without_mfa        | bool   | If defined does not ask for MFA token    | no       | undefined                     |
| caching_enabled    | bool   | Enables caching of assumed credentials   | yes      | no                            |


### Exported Variables
This Ansible role stores the temporary role credentials as facts.
These variables must be used on the tasks you want to run as that role.
```yaml
- name: "Create IAM User"
  iam_user:
    name: "thisdougb"
    state: present
    aws_access_key: "{{ sts_aws_access_key }}"
    aws_secret_key: "{{ sts_aws_secret_key }}"
    security_token: "{{ sts_security_token }}"
```
The role is designed to use variables in this way specifically so we can assume different roles within a playbook.
If the role simply overwrite the standard AWS env vars, we would not be able to 'un-assume' a role and switch to a new role.

### Example Playbook
Some quite contrived examples.
I tend to work in multi-account AWS environments, but running in a single account doesn't change how the role works.

Example usage for `aws_sts_assume_role`:

```yaml
- hosts: localhost
  roles:
    - {
        role: aws_sts_assume_role,
        vars: { 
            aws_sts_assume_role_aws_account_id: "533274721903", 
            aws_sts_assume_role_switch_to_role: "admin-instances-role" 
        }
      }
    - fire_up_load_test_env
```
Example usage in automated environment, without prompting for an MFA token:
```yaml
- hosts: localhost
  roles:
    - { 
        role: aws_sts_assume_role, 
        vars: {
            aws_sts_assume_role_aws_account_id: "533274721903", 
            aws_sts_assume_role_switch_to_role: "rds-admin-role", 
            without_mfa: yes 
        }
    }
    - rotate_database_service_credentials
    
    # now we switch role for another task
    - { 
        role: aws_sts_assume_role, 
        vars: { 
            aws_sts_assume_role_aws_account_id: "533274721903", 
            aws_sts_assume_role_switch_to_role: "dev-instances-role", 
            without_mfa: yes
        }
    }
    - terminate_all_unused_instances_at_night
```

### License

MIT

### Authors Information

- Doug Bridgens (@thisdougb)
