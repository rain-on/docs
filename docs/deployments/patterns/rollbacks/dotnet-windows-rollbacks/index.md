---
title: Rollback .NET Application on Windows Server
description: A guide on how to rollback a .NET application hosted on Windows Servers.
position: 5
hideInThisSectionHeader: true
---

This guide will walk through rolling back .NET Windows Services and .NET Web Applications hosted on IIS.  It will use the [OctoFX Sample Application](https://github.com/OctopusSamples/OctoFX).  That application has three components:

- Database
- Windows Service
- Website

Rolling back a database is out of the scope of this guide.  As stated in this [article](https://octopus.com/blog/database-rollbacks-pitfalls), rolling back a database schema change could result in wrong or deleted data.  This guide focuses on scenarios where there were no database changes or the database changes are backward compatible.  Because the database changes are out of scope for rollbacks, the database packages will be "skipped" during the rollback process.

## Existing Deployment Process

For this guide, we will start with the following deployment process for the OctoFX application:

1. Run Database Creation Runbook
1. Deploy the OctoFX database
1. Deploy the OctoFX Windows Service
1. Deploy the OctoFX website
1. Verify the application
1. Notify stakeholders

![original windows deployment process](images/original-windows-deployment-process.png)

:::success
View the deployment process on our [samples instance](https://samples.octopus.app/app#/Spaces-762/projects/01-octofx-original/deployments/process).  Please login as a guest.
:::

## Zero Configuration Rollback
!include <zero-configuration-rollback>

## Simple Rollback Process

The typical rollback strategy is to skip specific steps and run additional ones during a rollback.  In this example, the database steps will be skipped with another step to [prevent that release from progressing](/docs/releases/prevent-release-progression.md) will run during a rollback.

The updated deployment process will be:

1. Calculate Deployment Mode
1. Run Database Creation Runbook (skip during rollback)
1. Deploy the OctoFX Database (skip during rollback)
1. Deploy the OctoFX Windows Service
1. Deploy the OctoFX Website
1. Block Release Progression (only run during rollback)
1. Verify the Application
1. Notify stakeholders

![simple rollback for windows deployment](images/windows-simple-rollback-process.png)

:::success
View the deployment process on our [samples instance](https://samples.octopus.app/app#/Spaces-762/projects/02-octofx-simple-rollback/deployments/process).  Please login as a guest.
:::

### Calculate Deployment Mode

!include <calculate-deployment-mode>

### Skip Database Deployment Steps

The two steps related to database deployments, Run Database Creation Runbook and Deploy OctoFX Database, should be skipped during a rollback.  Unlike code, databases cannot easily be rolled back without risking data loss.  For most rollbacks, you won't have database changes.  However, a rollback could accidentally be triggered with a database change.  For example, rolling back a change in **Test** to unblock the QA team.   Skipping these steps during the rollback reduces the chance of accidental data loss.  

To skip these steps during a rollback, set the variable run condition to be:

```text
#{Octopus.Action[Calculate Deployment Mode].Output.RunOnDeploy}
```

We also recommend adding or updating the notes field to indicate it will only run on deployments.

![windows updating notes field](images/windows-updating-notes-field.png)

### Prevent Release Progression

!include <prevent-release-progression>

## Complex Rollback Process

As mentioned earlier, re-deploying the website and windows service involves re-extracting the package, running configuration transforms, and any embedded scripts.  Generally, those steps will finish within 60 seconds.  However, re-deploying those packages carries a small amount of risk because variable snapshots can be updated.  Or, the embedded scripts are complex and take time to finish.  

By default, Octopus Deploy will keep all releases on your Windows Server (this can be changed via [retention policies](https://octopus.com/docs/administration/retention-policies)), which means the previously extracted and configured Windows Service or Website already exists.  Back in Octopus 3.x we added the system variable `Octopus.Action.Package.SkipIfAlreadyInstalled`.  When that variable is set to `True`, Octopus Deploy will:

1. Check the `deploymentjournal.xml` to see if the package has already been installed.
2. If it hasn't been installed, then it will proceed with the deployment.
3. If it is installed, it will skip the deployment but still set the output variable `Octopus.Action[STEP NAME].Output.Package.InstallationDirectoryPath`.

The rollback process in this section will use that functionality to update IIS and the Windows Service registration to point to those older (pre-existing) folders.

The resulting process will be:

1. Calculate Deployment Mode
1. Run Database Creation Runbook (skip during rollback)
1. Deploy the OctoFX Database (skip during rollback)
1. Deploy the OctoFX Windows Service (with `Octopus.Action.Package.SkipIfAlreadyInstalled` set to `True`)
1. Deploy the OctoFX Website  (with `Octopus.Action.Package.SkipIfAlreadyInstalled` set to `True`)
1. Update Windows Service Binary Path
1. Restart Windows Service
1. IIS Update Physical Path
1. Block Release Progression (only run during a rollback)
1. Verify the Application
1. Notify stakeholders

![windows complex rollbacks](images/windows-complex-rollbacks.png)

:::success
View that deployment process on [samples instance](https://samples.octopus.app/app#/Spaces-762/projects/03-octofx-complex-rollback/deployments/process).  Please login as a guest.
:::

### Comparison to Simple Rollback Process

The complex rollback process and simple rollback process have some overlap.  Please refer to the earlier section on how to configure these steps.

1. Add Calculate Deployment Mode step
1. Update Run Database Creation Runbook to skip during rollback
1. Update Deploy OctoFX database to skip during rollback
1. Add Block Release Progression step

The primary difference between the simple and complex rollback process is the complex rollback process reuses the pre-existing extracted application.

### Add System Variable to Skip Package Deployment

Adding the system variable `Octopus.Action.Package.SkipIfAlreadyInstalled` will skip already installed packages.  That makes a lot of sense for rollbacks but less sense for regular deployments.  To _only_ skip package installation for rollbacks, set the variable value to be:

```text
#{if Octopus.Action[Calculate Deployment Mode].Output.DeploymentMode == "Deploy"}False#{else}True#{/if}
```

![windows skip if already installed](images/windows-skip-if-already-installed.png)

### Windows Service Rollback

Updating the existing Windows Service to point to an earlier version of the application involves two steps.

1. [Update Windows Service Binary Path](https://library.octopus.com/step-templates/b6860fcf-9dee-48a0-afac-85e2098df692/actiontemplate-windows-service-change-binary-path)
1. [Restart Windows Service](https://library.octopus.com/step-templates/d1df734a-c0da-4022-9e70-8e1931b083da/actiontemplate-windows-service-restart)

The binary path must include the application's .exe file.  For example, `#{Octopus.Action[STEP NAME].Output.Package.InstallationDirectoryPath}\YOUREXEFILE.exe`. For this guide, that value will be:

```text
#{Octopus.Action[Deploy OctoFX Windows Service].Output.Package.InstallationDirectoryPath}\OctoFX.RateService.exe
```

Set the run condition for this step to:

```text
#{Octopus.Action[Calculate Deployment Mode].Output.RunOnRollback}
```

### Website Rollback

In modern versions of IIS, updating the physical path is an instantaneous action.  All traffic is routed to that new path.  To do that, use the [IIS Website - Update Property](https://library.octopus.com/step-templates/34118a0e-f872-435a-8522-d3c7f8515cb8/actiontemplate-iis-website-update-property) step template. 

The parameters to set for this step template are:
- Web site name: The name of your website
- Name of property to set: `physicalPath`
- Value of property to set: `#{Octopus.Action[Deploy OctoFX Website].Output.Package.InstallationDirectoryPath}`

Set the run condition for this step to:

```text
#{Octopus.Action[Calculate Deployment Mode].Output.RunOnRollback}
```

:::hint
If you are using application pools instead of websites, use [IIS AppPool - Update Property](https://library.octopus.com/step-templates/183c1676-cb8e-44e8-a348-bbcb2b77536e/actiontemplate-iis-apppool-update-property) step template.
:::

## Simple or Complex Rollback Process

We recommend starting with the simple rollback process first.  That requires the least amount of changes while at the same time gives you the rollback functionality.  Only move to the complex rollback process if you determine the simple rollback process isn't meeting a specific need.