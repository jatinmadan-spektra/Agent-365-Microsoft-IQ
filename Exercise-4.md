# Exercise 4: End-to-End Governed Agent in Teams

### Estimated Duration: 60 minutes

Finley has an identity, policies, and compute. In this final exercise you build a low-code agent, ground it in Microsoft IQ, publish it to Microsoft Teams through an IT-approved flow, audit what it actually did, and then retire it.

## Overview

**Microsoft IQ** is the unified intelligence layer for enterprise AI. It has four capabilities: **Work IQ** (how people work), **Fabric IQ** (the live state of the business), **Foundry IQ** (curated institutional knowledge), and **Web IQ** (real-world intelligence from the web). This exercise uses **Work IQ**, because Contoso's expense agent needs Microsoft 365 context: mail, calendar, and files.

Work IQ is built on three layers: **Data** unifies signals from files, emails, meetings, chats, and business systems; **Memory** builds persistent understanding of how people and teams work; and **Inference** brings together models, skills, and tools so agents can reason and act through **Work IQ MCP tools**, while the Agent 365 control plane keeps every action observable and governed.

The critical governance point in this exercise is that Copilot Studio **automatically creates a Microsoft Entra Agent ID for every new agent**. That means the agent you build here is not an ungoverned side project; it inherits the identity model from Exercise 1 and falls under the Conditional Access policies you wrote in Exercise 2 without any extra work. Connector permissions appear as API permissions on that agent identity, so you can see and target them from Microsoft Entra.

>**Note:** This exercise deliberately builds a **second agent**, separate from Finley. Exercises 1 to 3 governed a pro-code agent provisioned through the Agent 365 CLI. This exercise governs a **low-code Copilot Studio agent**, which receives its identity automatically. The point is that both paths land in the same control plane: the same registry, the same Conditional Access policies, the same Defender tables. Do **not** delete Finley or the Conditional Access policies from Exercises 2 and 3 - this exercise depends on them.

## Prerequisites: licensing and roles

| Requirement | Needed for | Verified in this lab tenant |
| --- | --- | --- |
| **Microsoft 365 Copilot license** | Required to add Work IQ MCP servers to an agent in Copilot Studio | Yes - `M365_COPILOT_APPS`, `M365_COPILOT_BUSINESS_CHAT`, `M365_COPILOT_TEAMS`, `M365_COPILOT_SHAREPOINT` all provisioned |
| **Copilot Studio access** | Tasks 1 to 4 | Yes - `COPILOT_STUDIO_IN_COPILOT_FOR_M365` and `POWER_VIRTUAL_AGENTS_O365_P3` provisioned |
| **Microsoft Defender for AI** | Advanced hunting in Task 6 | Yes - `DEFENDER_FOR_AI` provisioned |
| **AI Administrator** or **Global Administrator** | Approving agent requests and lifecycle actions in Tasks 5 and 7 | Yes - Global Administrator |

>**Note:** **Work IQ MCP is a preview feature.** Copilot Studio currently offers only the preview version; general availability is coming. When Work IQ reaches GA in Copilot Studio, the Work IQ API transitions to a **usage-based (consumptive) billing model**, so cost management becomes relevant in production.

>**Note:** If Work IQ MCP tools are unavailable in your lab tenant's region, substitute any standard Copilot Studio connector tool. The publishing, approval, observability, and lifecycle tasks in Tasks 4 to 7 are unaffected. Administrators can also allow or block individual MCP servers in the Microsoft 365 admin center under **Agents and Tools**; if a server has been blocked tenant-wide it will not appear in the catalog. Microsoft notes that this allow/block capability **might not be available in every region yet**.

## In this exercise you will

- Create an agent in Microsoft Copilot Studio
- Ground it in Microsoft IQ by adding a Work IQ MCP tool and testing it
- Verify the agent's automatically created Entra Agent ID
- Publish the agent to Teams and Microsoft 365 Copilot
- Approve the agent as an administrator
- Trace the agent's activity in Microsoft Defender advanced hunting
- Exercise lifecycle governance actions from the Agent 365 registry

## Objectives

- **Task 1**: Create an agent in Microsoft Copilot Studio
- **Task 2**: Add a Work IQ MCP tool and test the agent
- **Task 3**: Verify the agent's Entra Agent ID and connector permissions
- **Task 4**: Publish the agent to Teams and Microsoft 365 Copilot
- **Task 5**: Approve the agent in the Microsoft 365 admin center
- **Task 6**: Trace agent activity in Microsoft Defender advanced hunting
- **Task 7**: Exercise lifecycle governance actions

### Task 1: Create an agent in Microsoft Copilot Studio

```admin.powerplatform.com login```
Copilot Studio is where you define what the agent does and says. This is low-code configuration, not programming.

1. On the lab virtual machine, open a new browser tab and navigate to Microsoft Copilot Studio:

    ```
    https://copilotstudio.microsoft.com/
    ```

1. If prompted, sign in with the following credentials:

   - **Email/Username:** <inject key="AzureAdUserEmail"></inject>
   
   - **Temporary Access Pass:** <inject key="AzureAdUserPassword"></inject>

1. New Copilot Studio experience pop up will appear, click on **Try now**.

    ![](./media/ex4-1.png)

1. On Copilot home screen, select **Agent**

    ![](./media/ex4-2.png)
  
1. Enter **Name (1)** as **Finley Expense Assistant** for your agent.

1. In the **Instructions (2)** field, enter the following system prompt:

    ```
    You are Finley, Contoso's expense assistant. Help employees understand expense policy and follow up on the status of submitted expense reports. Be concise and factual. If you are unsure about a policy detail, say so rather than guessing.
    ```
    ![](./media/ex4-3.png)

1. Select **Create**.

1. Wait for the agent to be created. When the authoring canvas opens, your agent is ready to configure.

    ![Copilot Studio authoring canvas for the newly created agent](./media/a365-ex4-t1-03.png)

### Task 2: Add a Work IQ MCP tool and test the agent

Right now your agent can talk about expenses but cannot act. In this task you give it a real capability by connecting a Work IQ MCP tool, then prove it works.

1. In your agent, select the **Tools** tab from the right pane.

    ![](./media/ex4-5.png)

1. On the **Add a tool** page, select **Model Context Protocol (MCP) (1)** to see the Work IQ MCP servers and other MCP servers.

1. Select **Work IQ Mail (Preview)** from the results.

    ![](./media/ex4-6.png)

    >Note: If not visible, type **Work IQ Mail** in the search box.

1. Expand the **connection (1)** dropdown and select **Create New Connection (2)**.
    
    ![](./media/ex4-7.png)

1. Select **Create**.

1. When prompted, provide your credentials and complete the sign-in process:

   - **Email/Username:** <inject key="AzureAdUserEmail"></inject>

   - **Password:** <inject key="AzureAdUserPassword"></inject>

1. Select **Add** to complete adding the tool.
    
    ![](./media/ex4-8.png)

1. Confirm **Work IQ Mail** now appears in your agent's tools list.

    ![](./media/ex4-9.png)

1. Now test the tool. Select **Preview (1)** to open the test pane.

1. Enter the following prompt (2):

    ```
    Send an email to <inject key="AzureAdUserEmail"></inject> with the subject "Finley test" and ask how the hands-on lab is going.
    ```
1. Press **Enter (3)**.
     
     ![](./media/ex4-10.png)

     >Note: If asked to allow the Work IQ tool to connect and use services, select **Allow**.

1. After a few moments, check your mailbox and confirm you received the email.

    ![](./media/ex4-11.png)

1. Select **Publish** in the top menu bar to publish your agent for the first time.

### Task 3: Verify the agent's Entra Agent ID

This is the task that ties this exercise back to the first two. Copilot Studio created an Entra Agent ID for your agent automatically, and that identity is subject to the policies you already wrote.

1. Navigate to the Microsoft Entra admin center tab.

    ```
    https://entra.microsoft.com
    ```

1. Navigate to **Entra ID (1)** > **Agents (2)** > **Agent identities (3)**.

1. You should now see **two** agent identities in this list (4): **finley-<inject key="DeploymentID" enableCopy="false"/> Identity** from Exercise 1, and the identity for the Copilot Studio agent you just built. Two very different build paths, one governance surface.

    ![](./media/ex4-12.png)

1. Optionally, assign the **AgentApprovalStatus** custom security attribute to this identity with the value **Finance_Approved**, following the same steps as Exercise 2, Task 3. This brings the agent into scope of your attribute-driven policy.

    >**Note:** This is worth doing if you have time. The policy you built in Exercise 2 blocks all agent identities **except** those tagged `Finance_Approved`. Without the tag, this new agent is exactly the kind of untagged agent that policy is designed to catch, which you can confirm in the policy's report-only impact data.

### Task 4: Publish the agent to Teams and Microsoft 365 Copilot

Now you give the agent a place to work. Publishing happens in two stages: first you install it for yourself as a trial, then you submit it for admin approval so the whole organization can find it.

1. Return to the Microsoft Copilot Studio tab and open your agent.

1. On the top menu bar, select **dropdown** next to publish.
  
    ![](./media/ex4-13.png)

1. Select the **Teams + Microsoft 365** tile to open its configuration panel.

    ![](./media/a365-ex4-t4-01.png)

1. Under **Turn on Microsoft 365**, ensure **Make agent available in Microsoft 365 Copilot** is selected.

1. Select **Add channel**.

    ![Teams and Microsoft 365 Copilot configuration panel with Add channel selected](./media/a365-ex4-t4-02.png)

1. Select **Edit details** and provide the information users will see in the app store:

   - Confirm the agent **short description** and **Long Description**
   
   - Select **More** and add a **Developer name** and **Website**

1. Select **Save**.

    ![Edit details panel showing agent appearance and developer information](./media/a365-ex4-t4-03.png)

1. Now install the agent for yourself. In the configuration panel, select **See agent in Teams**.

1. In the Teams dialog that opens, select **Add**.

    ![Teams Add agent dialog for the Finley agent](./media/a365-ex4-t4-04.png)

1. Confirm the agent appears in your Teams agent list, and send it a test message to verify it responds.

    ![Teams chat showing a conversation with the Finley agent](./media/a365-ex4-t4-05.png)

1. Now make it available to the organization. Return to the **Teams and Microsoft 365 Copilot** configuration panel in Copilot Studio and select **Availability options**.

1. Confirm the agent is **not** currently shown to teammates or shared users. If it shows **Added to Teams**, remove it first.

    >**Note:** If you submit an agent for admin approval while it is also shown in the **Built with Power Platform** section, it can end up appearing in two places in the app store.

1. Select **Show to everyone in my org**.

1. Review the submission requirements, then select **Submit to org catalog**.

    ![Availability options panel with Show to everyone in my org and Submit for admin approval](./media/a365-ex4-t4-06.png)

1. At the confirmation prompt, select **Yes**.

    >**Note:** After submitting, do not change the agent's access setting to less than everyone in your organization. Doing so causes users who install it from the app store to be unable to chat with it.

### Task 5: Approve the agent in the Microsoft 365 admin center

You now change hats. You submitted the agent as a maker; you approve it as the administrator. This separation is the whole point of governed deployment: nothing reaches the organization without an IT review of what it is asking for.

1. Switch to the Microsoft 365 admin center tab, or navigate to:

    ```
    https://admin.microsoft.com/
    ```

1. In the left navigation pane, expand **Agents (1)** and select **Overview (2)**.

1. On the **Agent overview** dashboard, locate the **Pending requests** card and select the option to manage requests.

    ![Agent overview dashboard with the Pending requests for agents card](./media/a365-ex4-t5-01.png)

    >**Note:** Card names on this dashboard change frequently as the preview evolves. If you cannot find the card, navigate directly to **Agents** > **All agents** and select the **Requests** tab.

1. On **All agents** > **Requests**, locate your agent **Finley Expense Assistant <inject key="DeploymentID" enableCopy="false"/>** in the list.

    ![Requests tab showing the pending agent submission](./media/a365-ex4-t5-02.png)

1. Select the agent to review its details, including the publisher, the platform it was built on, and the permissions it is requesting.

    >**Note:** This review step is the governance control. Read the requested permissions and ask whether each one is necessary for the agent's stated purpose. An expense assistant that requests broad mail write access deserves a question.

1. Select **Publish to store** to make the agent available to members of your organization.

    ![Agent request details pane with the Publish to store action](./media/a365-ex4-t5-03.png)

    >**Note:** The alternative action is **Reject submission**, which prevents the agent from becoming available. Both actions require the **AI Administrator** or **Global Administrator** role.

1. Now deploy the agent to users. Navigate to **Agents** > **All agents** and ensure the **Registry** tab is selected.

1. Select the **Status** filter and choose **Available**, then locate and select your agent.

    >**Note:** Applying the **Status: Available** filter is required before the **Install** action appears. This is the documented sequence.

1. In the agent details pane, immediately under the agent's name, select **Install**.

    ![Agent details pane with the Install action highlighted](./media/a365-ex4-t5-04.png)

1. In the **Deploy agent to selected users** pane, choose whether to install for all users or specific users and groups. For this lab, select **specific users** and choose your own account. Select **Next**.

    ![Deploy agent to selected users pane](./media/a365-ex4-t5-05.png)

1. In the **Review permissions** pane, review the requested permissions. If acceptable, select **Grant admin consent**.

1. In the **Permissions requested** window, select **Accept**, then select **Next**.

    ![Review permissions pane showing the agent permissions and Grant admin consent](./media/a365-ex4-t5-06.png)

1. In the **Review & finish** pane, select **Finish deployment**.

1. Verify the agent now appears with an installed status in the registry.

    >**Note:** If the agent does not appear in the **Built for your org** section of the Teams app store after approval, Teams is caching app information. Sign out of the Teams desktop client and sign back in, or refresh the browser if you are using Teams on the web. This is documented, expected behaviour, not a failure.

### Task 6: Trace agent activity in Microsoft Defender advanced hunting

Governance without observability is a promise you cannot verify. In this task you query the actual record of what your agent is and what it can do.

1. Open a new browser tab and navigate to the Microsoft Defender portal:

    ```
    https://security.microsoft.com/
    ```

1. If prompted, sign in with the following credentials:

   - **Email/Username:** <inject key="AzureAdUserEmail"></inject>
   - **Password:** <inject key="AzureAdUserPassword"></inject>

1. In the left navigation pane, expand **Hunting** and select **Advanced hunting**.

    ![Microsoft Defender portal with Advanced hunting selected under Hunting](./media/a365-ex4-t6-01.png)

1. In the query editor, enter the following KQL query to list the agents in your tenant, then select **Run query**:

    ```
    AgentsInfo
    | summarize arg_max(Timestamp, *) by AgentId
    | project Timestamp, AgentName, Platform, EntraAgentId, PublishedStatus, LifecycleStatus, Owners
    | order by Timestamp desc
    ```

    ![Advanced hunting results showing agents from the AgentsInfo table](./media/a365-ex4-t6-02.png)

    >**Important:** Note the `summarize arg_max(Timestamp, *) by AgentId` line. The `AgentsInfo` table stores **multiple snapshots of each agent over time**, so a query without this line returns many rows for the same agent and looks confusing. `arg_max` returns only the latest state of each agent. Use this pattern in every query against this table.

    >**Note:** Use the **`AgentsInfo`** table. The older `AIAgentsInfo` table remained accessible only until **1 July 2026** and queries written against it now fail. If you find older documentation or scripts referencing `AIAgentsInfo`, they need migrating.

1. Narrow the query to your own agent:

    ```
    AgentsInfo
    | where AgentName contains "Finley"
    | summarize arg_max(Timestamp, *) by AgentId
    | project Timestamp, AgentName, Platform, EntraAgentId, EntraBlueprintId, LifecycleStatus, Availability, Owners
    ```

    >**Note:** `PublishedStatus` returns either `Draft` or `Published`. `LifecycleStatus` returns `Active`, `Blocked`, `Uninstalled`, or `Deleted`. Knowing the exact permitted values makes it much easier to confirm that a governance action actually took effect, which you will use in Task 7.

    >**Note:** If the query returns no rows, advanced hunting has **ingestion latency** and the data has not arrived yet. Wait a few minutes and run the query again. An empty first result is normal and is not a sign that anything is broken.

1. Now inspect what tools and MCP servers the agent has been given. Run the following query:

    ```
    AgentsInfo
    | where AgentName contains "Finley"
    | summarize arg_max(Timestamp, *) by AgentId
    | project AgentName, McpServers, DeclaredTools, DeclaredDataSources, Permissions
    ```

    ![Advanced hunting results showing MCP servers and declared tools for the agent](./media/a365-ex4-t6-03.png)

    >**Note:** The `McpServers` column shows the MCP servers connected to the agent, including server URLs and credential configuration. This is how a security analyst answers "what can this agent actually reach?" without opening Copilot Studio. `Permissions` shows requested and granted permissions with their approval state.

1. Review the agent's system prompt and model, which reveal what the agent was instructed to do:

    ```
    AgentsInfo
    | where AgentName contains "Finley"
    | summarize arg_max(Timestamp, *) by AgentId
    | project AgentName, Model, Instructions, Channels, Capabilities, Guardrails
    ```

    >**Note:** The `Instructions` column contains the exact system prompt you typed in Task 1. Being able to read an agent's instructions from a security portal, without access to the authoring tool, is how an analyst determines intent during an investigation.

1. Optionally, inspect the agent's runtime surface and multi-agent connections:

    ```
    AgentsInfo
    | where AgentName contains "Finley"
    | summarize arg_max(Timestamp, *) by AgentId
    | project AgentName, Endpoints, ConnectedAgents, Triggers, Memory, InstanceCount, ObservabilityId
    ```

    >**Note:** `Endpoints` lists agent runtime endpoints including URL, transport type, and an external connectivity flag. `ConnectedAgents` lists other agents wired in for multi-agent orchestration, which is how you discover an agent chain nobody documented.

1. Finally, run a tenant-wide posture query that would be useful in a real environment, summarising agents by platform and lifecycle state:

    ```
    AgentsInfo
    | summarize arg_max(Timestamp, *) by AgentId
    | summarize AgentCount = count() by Platform, LifecycleStatus
    | order by AgentCount desc
    ```

    ![Advanced hunting results summarising agent counts by platform and lifecycle status](./media/a365-ex4-t6-04.png)

    >**Note:** This last query is the kind of thing you would pin to a dashboard. It answers "how many agents do we have, on what platforms, and how many are blocked or uninstalled?" which is exactly the question Contoso could not answer at the start of this lab.

### Task 7: Exercise lifecycle governance actions

Finally, prove you can control the agent after deployment. An agent you cannot suspend, reassign, or remove is not governed.

1. Switch to the Microsoft 365 admin center tab.

1. Navigate to **Agents** > **All agents** and ensure the **Registry** tab is selected.

1. Locate and select your agent **Finley Expense Assistant <inject key="DeploymentID" enableCopy="false"/>**.

    >**Note:** To find it quickly, use the **Platform** filter and select **Copilot Studio**.

1. **Suspend the agent.** In the agent details pane, immediately under the agent's name, select **Block**.

1. In the **Block agent** pane, select **Block agent**, then select **Save**.

    ![Block agent pane with the Block agent option selected](./media/a365-ex4-t7-01.png)

    >**Note:** Blocking restricts access to the agent across the organization so no user can use it. It is reversible and is the correct first action when an agent is behaving unexpectedly, because it stops the behaviour without destroying evidence. For agents built in Copilot Studio or Agent Builder, blocking affects availability in Microsoft 365 Copilot **and** host products such as Outlook and Teams. For agents built in SharePoint or Microsoft Foundry, blocking only affects Microsoft 365 Copilot Chat.

1. Verify the agent's status now shows as blocked in the registry.

1. **Restore the agent.** Select the agent again and select **Unblock**, then confirm.

1. **Reassign ownership.** With the agent selected, in the details pane select **Assign new owner**.

1. In the **Assign a new owner** pane, enter a user from your organization and select **Assign**.

    ![Assign a new owner pane](./media/a365-ex4-t7-02.png)

    >**Note:** After reassignment, the new owner gets full edit and delete permissions plus access to any files the previous owner uploaded, and the previous owner loses all access including read rights. This action is supported **only** for Agent Builder and Copilot Studio agents, which is why it works for this agent but would not work for Finley from Exercise 1.

1. **Uninstall the agent.** Select the agent, then select **Uninstall**.

    >**Note:** If you do not see the **Uninstall** option, the agent is not currently installed. Confirm you completed the **Install** step in Task 5, and that the **Status: Available** filter is applied.

1. In the **Remove agent** pane, select the **Remove agent** option, then select the **Uninstall Agent** button.

    ![Remove agent pane with the Uninstall Agent button](./media/a365-ex4-t7-03.png)

    >**Note:** Uninstalling affects the agent's availability and functionality in Copilot and in host products such as Outlook and Teams. It is distinct from **Delete**, which permanently removes the agent, all associated files, and the underlying SharePoint Embedded container. Deletion is irreversible and can take up to 24 hours to reach all users, during which users may still see the agent listed but cannot interact with it.

    >**Note:** In the Microsoft 365 admin center, **Delete** is documented for agents created with **Microsoft 365 Copilot Agent Builder**, reached through the vertical ellipses (**⁝**) next to the agent. It is not the path for retiring a Copilot Studio agent. To delete a Copilot Studio agent, delete it in **Copilot Studio** - which also deletes its associated Microsoft Entra Agent ID automatically.

1. Return to **Agents** > **Overview** and confirm the governance cards reflect your changes.

1. Re-run the advanced hunting query from Task 6 to observe the `LifecycleStatus` column change:

    ```
    AgentsInfo
    | where AgentName contains "Finley"
    | summarize arg_max(Timestamp, *) by AgentId
    | project Timestamp, AgentName, LifecycleStatus, Availability
    ```

    >**Note:** Expect `LifecycleStatus` to read **`Uninstalled`** after the uninstall action. Had you stopped after blocking, it would read **`Blocked`**. Remember ingestion latency - the change may take a few minutes to appear.

    >**Troubleshooting:** If a lifecycle action appears to succeed in the UI but never takes effect on the agent, the agent may live in a Power Platform environment configured with **Power Platform Firewall** in **active enforcement mode**, which rejects admin actions originating from the Microsoft 365 admin center. Check this in the Power Platform admin center under **Security** > **Identity and access** > **IP firewall**, on the **Advanced** tab. If **IP Firewall** is **On** and **Turn on IP firewall in audit-only mode** is **Off**, the environment is in active enforcement mode. In that state the action must be run directly against the Power Platform API instead.

## Conclusion

In this exercise you completed the agent lifecycle. You built a low-code agent in Copilot Studio, grounded it in Microsoft IQ through a Work IQ MCP tool and proved the grounding worked by having it send a real email, confirmed that Copilot Studio had automatically issued it an Entra Agent ID subject to the policies you wrote in Exercise 2, published it to Teams through an approval flow that required an administrator to review its permissions, audited it in Defender advanced hunting, and finally suspended, reassigned, and removed it.

The thread running through all four exercises is that governance is not a gate you add at the end. The agent was governable in Exercise 4 because the identity model existed in Exercise 1 and the policies existed in Exercise 2. Note also that the two agents in this lab were built in completely different ways - one through a developer CLI, one through a low-code studio - and both landed in the same registry, under the same Conditional Access policies, in the same Defender table. That convergence is the whole point of a control plane.

## Review

In this exercise, you have completed the following:

- Created an agent in Microsoft Copilot Studio
- Added a Work IQ MCP tool and tested the agent
- Verified the agent's Entra Agent ID and connector permissions
- Published the agent to Teams and Microsoft 365 Copilot
- Approved the agent in the Microsoft 365 admin center
- Traced agent activity in Microsoft Defender advanced hunting
- Exercised lifecycle governance actions

## Summary

In this exercise, you built an agent in Copilot Studio, grounded it in Microsoft 365 context using a Work IQ MCP tool, verified its automatically created Entra Agent ID and connector permissions, published it to Teams through an IT-approved flow, traced its properties and tooling in Microsoft Defender advanced hunting using the `AgentsInfo` table, and exercised block, reassign, and uninstall lifecycle actions from the Agent 365 registry.

You have now governed agents across identity, policy, compute, intelligence, and lifecycle, which is the complete Agent 365 control plane.

### You have successfully completed the lab. Click on **Next** to proceed.

![Next button in the lower right corner of the lab guide](./media/a365-gs-07.png)
