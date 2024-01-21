---
title: "Vault policy mistake can lead to escalation of privilege"
excerpt: "This is an example of how a misconfigured policy editor can cause escalation of privilege within Vault"
tags: 
  - vault
---

# Vault policy mistake can lead to escalation of privilege

This is a simple talk about how to ensure your admin's cannot escalate their access within HashiCorp Vault. 

## The Problem

Consider the below policy. 

bad-policy.hcl
```hcl
path "sys/policies/acl/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

It's a fairly common full admin policy for a policy admin. There is however a problem with creating a policy-admin policy like this, and that is that this allows the policy admin to edit their own policy. The culprit here is the "update" capability, because we allow full access to any policy.

A login with this policy get's permission denied on running `vault secrets list`.

For instance, I can run this command `vault policy write policy ./full-access.hcl` with the following content:

full-access.hcl
```hcl
path "*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
```

And suddenly by just logging out and in again I can gain full access to almost anything in Vault. 
`vault secrets list` gives the following output when running vault in dev mode:
```
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_44388b20    per-token private secret storage
identity/     identity     identity_7537a574     identity store
secret/       kv           kv_77bcf5a3           key/value secret storage
sys/          system       system_921f5983       system endpoints used for control, policy and debugging
```

## Solution

Luckily there is quite a simple fix for this. For every similar admin policy, I need to add a deny to that specific path to ensure that they cannot escalate their privilege. This works for any policy-admin or authentication admins. 

good-policy.hcl
```hcl
path "sys/policies/acl/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "sys/policies/acl/policy-admin" {
  capabilities = ["deny"]
}
```

This last policy will allow the policy admin full access to any policy except the policy named 'policy-admin'. Edit it to fit your needs. This is the output from trying to edit the policy-admin policy with the new deny:

```
Error uploading policy: Error making API request.

URL: PUT http://127.0.0.1:8200/v1/sys/policies/acl/policy
Code: 403. Errors:

* 1 error occurred:
        * permission denied
```