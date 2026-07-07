---
title: "Using Azure Policy to prevent unscoped blob lifecycle rules"
date: 2026-07-07T10:00:00+02:00
draft: false
toc: false
images:
tags:
  - azure
  - blob storage
  - lifecycle management
  - azure policy
---

Azure Blob Storage lifecycle management is a nice way to move blobs between tiers or delete them after a certain amount of time. You configure the rule once and Azure takes care of the rest. The part that can be easy to miss is that a rule without filters applies to everything in the storage account.

That can be fine when you really do want a global rule, but most of the time it is risky. A missing `prefixMatch` or `blobIndexMatch` can suddenly transition or delete far more blobs than intended. In production you probably have reviews and deployment checks in place, but in lower environments it is still easy for someone to push a bad change and create a mess that takes time to clean up.

I ran into this while working with lifecycle rules for automatic cleanup. In another post I wrote about [Automatically deleting blobs in Azure Blob Storage](/posts/automatic-delete-blob-azure/), where I explain how lifecycle rules can be useful for targeted cleanup. This post is about the other side of that story: how to prevent overly broad rules from being created in the first place.

## Why block unfiltered lifecycle rules

The problem is not lifecycle management itself. The problem is that the safest option is not the default one. If you forget to add a filter, Azure will happily accept a rule that targets every blob in the account.

Using infrastructure as code already reduces that risk a lot. A Bicep change in a pull request is much easier to review than a manual change in the portal. Still, that only helps if every change goes through the same path. In shared dev or test subscriptions people sometimes need to move fast, and that is usually where these kinds of mistakes slip in.

What I wanted was a guardrail that simply refuses lifecycle rules when no filters are defined.

## Creating the Azure Policy definition

Azure Policy is a good fit for this. The policy definition describes which resources it applies to and what condition should be checked. After that you assign the policy at management group, subscription, resource group, or individual resource scope.

In the example below the policy targets `Microsoft.Storage/storageAccounts/managementPolicies`. It counts rules where both `prefixMatch` and `blobIndexMatch` are missing. If at least one such rule exists, the policy denies the change.

Finding the right field aliases is usually the part that takes me the most time when writing a policy. What I often do is create the resource once and inspect the generated ARM or Bicep. Another useful option is this CLI command:

```bash
az provider show --namespace Microsoft.Storage --expand "resourceTypes/aliases" --query "resourceTypes[].aliases[].name"
```

That gives you the available policy aliases for the provider. These days Copilot is also pretty good at finding the right field names quickly when you describe the scenario clearly.

```bicep
targetScope = 'subscription'

@description('Name of the custom policy definition.')
param policyDefinitionName string = 'require-blob-lifecycle-filter-scope'

@description('Display name for the custom policy definition.')
param policyDisplayName string = 'Require prefixMatch or blobIndexMatch in Blob lifecycle rules'

@description('Description for the custom policy definition.')
param policyDescription string = 'Prevents lifecycle rules without prefixMatch or blobIndexMatch to avoid accidental deletes across the entire storage account.'

resource lifecycleFilterPolicy 'Microsoft.Authorization/policyDefinitions@2026-06-01' = {
  name: policyDefinitionName
  properties: {
    policyType: 'Custom'
    mode: 'All'
    displayName: policyDisplayName
    description: policyDescription
    metadata: {
      category: 'Storage'
      version: '1.0.0'
    }
    parameters: {}
    policyRule: {
      if: {
        allOf: [
          {
            field: 'type'
            equals: 'Microsoft.Storage/storageAccounts/managementPolicies'
          }
          {
            count: {
              field: 'Microsoft.Storage/storageAccounts/managementPolicies/policy.rules[*]'
              where: {
                allOf: [
                  {
                    field: 'Microsoft.Storage/storageAccounts/managementPolicies/policy.rules[*].definition.filters.prefixMatch[*]'
                    exists: 'false'
                  }
                  {
                    field: 'Microsoft.Storage/storageAccounts/managementPolicies/policy.rules[*].definition.filters.blobIndexMatch[*]'
                    exists: 'false'
                  }
                ]
              }
            }
            greater: 0
          }
        ]
      }
      then: {
        effect: 'deny'
      }
    }
  }
}
```

## Assigning the policy

Once the definition exists you still need to assign it. In the Bicep example below the assignment is created at subscription scope. The `enforcementMode` can be `Default` or `DoNotEnforce`.

`Default` means the policy is fully active. For this policy that means Azure rejects any lifecycle rule update that does not define either `prefixMatch` or `blobIndexMatch`.

`DoNotEnforce` is useful when you want to roll out the assignment more carefully. Azure still evaluates the resources against the policy and marks violations as non-compliant, but it does not block the change. For this specific case that means a filterless lifecycle rule can still be created or updated, while the policy compliance results show that it would have been denied once enforcement is turned on.

I also added `excludedStorageAccounts` as an escape hatch. If you really do have a valid scenario where a storage account needs a lifecycle rule that applies to everything, you can exclude that specific account from the assignment instead of weakening the rule for everyone.

```bicep
targetScope = 'subscription'

@description('Name of the policy assignment at subscription scope.')
param policyAssignmentName string = 'assign-require-blob-lifecycle-filter-scope'

@description('Display name for the policy assignment.')
param policyAssignmentDisplayName string = 'Assign: Require prefixMatch or blobIndexMatch in Blob lifecycle rules'

@description('Description for the policy assignment.')
param policyAssignmentDescription string = 'Ensures each Blob lifecycle rule includes prefixMatch or blobIndexMatch.'

@description('Name of an existing policy definition at the current subscription scope.')
param policyDefinitionName string = 'require-blob-lifecycle-filter-scope'

type excludedStorageAccount = {
  name: string
  resourceGroupName: string
}

@description('Optional storage account exceptions by name and resource group.')
param excludedStorageAccounts excludedStorageAccount[] = []

@description('Assignment enforcement mode. Use DoNotEnforce for audit-only rollout.')
@allowed([
  'Default'
  'DoNotEnforce'
])
param enforcementMode string = 'Default'

var excludedStorageAccountScopes = [
  for storageAccount in excludedStorageAccounts: resourceId(
    storageAccount.resourceGroupName,
    'Microsoft.Storage/storageAccounts',
    storageAccount.name
  )
]

resource policyDefinition 'Microsoft.Authorization/policyDefinitions@2026-06-01' existing = {
  name: policyDefinitionName
}

resource policyAssignment 'Microsoft.Authorization/policyAssignments@2026-06-01' = {
  name: policyAssignmentName
  properties: {
    displayName: policyAssignmentDisplayName
    description: policyAssignmentDescription
    policyDefinitionId: policyDefinition.id
    notScopes: excludedStorageAccountScopes
    enforcementMode: enforcementMode
  }
}
```

## The result

With this in place you get a guardrail at the platform level. Even if someone updates a lifecycle policy directly, Azure blocks the change when the rule has no `prefixMatch` or `blobIndexMatch`. I like this kind of protection for shared environments, because the rule is enforced by the platform instead of relying on everyone to remember it.

When someone tries to add a filterless lifecycle rule they will get a `Failed to update lifecycle management policy` error. In the details Azure Policy shows which assignment was violated, in this case `Assign: Require prefixMatch or blobIndexMatch in Blob lifecycle rules`.

If you are already using lifecycle management for cleanup, be sure to also check out my post about [Automatically deleting blobs in Azure Blob Storage](/posts/automatic-delete-blob-azure/). That one focuses on how to use lifecycle rules effectively. This post is mainly about making that setup safer.

## References

If you want to read more about the pieces used here, these are good starting points:

- https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview
- https://learn.microsoft.com/en-us/azure/governance/policy/overview
- https://learn.microsoft.com/en-us/azure/governance/policy/concepts/definition-structure-basics
