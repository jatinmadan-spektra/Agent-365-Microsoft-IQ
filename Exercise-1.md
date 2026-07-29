# Exercise 1: Entra Agent ID - Identity for Agents

### Estimated Duration: 60 minutes

Before an AI agent can be governed, it has to exist as a first-class identity in your directory. In this exercise you give Contoso's expense-processing agent, **Finley**, an enterprise identity in Microsoft Entra, establish who is accountable for it, and confirm it is discoverable in the Agent 365 registry.

## Overview

Human employees get an identity before they get a laptop or a mailbox. AI agents work the same way. **Microsoft Entra Agent ID** provides purpose-built identity constructs for agents so you can authenticate, authorize, govern, and protect them at enterprise scale.

Two constructs matter in this exercise:

- An **agent identity blueprint** is the reusable template that defines a class of agent. It declares which permissions agents of this kind may request and who is accountable for the class. If Contoso later wants ten expense agents, all ten come from this one blueprint.
- An **agent identity** is a specific agent created from that blueprint. This is Finley. It is the identity that actually requests access tokens and performs work.

You will also set an **owner** and a **sponsor**. A sponsor is the person accountable for why the agent exists, similar to a hiring manager, and is required. An owner is the person who can change the agent's configuration, and is recommended.

>**Note:** You will create the blueprint using the **Agent 365 CLI**, not the Microsoft Entra admin center wizard. This is deliberate and important. Blueprints must have the `managerApplications` property set to be accepted by the Agent 365 platform, and only the CLI sets this automatically. A blueprint created through the portal wizard will not appear in the Intune agent picker in Exercise 3.

In this exercise you will:

- Install and verify the Agent 365 CLI
- Provision the agent blueprint, Azure infrastructure, and agent identity in a single command
- Verify the blueprint and its granted API permissions in Microsoft Entra
- Assign an owner and a sponsor to establish accountability
- Confirm the agent appears in the Agent 365 registry

## Objectives

- **Task 1**: Install and verify the Agent 365 CLI
- **Task 2**: Provision the agent blueprint using the Agent 365 CLI
- **Task 3**: Verify the blueprint and permissions in Microsoft Entra
- **Task 4**: Assign owners and sponsors to establish accountability
- **Task 5**: Confirm the agent in the Agent 365 registry

### Task 1: Install and verify the Agent 365 CLI

In this task you prepare your tooling. The Agent 365 CLI is a .NET global tool, so you first confirm .NET is present, then install the CLI and authenticate to Azure.

1. On the lab virtual machine, select the **Start** button, type **Windows PowerShell**, and select **Windows PowerShell**.

    ![Windows Start menu with Windows PowerShell in the search results](./media/a365-ex1-t1-01.png)

1. Confirm that .NET 8.0 or later is installed by running the following command:

    ```
    dotnet --version
    ```

    >**Note:** The Agent 365 CLI requires .NET 8.0 or later. The lab virtual machine has this pre-installed. If the command returns a version lower than 8.0, contact lab support.

1. Install the Agent 365 CLI globally by running the following command:

    ```
    dotnet tool install --global Microsoft.Agents.A365.DevTools.Cli
    ```

    ![PowerShell showing the successful installation of the Agent 365 CLI global tool](./media/a365-ex1-t1-02.png)

    >**Note:** If the CLI is already installed, this command reports that the tool is already installed. In that case, run `dotnet tool update --global Microsoft.Agents.A365.DevTools.Cli` instead to ensure you are on the latest version.

1. Verify the installation by displaying the CLI help:

    ```
    a365 -h
    ```

    You should see the list of available `a365` commands. This confirms the CLI is on your path and ready to use.

    ![PowerShell showing the a365 CLI help output with available commands](./media/a365-ex1-t1-03.png)

    >**Note:** If `a365` is not recognized, close and reopen PowerShell. The .NET SDK adds `%USERPROFILE%\.dotnet\tools` to your path when a global tool is first installed, and the new path is only picked up by a new shell session.

1. Authenticate to Azure. The CLI creates Azure resources on your behalf, so it needs an authenticated Azure context.

    ```
    az login
    ```

1. A browser window opens. Sign in with the following credentials:

   - **Email/Username:** <inject key="AzureAdUserEmail"></inject>
   - **Password:** <inject key="AzureAdUserPassword"></inject>

1. Return to PowerShell and confirm the correct subscription is selected:

    ```
    az account show
    ```

    ![PowerShell showing the az account show output with the active subscription](./media/a365-ex1-t1-04.png)

You now have the Agent 365 CLI installed and an authenticated Azure context.

### Task 2: Provision the agent blueprint using the Agent 365 CLI

In this task you run a single command that does a surprising amount of work: it creates the Azure hosting infrastructure, registers the agent identity blueprint in Microsoft Entra, and configures the Microsoft Graph permissions that agents created from the blueprint will inherit.

1. In PowerShell, create and move into a working folder for the agent. The CLI writes its configuration output here, so keep track of this folder.

    ```
    New-Item -ItemType Directory -Path C:\LabFiles\finley -Force
    cd C:\LabFiles\finley
    ```

1. Run the setup command. This creates the blueprint in config-free mode, meaning the CLI resolves your tenant and client app automatically without requiring a configuration file.

    ```
    a365 setup all --agent-name finley-<inject key="DeploymentID" enableCopy="false"/>
    ```

    >**Note:** Do not use spaces or special characters in the agent name. Setup typically takes 3 to 5 minutes.

1. Because you are signed in as a Global Administrator, the CLI opens a browser window requesting **admin consent** for the permissions the blueprint needs. Review the requested permissions and select **Accept**.

    ![Browser showing the admin consent dialog for the agent blueprint permissions](./media/a365-ex1-t2-01.png)

    >**Note:** Do not close this browser window without completing consent. If you do, setup will finish with pending permission grants. You can recover by running `a365 setup all` again, which re-prompts for consent.

1. Wait for the command to complete. The output summarizes each step the CLI performed: creating the resource group, App Service Plan, and Web App with managed identity; registering the agent blueprint; configuring API permissions; and writing the generated configuration file.

    ![PowerShell showing the completed a365 setup all summary output](./media/a365-ex1-t2-02.png)

1. Inspect the generated configuration file to capture the identifiers you will need in later exercises:

    ```
    Get-Content a365.generated.config.json | ConvertFrom-Json
    ```

1. Confirm the following key values are present, and copy them into a text file for reference:

    | Field | What it is | What to check |
    | --- | --- | --- |
    | `agentBlueprintId` | The blueprint's application (client) ID | Should be a valid GUID |
    | `agentBlueprintObjectId` | The blueprint's object ID in Microsoft Entra | Should be a valid GUID |
    | `managedIdentityPrincipalId` | The managed identity used for authentication | Should be a valid GUID |
    | `messagingEndpoint` | Where Teams and Outlook send messages to the agent | Should be an `https://` URL |
    | `resourceConsents` | The APIs consented to | Should list Microsoft Graph, Agent 365 Tools, Messaging Bot API, Observability API |
    | `completed` | Setup status | Should be `true` |

    ![PowerShell showing the parsed contents of a365.generated.config.json](./media/a365-ex1-t2-03.png)

    >**Note:** If `completed` is `false` or `resourceConsents` is empty, the OAuth2 permission grants did not finish. Re-run `a365 setup all --agent-name finley-<inject key="DeploymentID" enableCopy="false"/>` and complete the consent prompt.

1. Verify the Azure resources were created:

    ```
    az resource list --resource-group <your-resource-group> --output table
    ```

    >**Note:** Replace `<your-resource-group>` with the resource group name shown in the setup output. You should see an App Service Plan and a Web App.

Finley now has a registered agent identity blueprint and hosting infrastructure.

### Task 3: Verify the blueprint and permissions in Microsoft Entra

Creating the blueprint through the CLI is fast, but as an administrator you should always confirm what was actually created in the directory. In this task you verify the blueprint object and, importantly, that its permissions were granted rather than left pending.

1. Open a new browser tab and navigate to the Microsoft Entra admin center:

    ```
    https://entra.microsoft.com
    ```

1. In the left navigation pane, expand **Entra ID (1)** and select **App registrations (2)**.

    ![Microsoft Entra admin center with Entra ID expanded and App registrations selected](./media/a365-ex1-t3-01.png)

1. Select the **All applications (1)** tab, then paste your `agentBlueprintId` into the search box **(2)** and press **Enter**.

    ![App registrations page with the All applications tab selected and a blueprint ID in the search box](./media/a365-ex1-t3-02.png)

1. Select your agent blueprint application from the results.

1. In the left navigation pane of the application, select **API permissions**.

1. Review the permissions list and confirm that each permission shows a green check mark and the status **Granted for Contoso**.

    ![API permissions page showing Microsoft Graph permissions with Granted status and green check marks](./media/a365-ex1-t3-03.png)

    >**Note:** If any permission shows **Not granted**, select **Grant admin consent for Contoso** and confirm. Permissions left ungranted here will cause failures in Exercise 4 when the agent tries to call Microsoft 365 services.

1. Now view the same object through the agent-specific experience. In the left navigation pane, expand **Entra ID (1)**, select **Agents (2)**, and then select **Agent blueprints (3)**.

    ![Microsoft Entra admin center showing Entra ID, Agents, and Agent blueprints in the navigation](./media/a365-ex1-t3-04.png)

1. Confirm your blueprint **finley-<inject key="DeploymentID" enableCopy="false"/>** appears in the list.

1. Select **Agent identities** in the navigation pane and confirm that an agent identity derived from your blueprint appears here.

    ![Agent identities page listing the agent identity created from the blueprint](./media/a365-ex1-t3-05.png)

    >**Note:** Every agent identity is a child of exactly one agent identity blueprint. This parent-child relationship is what allows you to apply a single Conditional Access policy to an entire class of agents, which you will do in Exercise 2.

### Task 4: Assign owners and sponsors to establish accountability

An agent with no accountable human is a governance gap. Microsoft 365 admin center surfaces "agents without owners" as a governance action precisely because ownerless agents accumulate risk. In this task you close that gap before the agent does any work.

1. Still in the Microsoft Entra admin center, navigate to **Entra ID** > **Agents** > **Agent blueprints**.

1. Select your blueprint **finley-<inject key="DeploymentID" enableCopy="false"/>**.

1. On the blueprint detail page, locate the **Owners** field and select the pencil icon next to it.

    ![Blueprint detail page with the pencil icon next to the Owners field highlighted](./media/a365-ex1-t4-01.png)

1. Search for and select your lab user account, then select **Save**. The owner is the identity that can make configuration changes to this blueprint.

1. Locate the **Sponsors** field and select the pencil icon next to it.

1. Search for and select your lab user account, then select **Save**. The sponsor is the identity accountable for the agent's existence and business purpose.

    ![Blueprint detail page showing an assigned owner and sponsor](./media/a365-ex1-t4-02.png)

    >**Note:** Sponsors can be users, dynamic membership groups, or Microsoft 365 groups. Security groups and role-assignable groups are **not** supported as sponsors. Groups are not supported as owners at all. In production, sponsors are usually the business owner and owners are usually the IT or platform team.

1. Now repeat this for the agent identity itself. Navigate to **Entra ID** > **Agents** > **Agent identities**.

1. Select the agent identity derived from your blueprint.

1. Assign the same **Owners** and **Sponsors** values as above.

    >**Note:** Blueprint-level and identity-level accountability are separate. In a real deployment, a platform team might own the blueprint while individual business units sponsor each agent identity created from it.

Finley now has an accountable owner and sponsor recorded in the directory.

### Task 5: Confirm the agent in the Agent 365 registry

The Microsoft Entra admin center shows you the identity. The **Agent 365 registry** in the Microsoft 365 admin center is the control plane where IT observes and governs every agent in the tenant, regardless of where it was built. Confirming the agent appears here proves the platform has accepted it.

1. Switch to the browser tab with the Microsoft 365 admin center, or navigate to:

    ```
    https://admin.microsoft.com/
    ```

1. In the left navigation pane, expand **Agents (1)** and select **Overview (2)**.

    ![Microsoft 365 admin center with Agents expanded and Overview selected](./media/a365-ex1-t5-01.png)

1. Review the **Agent overview** dashboard. Note the hero metrics: **Agent registry** total, **Active users**, and **Agent run-time**. Also note the governance action cards, which surface **Pending requests**, **Agents at risk**, and **Agents without owners**.

    ![Agent overview dashboard showing hero metrics and governance action cards](./media/a365-ex1-t5-02.png)

    >**Note:** The Active users and Agent run-time metrics only begin accumulating from the moment Agent 365 licenses are activated in the tenant, so they may show little or no data in a lab environment. This is expected.

1. Select **Agents (1)** > **All agents (2)**, and make sure the **Registry (3)** tab is selected.

    ![All agents page with the Registry tab selected showing the agent inventory](./media/a365-ex1-t5-03.png)

1. In the search box, type **finley** and confirm your agent appears in the registry.

1. Select your agent to open the agent details pane. Review the following:

   - The **Platform** the agent came from
   - The **Owner** you assigned in Task 4
   - The agent's **Permissions**
   - The **Status** of the agent

    ![Agent details pane showing platform, owner, permissions, and status](./media/a365-ex1-t5-04.png)

    >**Note:** If your agent does not appear in the registry, the blueprint may be missing the `managerApplications` property, which happens when a blueprint is created through the portal wizard instead of the CLI. Re-run `a365 setup all` to correct this. You can also re-run only the registration step with `a365 setup all --agent-registration-only`.

## Conclusion

In this exercise you gave Contoso's expense agent a real enterprise identity. You installed the Agent 365 CLI, provisioned an agent identity blueprint together with its Azure infrastructure and inherited Graph permissions, verified in Microsoft Entra that permissions were actually granted rather than left pending, assigned an owner and a sponsor to close the accountability gap, and confirmed the agent is visible in the Agent 365 registry.

The important idea here is sequencing. Identity comes first, because every governance control in the exercises that follow, whether a Conditional Access policy, a device compliance requirement, or an audit query, targets this identity. Without it, there is nothing to govern.

## Review

In this exercise, you have completed the following:

- Installed and verified the Agent 365 CLI
- Provisioned the agent blueprint using the Agent 365 CLI
- Verified the blueprint and permissions in Microsoft Entra
- Assigned owners and sponsors to establish accountability
- Confirmed the agent in the Agent 365 registry

## Summary

In this exercise, you created an agent identity blueprint and agent identity in Microsoft Entra using the Agent 365 CLI, established accountability through owners and sponsors, and verified the agent is discoverable and governable in the Agent 365 registry. This identity is the foundation for the policies, compute, and observability you configure in the remaining exercises.

Click **Next** from the lower right corner to move on to the next page.

![Next button in the lower right corner of the lab guide](./media/a365-gs-07.png)
