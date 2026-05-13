---
title: "Automatically deleting blobs in Azure Blob Storage"
date: 2026-05-13T10:00:00+02:00
draft: true
toc: false
images:
tags:
  - azure
  - blob storage
  - lifecycle management
  - csharp
  - bicep
---

Recently I worked on a task where I needed to remove uploaded blobs after a certain amount of time. My mind immediately went to `Azure Blob storage lifecycle management`, but the problem appeared to be a bit more nuanced than I first thought.

With lifecycle management you can define rules that automatically perform actions on blobs. Each rule consists of conditions, actions, and filters.

- `Conditions`: Specify if the rule applies based on when a blob was created, last modified, or last accessed.
- `Actions`: Specify which action to apply. This can be changing the blob's tier or deleting it.
- `Filters`: Limits the rule to a subset of blobs. By default it applies to all blobs, but filters allow for filtering for certain tags or prefixes.

A common use case is to automatically clean up temporary data. You'd create a rule that filters on a prefix like `tmp/`, set a condition to match blobs that haven't been modified in 30 days, and set the action to delete. You can also combine actions in a single rule, for example first transitioning a blob from hot to cool after 7 days and then deleting it after 90 days. As long as your blobs are organized by prefix or tagged consistently, this works great.

But prefix filters don't support wildcards. They only match from the start of the blob name, so there's no way to target blobs by extension or by a pattern further down the path. You can only filter on up to 10 tags per rule. These limits didn't seem like a problem at first, until we looked at our actual setup.

## Our setup

When designing how we stored data in Blob Storage we opted for a tenant separated approach. This means that each tenant got its own blob container with all the data that belongs to that tenant. Some files needed to be kept always, while others should only live temporarily. Temporary files are mostly files that needed to be processed once and could be cleaned up afterwards. Originally we had the code delete the file directly, but there were some cases in which we wanted to reprocess a file shortly after, which was no longer possible if the file was already deleted.

```
storage-account
|-- tenant_a/
|   |-- file_to_keep.json
|   |-- file_to_delete.pdf
|-- tenant_b/
|   |-- file_to_keep.json
|   |-- file_to_delete.png
```

This setup shows that the prefix filter isn't an option as the tenants have unique names and would require a separate rule per tenant. With thousands of tenants this is impossible to set up.

This left us with the tag filter as the only option. We had to decide how long certain files needed to be kept and if we needed any variety in there. In the end we settled for automatic deletion after seven days, but also shortly entertained a version that was flexible and would support any timespan given without the need of setting up lifecycle rules at all.

## Using one lifecycle rule

We settled on seven days for automatic deletion, but still needed a way to determine which blobs should fall under that rule and which don't. When uploading those blobs we determine if it needs to be automatically removed and add a tag to it. The lifecycle rule would then filter on that tag and cleanup the blob when needed. When a blob has no tag it's ignored and stays in the system permanently.

How you add a tag in C# depends on how you upload the blob. When you use the `Upload` or `UploadAsync` methods you can pass in a `new BlobUploadOptions()` object. If you use `OpenWrite` or `OpenWriteAsync` you need to pass in a `new BlobOpenWriteOptions()` object. Both objects have a `Tags` property which is a `Dictionary<string,string>`.

```csharp
var options = new BlobUploadOptions
{
	Tags = new Dictionary<string, string>
	{
		{ "auto-remove", true.ToString() }
	}
};

await blobClient.UploadAsync(blobStream, options);
```

For the second part, the lifecycle rule, we used Bicep to create it. The rule filters for blobs with the `auto-remove` tag set to true and deletes them when the blob is created more than 7 days ago.

```bicep
resource lifecyclePolicy 'Microsoft.Storage/storageAccounts/managementPolicies@2026-04-01' = {
  name: 'default'
  parent: storageAccount
  properties: {
    policy: {
      rules: [
        {
          name: 'auto-remove-7-days'
          type: 'Lifecycle'
          definition: {
            actions: {
              baseBlob: {
                delete: {
                  daysAfterCreationGreaterThan: 7
                }
              }
            }
            filters: {
              blobTypes: [
                'blockBlob'
              ]
              blobIndexMatch: [
                {
                  name: 'auto-remove'
                  op: '=='
                  value: 'true'
                }
              ]
            }
          }
        }
      ]
    }
  }
}
```

When your setup is simple and only few different retention periods are needed this works great. If you would have a second case where for some reason you want to delete some files after three days as well you can add another tag and rule, like `auto-remove-3-days`, with its own lifecycle rule that deletes after three days.

## A code based alternative

Adding rules works if the total amount of retention periods stays low. Each additional period means a new tag and lifecycle rule you need to keep track of and in code you need to know which one to use and where. When in need of true flexibility you can skip the lifecycle rule and handle similar logic in code.

With this approach a tag is added called `expiresOn` which has the exact moment the blob should expire. For easy comparison we used the `Ticks` property to get a numerical comparison which avoids timezone and formatting issues. Something to note here is that tags are strings, so technically it's a lexicographic comparison, but as ticks size always are the same length this worked for our case.

```csharp
var expiresOn = DateTime.UtcNow.AddDays(7);
var options = new BlobUploadOptions
{
	Tags = new Dictionary<string, string>
	{
		{ "expiresOn", expiresOn.Ticks.ToString() }
	}
};

await blobClient.UploadAsync(blobStream, options);
```

With this approach the number of days isn't fixed. The logic in your code can pass any `DateTime` object and it doesn't matter if it's in 7 days or 7 years, it all works the same.

To delete the blobs we added a `TimerTrigger` inside an Azure Functions App. It runs once per day, similar to how lifecycle rules are only triggered once per day, but if you want you can easily adjust it to run multiple times a day if that suits your needs better.

```csharp
[Function(nameof(CleanupExpiredDocumentsTrigger))]
public async Task CleanupExpiredDocumentsTrigger(
  [TimerTrigger("0 0 2 * * *", RunOnStartup = false)] TimerInfo myTimer)
{
  var blobServiceClient = ...; // Injected via constructor
  var expiredBlobs = blobServiceClient.FindBlobsByTagsAsync($"expiresOn < '{DateTime.UtcNow.Ticks}'");

  await foreach (var expiredBlob in expiredBlobs)
  {
    var blobClient = blobServiceClient
      .GetBlobContainerClient(expiredBlob.BlobContainerName)
      .GetBlobClient(expiredBlob.BlobName);

    await blobClient.DeleteIfExistsAsync();
  }
}
```

`FindBlobsByTagsAsync` searches across all containers in a storage account and returns the ones where the query matches on the tag. If the tag does not exist it's ignored. The approach gives you more flexibility, but makes you responsible for cleanup and monitoring around it yourself. If the function starts failing blobs remain in the system, where with lifecycle management this is all handled by Azure.

## References

- https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview
