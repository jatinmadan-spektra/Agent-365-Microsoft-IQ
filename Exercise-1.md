# Exercise 1: Entra Agent ID - Identity for Agents

### Estimated Duration: 60 minutes

Before an AI agent can be governed, it has to exist as a first-class identity in your directory. In this exercise you give Contoso's expense-processing agent, **Finley**, an enterprise identity in Microsoft Entra, establish who is accountable for it, and confirm it is discoverable in the Agent 365 registry.

## Overview

Human employees get an identity before they get a laptop or a mailbox. AI agents work the same way. **Microsoft Entra Agent ID** provides purpose-built identity constructs for agents so you can authenticate, authorize, govern, and protect them at enterprise scale.

Two constructs matter in this exercise:

- An **agent identity blueprint** is the reusable template that defines a class of agent. It declares which permissions agents of this kind may request and who is accountable for the class. If Contoso later wants ten expense agents, all ten come from this one blueprint. Every blueprint also has an **agent blueprint principal** - the service principal that actually holds the granted permissions.
- An **agent identity** is a specific agent created from that blueprint. This is Finley. It is the identity that actually requests access tokens and performs work.

You will also confirm an **owner** and a **sponsor**. A sponsor is the person accountable for why the agent exists, similar to a hiring manager, and is required. An owner is the person who can change the agent's configuration, and is recommended.

>**Note:** You will create the blueprint using the **Agent 365 CLI**, not the Microsoft Entra admin center wizard. The admin center does have a **New agent blueprint (Preview)** wizard, and it works - but it only creates the blueprint, its principal, and its owners and sponsors. It does **not** configure the inheritable Microsoft Graph permissions that agent identities need, and it does **not** register the agent in the Agent 365 registry, which is a separate Agent Registry API call. The CLI does all three in one command, which is why this exercise uses it.

In this exercise you will:

- Verify the Agent 365 CLI and authenticate to Azure
- Provision the agent blueprint, its principal, permissions, and agent identity in a single command
- Verify the blueprint and its granted permissions in the Microsoft Entra admin center
- Confirm and complete owner and sponsor accountability in the portal
- Confirm the agent appears in the Agent 365 registry

## Objectives

- **Task 1**: Verify the Agent 365 CLI and authenticate to Azure
- **Task 2**: Provision the agent blueprint using the Agent 365 CLI
- **Task 3**: Verify the blueprint and permissions in Microsoft Entra
- **Task 4**: Confirm and complete owner and sponsor accountability
- **Task 5**: Confirm the agent in the Agent 365 registry

### Task 1: Verify the Agent 365 CLI and authenticate to Azure

In this task you prepare your tooling. The Agent 365 CLI is a .NET global tool and is **pre-installed on the lab virtual machine**, so you verify it rather than install it, then authenticate to Azure.

1. On the lab virtual machine, select the **Start** button, type **Windows PowerShell**, and select **Windows PowerShell**.

    ![Windows Start menu with Windows PowerShell in the search results](./media/a365-ex1-t1-01.png)

1. Confirm that .NET 8.0 or later is installed by running the following command:

    ```
    dotnet --version
    ```

    You should see **8.0.417** or later.

    >**Note:** The Agent 365 CLI requires .NET 8.0 or later. If the command returns a version lower than 8.0, contact lab support.

1. Confirm the Agent 365 CLI is already installed by listing your .NET global tools:

    ```
    dotnet tool list --global
    ```

    You should see **microsoft.agents.a365.devtools.cli** in the list.

    ![PowerShell showing microsoft.agents.a365.devtools.cli in the global tool list](./media/a365-ex1-t1-02.png)

    >**Note:** If the tool is **not** listed, install it with `dotnet tool install --global Microsoft.Agents.A365.DevTools.Cli`. Do not run the install command if it is already listed - it will fail. Use `dotnet tool update --global Microsoft.Agents.A365.DevTools.Cli` if you need a newer version.

1. Verify the CLI runs by displaying its help:

    ```
    a365 -h
    ```

    You should see the CLI version banner and the list of available commands, including `setup`, `query-entra`, `publish`, and `cleanup`. This confirms the CLI is on your path and ready to use.

    ![PowerShell showing the a365 CLI help output with available commands](./media/a365-ex1-t1-03.png)

    >**Note:** If `a365` is not recognized, close and reopen PowerShell so the shell picks up the .NET global tools folder on your path.

1. Authenticate to Azure. The CLI resolves your tenant from the Azure CLI context, so it needs an authenticated Azure session.

    ```
    az login
    ```

1. A browser window opens. Sign in with the following credentials:

   - **Email/Username:** <inject key="AzureAdUserEmail"></inject>
   - **Password:** <inject key="AzureAdUserPassword"></inject>

1. Return to PowerShell and confirm the correct subscription and tenant are selected:

    ```
    az account show
    ```

    ![PowerShell showing the az account show output with the active subscription](./media/a365-ex1-t1-04.png)

    >**Note:** If you are already signed in from a previous session, `az account show` returns your context and you can skip `az login`.

You now have the Agent 365 CLI verified and an authenticated Azure context.

### Task 2: Provision the agent blueprint using the Agent 365 CLI

In this task you run a single command that does a surprising amount of work: it registers the agent identity blueprint and its principal in Microsoft Entra, configures the inheritable Microsoft Graph and Agent 365 permissions that agent identities will receive, creates the agent identity, and registers the agent in the Agent 365 registry.

>**Note:** This command is **interactive**. It pauses for a confirmation prompt and opens a browser for admin consent. Stay at the keyboard for the whole run - do not start it and walk away.

1. In PowerShell, create and move into a clean working folder for the agent. The CLI writes its configuration output here, so keep track of this folder.

    ```
    New-Item -ItemType Directory -Path C:\LabFiles\finley -Force
    cd C:\LabFiles\finley
    ```

    >**Note:** Use an empty folder. If the folder already contains an `a365.config.json` from a previous run, the CLI may reuse that agent's name instead of the one you pass.

1. Run the setup command. This creates the blueprint in config-free mode, meaning the CLI resolves your tenant and client app automatically without requiring a configuration file.

    ```
    a365 setup all --agent-name finley-<inject key="DeploymentID" enableCopy="false"/>
    ```

    >**Note:** Do not use spaces or special characters in the agent name. Setup typically takes 4 to 8 minutes, most of which is waiting on admin consent.

1. The CLI first validates prerequisites. You will see a **Warn** result for **Frontier Preview Program** stating that tenant enrollment cannot be verified automatically. **This warning is expected in this lab and does not block setup.** Setup continues as long as the summary line reads `0 failed`.

1. The CLI creates the blueprint application, its service principal, and a client secret. The secret value is printed to the console **once**.

    ```
    Blueprint client secret: <value shown once>
    ```

    >**Note:** You do **not** need the secret for any task in this lab, because the agent is not deployed here. Do not paste it into chat, a document, or source control. If you ever need it later, run `a365 setup blueprint --show-secret` from this same folder, on this same machine, as this same user.

1. The CLI then configures inheritable permissions and pauses at a prompt:

    ```
    Assign this application permission now? [y/N]:
    ```

    Type **y** and press **Enter**.

    >**Note:** The default is **N**. If you press Enter without typing `y`, setup prints **Setup cancelled** and stops. If that happens, simply re-run the same `a365 setup all` command - it detects the existing blueprint and resumes.

1. The CLI opens a browser window requesting **admin consent** for the delegated permissions the blueprint needs. Review the requested permissions and select **Accept**.

    ![Browser showing the admin consent dialog for the agent blueprint permissions](./media/a365-ex1-t2-01.png)

    >**Note:** The CLI polls for consent for **180 seconds**, printing progress messages such as `Still waiting for admin consent... (34s / 180s)` while it waits. Complete the consent within that window.

1. **Expected behavior:** after you select **Accept**, the browser tab will most likely show an error page similar to the following:

    ```
    Try that again using a different browser
    We couldn't connect to that service, likely because of settings put in place
    by your IT team. Open Azure in a different Web browser to try again.
    ```

    **This is expected and harmless. Your consent has already succeeded.** Do not switch browsers, and do not re-run consent. Simply leave the tab and return to PowerShell.

    >**Note:** Why this happens: Entra commits the tenant-wide permission grant at the moment you select **Accept**. It then redirects your browser to the CLI application's registered reply URL, which is `http://localhost:8400/`. The CLI does not run a local listener for this step - it polls Microsoft Graph instead - so nothing answers on that port and the browser renders the error page above. The message about "settings put in place by your IT team" is generic text and does not reflect any policy in this lab tenant.

1. Return to PowerShell and wait for the CLI to confirm:

    ```
    Consent granted (All permissions).
    ```

    Once you see this line, consent completed successfully regardless of what the browser tab displayed.

    >**Note:** Consent has only genuinely failed if the CLI reaches the 180-second timeout **without** printing `Consent granted`. In that case, re-run `a365 setup all --agent-name finley-<inject key="DeploymentID" enableCopy="false"/>` and type **y** at the permission prompt. You can independently verify the grants at any time by running `a365 query-entra blueprint-scopes`, which should report `4 of 4 resource(s) have at least one granted permission on the blueprint SP`.

1. Wait for the command to complete and read the **Setup Summary** block. This is your authoritative success check. Confirm all seven lines and the final message:

    ```
    Setup Summary

      1. Prerequisites                validated
      2. Blueprint                    created  'finley-<DeploymentID> Blueprint' (ID: ...)
      3. Inheritable Permissions      configured
      4. Blueprint Permission Grants  granted  tenant-wide delegated
      5. Agent identity               created  'finley-<DeploymentID> Identity' (ID: ...)
      6. Agent Registration           registered  'finley-<DeploymentID> Agent' (ID: T_...)
      7. Messaging endpoint           skipped (non-M365 agent)

    Setup completed successfully
    ```

    ![PowerShell showing the completed a365 setup all summary output](./media/a365-ex1-t2-02.png)

    >**Note:** Line 7 reads **skipped (non-M365 agent)** and that is correct for this lab. A messaging endpoint is only registered when you pass `--m365`, which requires an already-deployed HTTPS endpoint. Finley is not deployed in this exercise.

    >**Note:** If you re-run the command, line 2 reads **reused** instead of **created**. That is expected and harmless.

1. Inspect the generated configuration file to capture the identifiers you will need in later exercises:

    ```
    Get-Content a365.generated.config.json | ConvertFrom-Json
    ```

1. Confirm the following key values are present, and copy them into a text file for reference:

    | Field | What it is | What to check |
    | --- | --- | --- |
    | `agentBlueprintId` | The blueprint's application (client) ID | Should be a valid GUID |
    | `agentBlueprintObjectId` | The blueprint's object ID in Microsoft Entra | Should be a valid GUID. For agent blueprints this is the **same value** as `agentBlueprintId` |
    | `agentBlueprintServicePrincipalObjectId` | The object ID of the agent blueprint **principal**, which holds the granted permissions | Should be a valid GUID, different from the two above |
    | `agenticAppId` | The **agent identity** created from the blueprint | Should be a valid GUID |
    | `agentRegistrationId` | The agent's entry in the Agent 365 registry | Should start with `T_` |
    | `resourceConsents` | The APIs consented to | Should list four resources: **Microsoft Graph**, **Agent 365 Tools**, **Observability API**, **Power Platform API**, each with `consentGranted: True` |

    ![PowerShell showing the parsed contents of a365.generated.config.json](./media/a365-ex1-t2-03.png)

    >**Note:** This file also contains a `completed` field. In the current CLI version it remains **False** even after a fully successful run, so **ignore it**. Use the **Setup Summary** block from the previous step and the `consentGranted: True` values above as your success check.

    >**Note:** If any entry in `resourceConsents` shows `consentGranted: False`, the OAuth2 permission grants did not finish. Re-run `a365 setup all --agent-name finley-<inject key="DeploymentID" enableCopy="false"/>`, type **y** at the prompt, and complete the consent prompt.

Finley now has a registered agent identity blueprint, an agent identity, and a registry entry.

### Task 3: Verify the blueprint and permissions in Microsoft Entra

Creating the blueprint through the CLI is fast, but as an administrator you should always confirm what was actually created in the directory. This entire task is done in the portal.

>**Note:** Agent blueprints are a **distinct object type** in Microsoft Entra, not ordinary app registrations, and their permissions are held on the blueprint **principal** rather than declared on the application. For that reason you verify them under **Agents**, not under **App registrations**. The **App registrations > API permissions** blade will show an empty **Configured permissions** list for an agent blueprint even when permissions are correctly granted.

1. Open a new browser tab and navigate to the Microsoft Entra admin center:

    ```
    https://entra.microsoft.com
    ```

1. In the left navigation pane, expand **Entra ID (1)**, select **Agents (2)**, and then select **Agent blueprints (3)**.

    ![Microsoft Entra admin center showing Entra ID, Agents, and Agent blueprints in the navigation](./media/a365-ex1-t3-01.png)

1. In the search box, type **finley** and confirm your blueprint **finley-<inject key="DeploymentID" enableCopy="false"/> Blueprint** appears in the list.

    >**Note:** To find the blueprint by GUID instead, select **Add filters**, add the **Blueprint App ID** filter, and paste the `agentBlueprintId` you recorded in Task 2. The default search box matches on name or **object ID** of the principal, not the blueprint application ID.

1. Select your blueprint to open its management page.

    ![Agent blueprint management page for the finley blueprint](./media/a365-ex1-t3-02.png)

1. Under **Access**, select **Granted permissions**.

1. Make sure the **Admin consent** tab is selected. Confirm you see entries for all four resources, and review the **API name**, **Claim value**, **Permission**, **Type**, and **Granted through** columns.

    | API name | Claim values you should see |
    | --- | --- |
    | Microsoft Graph | `Mail.ReadWrite`, `Mail.Send`, `Chat.ReadWrite`, `User.Read.All`, `Sites.Read.All`, `Files.ReadWrite.All`, `ChannelMessage.Read.All`, `ChannelMessage.Send` |
    | Agent 365 Tools | `McpServersMetadata.Read.All` |
    | Observability API | `Agent365.Observability.OtelWrite` |
    | Power Platform API | `Connectivity.Connections.Read` |

    ![Granted permissions page with the Admin consent tab selected showing permissions for all four APIs](./media/a365-ex1-t3-03.png)

    >**Note:** These are granted tenant-wide as admin consent, which is why they appear on the **Admin consent** tab and not the **User consent** tab. Permissions missing here will cause failures in Exercise 4 when the agent tries to call Microsoft 365 services. If any are missing, return to Task 2 and re-run `a365 setup all`.

1. In the left menu, select **Linked agent identities**. Confirm that the agent identity **finley-<inject key="DeploymentID" enableCopy="false"/> Identity** appears, and note its **Status** and **Owners and Sponsors** columns.

    ![Linked agent identities page listing the agent identity created from the blueprint](./media/a365-ex1-t3-04.png)

    >**Note:** Every agent identity is a child of exactly one agent identity blueprint. This parent-child relationship is what allows you to apply a single Conditional Access policy to an entire class of agents, which you will do in Exercise 2.

1. Now confirm the identity is also visible in the tenant-wide list. Navigate to **Entra ID** > **Agents** > **Agent identities**, search for **finley**, and confirm your agent identity appears.

    ![Agent identities page listing the agent identity](./media/a365-ex1-t3-05.png)

1. **Optional CLI cross-check.** If you want to see the same information from the command line, return to PowerShell in `C:\LabFiles\finley` and run:

    ```
    a365 query-entra blueprint-scopes
    a365 query-entra inheritance
    ```

    The first lists the permissions granted on the blueprint principal. The second confirms that agent identities will actually inherit them, reporting `kind=allAllowed` per resource.

    >**Note:** `a365 query-entra inheritance` reports **WARN** on the **Roles** line for Microsoft Graph, Agent 365 Tools, and Power Platform API, because only the Observability API has an application permission granted. This is expected for this lab. What matters is the per-resource **Effective inheritance: OK** line and the summary reading `4 of 4 resource(s) have effective inheritance`.

### Task 4: Confirm and complete owner and sponsor accountability

An agent with no accountable human is a governance gap, and the Microsoft 365 admin center surfaces "agents without owners" as a governance action precisely because ownerless agents accumulate risk.

The Agent 365 CLI assigns you as **owner and sponsor of the agent blueprint** when it creates it, but it stops there. In this task you verify what the CLI did, then close the two gaps it leaves: the **agent blueprint principal has no owner or sponsor**, and the **agent identity has a sponsor but no owner**.

1. Still in the Microsoft Entra admin center, navigate to **Entra ID** > **Agents** > **Agent blueprints**.

1. Select your blueprint **finley-<inject key="DeploymentID" enableCopy="false"/> Blueprint**.

1. Under **Access**, select **Owners and sponsors**.

1. Select the **Agent blueprint** tab. Confirm that your lab user account is already listed as both an **Owner** and a **Sponsor**.

    ![Owners and sponsors page for the blueprint showing the lab user as owner and sponsor](./media/a365-ex1-t4-01.png)

    >**Note:** The blueprint creator is automatically set as owner and sponsor of the **agent blueprint**, which is why there is nothing to add on this tab - you are confirming accountability, not creating it. This does **not** extend to the blueprint principal or the agent identity, which you handle in the following steps.

1. Select the **Agent blueprint principal** tab. Unlike the **Agent blueprint** tab, this tab is **empty** - the CLI does not populate it. You will add both an owner and a sponsor here.

1. Select **Add** > **Add owner**, search for and select your lab user account, and then select **Add**.

1. Select **Add** > **Add sponsor**, search for and select your lab user account, and then select **Add**.

1. Confirm the **Agent blueprint principal** tab now lists your lab user account as both an owner and a sponsor.

    ![Agent blueprint principal tab showing an assigned owner and sponsor](./media/a365-ex1-t4-01b.png)

    >**Note:** The agent blueprint and the agent blueprint principal are separate directory objects, and accountability is tracked separately on each. The blueprint is the template definition; the principal is the identity in your tenant that actually holds the granted permissions you reviewed in Task 3. Both need an accountable owner.

1. Now handle the agent identity, where a real gap exists. Navigate to **Entra ID** > **Agents** > **Agent identities**.

1. Select the agent identity **finley-<inject key="DeploymentID" enableCopy="false"/> Identity**.

1. Under **Access**, select **Owners and sponsors**. Note that a **Sponsor** is already recorded but the **Owners** list is **empty**.

1. Select **Add** > **Add owner**.

1. Search for and select your lab user account, then select **Add**.

    ![Owners and sponsors page for the agent identity showing an assigned owner and sponsor](./media/a365-ex1-t4-02.png)

1. Confirm the identity now shows both an owner and a sponsor.

    >**Note:** Sponsors can be users, dynamic membership groups, or Microsoft 365 groups. Security groups and role-assignable groups are **not** supported as sponsors. Groups are not supported as owners at all. When you use a dynamic membership group as a sponsor, allow up to 24 hours after a membership or user property change before sponsorship authorization checks succeed.

    >**Note:** Blueprint-level and identity-level accountability are separate. In a real deployment, a platform team might own the blueprint while individual business units sponsor each agent identity created from it.

Finley now has an accountable owner and sponsor recorded on all three objects: the agent blueprint, the agent blueprint principal, and the agent identity.

### Task 5: Confirm the agent in the Agent 365 registry

The Microsoft Entra admin center shows you the identity. The **Agent 365 registry** in the Microsoft 365 admin center is the control plane where IT observes and governs every agent in the tenant, regardless of where it was built. Confirming the agent appears here proves the platform has accepted it.

1. Switch to the browser tab with the Microsoft 365 admin center, or navigate to:

    ```
    https://admin.microsoft.com/
    ```

1. In the left navigation pane, expand **Agents (1)** and select **Overview (2)**.

    ![Microsoft 365 admin center with Agents expanded and Overview selected](./media/a365-ex1-t5-01.png)

1. Review the **Agent overview** dashboard and its governance action cards, which surface items such as **Pending requests**, **Agents at risk**, and **Agents without owners**.

    ![Agent overview dashboard showing hero metrics and governance action cards](./media/a365-ex1-t5-02.png)

    >**Note:** Usage-based metrics only begin accumulating from the moment Agent 365 licenses are activated in the tenant, so they may show little or no data in a lab environment. This is expected. Card names and layout on this page change frequently as the preview evolves - focus on locating the governance cards rather than matching exact tile titles.

1. Select **Agents (1)** > **All agents (2)**, and make sure the **Registry (3)** tab is selected.

    ![All agents page with the Registry tab selected showing the agent inventory](./media/a365-ex1-t5-03.png)

1. Note the tenant-wide summary tiles above the list: **Total agents**, **Agents without owners**, and **Unmanaged agents**.

    >**Note:** Applying filters to the list does not change the **Total agents** count.

1. In the search box, type **finley** and confirm your agent appears in the registry.

    >**Note:** If you do not see it, select **Refresh**, and check that no **Status**, **Publisher type**, **Channel**, **Platform**, or **Data source** filter is still applied from an earlier step. Registry entries can also take a few minutes to surface after registration.

1. Confirm the **Status** column reads **Available** and the **Risks** column reads **0**.

    >**Note:** The **Platform** column will be **empty**, and this is expected. Platform records which authoring product an agent was built in, such as Copilot Studio or Agent Builder. Finley was provisioned directly by the Agent 365 CLI, so there is no authoring platform to report. The **Channel** column is empty for the same reason - you provisioned a non-M365 agent with no messaging endpoint, so Finley is not yet deployed to Teams, Outlook, or Copilot. Neither column is a governance signal. **Status** and **Risks** are the values that matter here.

1. Select your agent to open the agent details pane. Review the following:

   - The **Owner** you confirmed in Task 4
   - The agent's **Permissions**
   - The **Status** of the agent

    >**Note:** If you have run this exercise more than once in the same tenant, you may see **duplicate agent rows** with the same name. Registry entries are not always removed when the underlying blueprint is deleted, so an earlier run can leave an orphaned entry behind. Add the **Date created** column using **Customize view** to identify the current one. The orphaned entry shows no owner and no permissions in its details pane, because its Entra identity no longer exists, and can be removed with the **Delete** action on its **⋮** menu.

    ![Agent details pane showing platform, owner, permissions, and status](./media/a365-ex1-t5-04.png)

    >**Note:** If your agent does not appear in the registry at all, the registry registration step did not complete. Registration is a separate call from blueprint creation - creating a blueprint in Microsoft Entra alone does not put it in the registry. Re-run only the registration step with `a365 setup all --agent-registration-only` from `C:\LabFiles\finley`. This is also why this exercise provisions with the CLI rather than the Microsoft Entra admin center wizard: the wizard creates the blueprint but never calls the Agent Registry API.

## Conclusion

In this exercise you gave Contoso's expense agent a real enterprise identity. You verified the Agent 365 CLI, provisioned an agent identity blueprint together with its principal, its inheritable Graph permissions, an agent identity, and a registry entry, verified in the Microsoft Entra admin center that permissions were actually granted rather than left pending, confirmed and completed owner and sponsor accountability at both the blueprint and identity level, and confirmed the agent is visible in the Agent 365 registry.

The important idea here is sequencing. Identity comes first, because every governance control in the exercises that follow, whether a Conditional Access policy, a device compliance requirement, or an audit query, targets this identity. Without it, there is nothing to govern.

## Review

In this exercise, you have completed the following:

- Verified the Agent 365 CLI and authenticated to Azure
- Provisioned the agent blueprint using the Agent 365 CLI
- Verified the blueprint and permissions in Microsoft Entra
- Confirmed and completed owner and sponsor accountability
- Confirmed the agent in the Agent 365 registry

## Summary

In this exercise, you created an agent identity blueprint and agent identity in Microsoft Entra using the Agent 365 CLI, established accountability through owners and sponsors, and verified the agent is discoverable and governable in the Agent 365 registry. This identity is the foundation for the policies, compute, and observability you configure in the remaining exercises.

Click **Next** from the lower right corner to move on to the next page.

![Next button in the lower right corner of the lab guide](./media/a365-gs-07.png)
