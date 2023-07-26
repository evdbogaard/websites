---
title: "Branch policies in Azure Repos"
date: 2021-04-27T10:00:00+02:00
draft: false
toc: false
images:
tags:
  - azure
  - branch policies
---

Hi, I've been wanting to start writing some blog posts for a while, but always find excuses why not to do it. Now I had a small thing for work I had to figure out and though why not give it a go. Hope you find it interesting :)

# What are Azure Repos?

When you have an Azure DevOps account you get access to Azure Repos. It is a collection of version control tools that help with code management. There are two types of version control offered within Azure Repos: Git and Team Foundation Version Control (TFVC).

Personally I have only experience with Git repos so I'll be focusing on them in this post. With an Azure DevOps account you can create both private and public git repositories for free.

# What are branch policies?

Branch policies help you protect important branches and enforce code quality. When a branch has a policy set to it, it is no longer possible to commit directly to it. All changes need to be done through a Pull Request (PR).
There are different kind of policies that can be set up:
- Require a minimum number of reviewers
- Check for linked work items
- Check for comment resolution
- Limit merge types
- Build validation
- Status checks
- Automatically include code reviewers

I won't go into depth about what each policy does. Most names are self explanatory and in case you want to read up more on them I suggest reading [this article](https://docs.microsoft.com/en-us/azure/devops/repos/git/branch-policies?view=azure-devops).

## Setting policies through Azure DevOps

The easiest way to set branch policies is through the DevOps website. There are two ways to set them. Create a branch policy for all repositories at once, or set one per repository.

### All repositories branch policy
Go to `Project Settings` -> `Repositories` -> `Policies`
At the bottom of the page you see all project-wide branch policies currently active. By clicking on the Plus icon you can select which branch to protect and enable the policies you need.

![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/be9cmszcwco3xztz4ze4.png)

When activating a specific policy the page shows all options you can set for that policy. Some have more options than others. Changes are immediately saved and don't require a confirmation.

![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/96frjknnmugxd3psn315.png)

### Specific repository branch policy
Go to `Project Settings` -> `Repositories` -> Select the repository -> `Policies` -> Select the branch
You now see a page that shows the policies set for this specific branch/repo combination. Setting or changing a policy works the same as previously described. If there is a project-wide branch policy set you can also see it here, but not alter it. You will see a small information message after the policy itself.
![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/w6hkco5zv5j366e0jak9.png)
 
## Automate setting policies with Azure CLI
As with most things Azure there are multiple ways to do it. If you have multiple repositories or branches you want to set the same policies on, but cannot use the project-wide branch policies then automation is your friend.
I had a problem similar to this and want to show how I solved it.

What I tried to do was adding policies to the `main` branch for multiple repositories, however some repositories didn't needed these rules. I also wanted to make sure that running the script multiple times would not result in errors or duplicate policies.

Here is the code I came up with (PowerShell):
```
$orgUrl = "URL_OF_ORGANIZATION" # https://dev.azure.com/OrgName
$project = "NAME_OF_PROJECT"

$repositories = @("Repository", "Names", "To", "Set", "Policies", "For")
$branch = "NAME_OF_BRANCH"
$approversId = "ID_OF_GROUP_OR_USER"

foreach($repo in $repositories) {
    $repoId = (az repos show --org $orgUrl -p $project --repository $repo --query id -o tsv)
    echo "$repo has id: $repoId"

    $currentPolicies = (az repos policy list --org $orgUrl -p $project --repository-id $repoId --branch $branch --query [].type.displayName -o tsv)

    if ($currentPolicies -eq $null -Or !$currentPolicies.Contains("Minimum number of reviewers")) {
        echo "Creating minimum number of reviewers policy for $repo"
        az repos policy approver-count create --org $orgUrl -p $project --branch $branch --repository-id $repoId --allow-downvotes false --blocking true --creator-vote-counts true --enabled true --minimum-approver-count 1 --reset-on-source-push false -o none
    } else {
        echo "$repo already has minimum number of reviewers policy"
    }

    if ($currentPolicies -eq $null -Or !$currentPolicies.Contains("Comment requirements")) {
        echo "Creating comment requirements policy for $repo"
        az repos policy comment-required create --org $orgUrl -p $project --branch $branch --repository-id $repoId --blocking true --enabled true -o none
    } else {
        echo "$repo already has comment requirements policy"
    }

    if ($currentPolicies -eq $null -Or !$currentPolicies.Contains("Work item linking")) {
        echo "Creating work item linking policy for $repo"
        az repos policy work-item-linking create --org $orgUrl -p $project --branch $branch --repository-id $repoId --blocking true --enabled true -o none
    } else {
        echo "$repo already has work item linking policy"
    }

    if ($currentPolicies -eq $null -Or !$currentPolicies.Contains("Required reviewers")) {
        echo "Creating required reviewers policy for $repo"
        az repos policy required-reviewer create --org $orgUrl -p $project --branch $branch --repository-id $repoId --blocking true --enabled true --message "PR Approvers" --required-reviewer-ids $approversId -o none
    } else {
        echo "$repo already has required reviewers policy"
    }
}
```
Let's look at it step by step. First of all `az repos` commands need an organization url and a project name. These can be set as default with the `az devops configure` command, but if they are not set they need to be passed with each command.

The approverIds is the ID of a group I created in Azure DevOps. The ID I obtained with this command: `az devops team list --org $orgUrl -p $project --query '[].{name:name, id:id}'

After that I create an array with repo names and set the branch I want to operate on. We then loop through each repository and get the ID and already active policies for that repo/branch combination.

Each policy has an unique id or displayname. I grabbed the displaynames here to make it more understandable. For each policy we want to set we check if it's displayname is already in the active policies list and skip if so.

### Further improvements
The current code is really basic, but does it's job. We can improve it further by limiting the number of calls it needs to make. Especially with lots of repositories it can take a while to run.
I'm also not a fan of the `Contains` check, if someone knows a more efficient way to check please share it.
Finally there is an issue that if a policy is already set we simply ignore it, even if it has a different setup. Besides the `create` command there is also `update`. We probably need to do some kind of checking here to make sure all policies are updated correctly.

# Further reading
If this sounds interesting for you make sure to check out the following links for extra information.
- https://docs.microsoft.com/en-us/azure/devops/repos/get-started/what-is-repos?view=azure-devops
- https://docs.microsoft.com/en-us/azure/devops/repos/git/branch-policies-overview?view=azure-devops
- https://docs.microsoft.com/en-us/cli/azure/repos/policy?view=azure-cli-latest