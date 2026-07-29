# Exercise 4: End-to-End Governed Agent in Teams

### Estimated Duration: 60 minutes

Finley has an identity, policies, and compute. In this final exercise you give it real organizational context using Microsoft IQ, publish it to Microsoft Teams through an IT-approved flow, audit what it actually did, and then retire it.

## Overview

**Microsoft IQ** is the unified intelligence layer for enterprise AI. It has four capabilities: **Work IQ** (how people work), **Fabric IQ** (the live state of the business), **Foundry IQ** (curated institutional knowledge), and **Web IQ** (real-world intelligence from the web). This exercise uses **Work IQ**, because Contoso's expense agent needs Microsoft 365 context: mail, calendar, and files.

Work IQ is built on three layers: **Data** unifies signals from files, emails, meetings, and chats; **Context** builds persistent understanding of how people and teams work; and **Skills and Tools** lets agents reason and act through **Work IQ MCP tools**, while the Agent 365 control plane keeps every action observable and governed.

The critical governance point in this exercise is that Copilot Studio **automatically creates a Microsoft Entra Agent ID for every new agent**. That means the agent you build here is not an ungoverned side project; it inherits the identity model from Exercise 1 and falls under the Conditional Access policies you wrote in Exercise 2 without any extra work. Connector permissions appear as API permissions on that agent identity, so you can see and target them from Microsoft Entra.

>**Note:** Work IQ MCP is a **preview** feature and requires a Microsoft 365 Copilot license, which is pre-staged in your lab. If Work IQ MCP tools are unavailable in your lab tenant's region, substitute any standard Copilot Studio connector tool. The publishing, approval, observability, and lifecycle tasks are unaffected.

In this exercise you will:

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

Copilot Studio is where you define what the agent does and says. This is low-code configuration, not programming.

1. On the lab virtual machine, open a new browser tab and navigate to Microsoft Copilot Studio:

    ```
    https://copilotstudio.microsoft.com/
    ```

1. If prompted, sign in with the following credentials:

   - **Email/Username:** <inject key="AzureAdUserEmail"></inject>
   - **Password:** <inject key="AzureAdUserPassword"></inject>

1. Confirm the correct environment is selected in the top-right environment picker. If multiple environments are listed, select the environment provided for this lab.

    ![Copilot Studio home page with the environment picker in the top right](./media/a365-ex4-t1-01.png)

    >**Note:** Copilot Studio agents live inside a Power Platform environment. If you create an agent in the wrong environment, it will not appear where you expect it later, and admin approval routes differently.

1. Select **Create**, then select **New agent**.

1. Enter the following details for your agent:

   - **Name:** **Finley Expense Assistant <inject key="DeploymentID" enableCopy="false"/>**
   - **Description:** **Helps Contoso employees check expense policy and follow up on expense report status.**

1. In the **Instructions** field, enter the following system prompt:

    ```
    You are Finley, Contoso's expense assistant. Help employees understand expense policy and follow up on the status of submitted expense reports. Be concise and factual. If you are unsure about a policy detail, say so rather than guessing.
    ```

    ![New agent configuration page with name, description, and instructions filled in](./media/a365-ex4-t1-02.png)

    >**Note:** The instructions become the agent's system prompt, which defines its default behaviour, persona, and operating boundaries. This value is also recorded in the `Instructions` column of the Defender `AgentsInfo` table, which is how a security analyst can later review what an agent was told to do.

1. Select **Create**.

1. Wait for the agent to be created. When the authoring canvas opens, your agent is ready to configure.

    ![Copilot Studio authoring canvas for the newly created agent](./media/a365-ex4-t1-03.png)

### Task 2: Add a Work IQ MCP tool and test the agent

Right now Finley can talk about expenses but cannot act. In this task you give it a real capability by connecting a Work IQ MCP tool, then prove it works.

1. In your agent, select the **Tools** tab, then select **Add tool**.

    ![Agent Tools tab with the Add tool button](./media/a365-ex4-t2-01.png)

1. On the **Add tool** page, select **Model Context Protocol** to filter the catalog to MCP servers.

    ![Add tool page with Model Context Protocol selected](./media/a365-ex4-t2-02.png)

    >**Note:** This catalog contains the Work IQ MCP servers plus other MCP servers available in your tenant. Highlights include Work IQ Mail, Calendar, Teams, SharePoint, OneDrive, Word, User, and Windows 365 agents. Each server exposes granular, auditable tools rather than broad access.

1. In the search box, type **mail**.

1. Select **Work IQ Mail** from the results.

    ![Search results showing the Work IQ Mail MCP server](./media/a365-ex4-t2-03.png)

1. Expand the **connection** dropdown and select **Create new connection**.

1. Select **Create**.

1. When prompted, provide your credentials and complete the sign-in process:

   - **Email/Username:** <inject key="AzureAdUserEmail"></inject>
   - **Password:** <inject key="AzureAdUserPassword"></inject>

    ![Sign-in dialog for the Work IQ Mail MCP server connection](./media/a365-ex4-t2-04.png)

    >**Note:** Each MCP server corresponds to a permission on the Agent 365 application. The agent only gains access to the server after this consent is granted, which is what makes every tool call authorized rather than implicit.

1. Confirm **Work IQ Mail** now appears in your agent's tools list.

    ![Agent tools list showing Work IQ Mail added](./media/a365-ex4-t2-05.png)

1. Now test the tool. Select **Test** to open the test pane.

1. Enter the following prompt, replacing the placeholder with your own lab email address:

    ```
    Send an email to <inject key="AzureAdUserEmail"></inject> with the subject "Finley test" and ask how the hands-on lab is going.
    ```

1. Press **Enter**.

1. When asked to allow the Work IQ tool to connect and use services, select **Allow**.

    ![Test pane showing the permission prompt to allow the Work IQ tool](./media/a365-ex4-t2-06.png)

1. After a few moments, check your mailbox and confirm you received the email.

    ![Outlook inbox showing the test email sent by the agent](./media/a365-ex4-t2-07.png)

    >**Note:** This is the moment Finley stops being a chatbot and becomes an agent. It did not describe sending an email; it called a governed tool that actually sent one. Every such call is traced, which you will verify in Task 6.

1. Select **Publish** in the top menu bar to publish your agent for the first time.

    >**Note:** You must publish the agent at least once before it can be connected to Teams. Publishing makes the current version available to channels.

### Task 3: Verify the agent's Entra Agent ID and connector permissions

This is the task that ties this exercise back to the first two. Copilot Studio created an Entra Agent ID for your agent automatically, and that identity is subject to the policies you already wrote.

1. In Copilot Studio, open your agent and select **Settings**.

1. Select **Advanced**.

1. Expand the **Metadata** section and locate the GUID shown under **Entra Agent ID**. Copy this value.

    ![Copilot Studio agent settings showing the Entra Agent ID under Advanced Metadata](./media/a365-ex4-t3-01.png)

    >**Note:** Starting July 2026, all new agents must have Microsoft Entra Agent IDs and you can no longer opt out of automatic creation. Agents created before that rollout continue to use app registrations and will be migrated over time. Governance capabilities work for both during the transition.

1. Switch to the Microsoft Entra admin center tab, or navigate to:

    ```
    https://entra.microsoft.com
    ```

1. Navigate to **Entra ID** > **Agents** > **Agent identities**.

1. Search for the Entra Agent ID value you copied, and select the matching agent identity.

    ![Agent identities page with the Copilot Studio agent identity located](./media/a365-ex4-t3-02.png)

1. Review the agent identity's **API permissions**. Note that the Power Platform connector permissions the agent uses appear here as first-class API permissions.

    ![Agent identity API permissions showing connector scopes](./media/a365-ex4-t3-03.png)

    >**Note:** This visibility is deliberate. Before it, an Entra or Microsoft 365 admin had to open the Power Platform admin center to discover which connectors an agent could call. Now those scopes are visible in Entra and, crucially, can be targeted by Conditional Access policies for network location, device compliance, or risk conditions.

1. Also note the **blueprint** this identity belongs to. All Copilot Studio agent identities are children of a single tenant-wide blueprint named **Microsoft Copilot Studio agent identity blueprint**.

    >**Note:** Because every Copilot Studio agent shares one blueprint, applying a Conditional Access policy at the blueprint level covers every Copilot Studio agent in the tenant at once, including agents that do not exist yet. This is the same scaling principle as the attribute-driven policy you built in Exercise 2.

1. Optionally, assign the **AgentApprovalStatus** custom security attribute to this identity with the value **Finance_Approved**, following the same steps as Exercise 2, Task 3. This brings the agent into scope of your attribute-driven policy.

### Task 4: Publish the agent to Teams and Microsoft 365 Copilot

Now you give Finley a place to work. Publishing happens in two stages: first you install it for yourself as a trial, then you submit it for admin approval so the whole organization can find it.

1. Return to the Microsoft Copilot Studio tab and open your agent.

1. On the top menu bar, select **Channels**.

1. Select the **Teams and Microsoft 365 Copilot** tile to open its configuration panel.

    ![Channels page with the Teams and Microsoft 365 Copilot tile](./media/a365-ex4-t4-01.png)

1. Under **Turn on Microsoft 365**, ensure **Make agent available in Microsoft 365 Copilot** is selected.

    >**Note:** If you clear this option, the agent is only available in Teams and not in Microsoft 365 Copilot.

1. Select **Add channel**.

    ![Teams and Microsoft 365 Copilot configuration panel with Add channel selected](./media/a365-ex4-t4-02.png)

1. Select **Edit details** and provide the information users will see in the app store:

   - Confirm the agent **icon**, **colour**, and **short description**
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

1. Review the submission requirements, then select **Submit for admin approval**.

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

1. On the **Agent overview** dashboard, locate the **Pending requests for agents** card and select **Manage requests**.

    ![Agent overview dashboard with the Pending requests for agents card](./media/a365-ex4-t5-01.png)

1. You are taken to **All agents** > **Requests**. Locate your agent **Finley Expense Assistant <inject key="DeploymentID" enableCopy="false"/>** in the list.

    ![Requests tab showing the pending agent submission](./media/a365-ex4-t5-02.png)

1. Select the agent to review its details, including the publisher, the platform it was built on, and the permissions it is requesting.

    >**Note:** This review step is the governance control. Read the requested permissions and ask whether each one is necessary for the agent's stated purpose. An expense assistant that requests broad mail write access deserves a question.

1. Select **Publish to store** to make the agent available to members of your organization.

    ![Agent request details pane with the Publish to store action](./media/a365-ex4-t5-03.png)

    >**Note:** The alternative action is **Reject submission**, which prevents the agent from becoming available. Both actions require the **AI Administrator** or **Global Administrator** role.

1. Now deploy the agent to users. Navigate to **Agents** > **All agents** and ensure the **Registry** tab is selected.

1. Select the **Status** filter and choose **Available**, then locate and select your agent.

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

Governance without observability is a promise you cannot verify. In this task you query the actual record of what Finley is and what it can do.

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
    | project Timestamp, AgentName, Platform, EntraAgentId, PublishedStatus, LifecycleStatus, Owners
    | order by Timestamp desc
    ```

    ![Advanced hunting results showing agents from the AgentsInfo table](./media/a365-ex4-t6-02.png)

    >**Note:** Use the **`AgentsInfo`** table. The older `AIAgentsInfo` table was **retired on 1 July 2026**, and any query written against it now fails. If you find older documentation or scripts referencing `AIAgentsInfo`, they need migrating.

1. Narrow the query to your own agent:

    ```
    AgentsInfo
    | where AgentName contains "Finley"
    | project Timestamp, AgentName, Platform, EntraAgentId, EntraBlueprintId, LifecycleStatus, Availability, Owners
    | order by Timestamp desc
    ```

    >**Note:** If the query returns no rows, advanced hunting has **ingestion latency**; the data has not arrived yet. Wait a few minutes and run the query again. An empty first result is normal and is not a sign that anything is broken.

1. Now inspect what tools and MCP servers the agent has been given. Run the following query:

    ```
    AgentsInfo
    | where AgentName contains "Finley"
    | project Timestamp, AgentName, McpServers, DeclaredTools, DeclaredDataSources, Permissions
    ```

    ![Advanced hunting results showing MCP servers and declared tools for the agent](./media/a365-ex4-t6-03.png)

    >**Note:** The `McpServers` column shows the MCP servers connected to the agent, including server URLs and credential configuration. This is how a security analyst answers "what can this agent actually reach?" without opening Copilot Studio.

1. Review the agent's system prompt and model, which reveal what the agent was instructed to do:

    ```
    AgentsInfo
    | where AgentName contains "Finley"
    | project AgentName, Model, Instructions, Channels, Capabilities, Guardrails
    ```

1. Finally, run a tenant-wide posture query that would be useful in a real environment, summarising agents by platform and lifecycle state:

    ```
    AgentsInfo
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

1. **Suspend the agent.** In the agent details pane, immediately under the agent's name, select **Block**.

1. In the **Block agent** pane, select **Block agent**, then select **Save**.

    ![Block agent pane with the Block agent option selected](./media/a365-ex4-t7-01.png)

    >**Note:** Blocking restricts access to the agent across the organization so no user can use it. It is reversible and is the correct first action when an agent is behaving unexpectedly, because it stops the behaviour without destroying evidence.

1. Verify the agent's status now shows as blocked in the registry.

1. **Restore the agent.** Select the agent again and select **Unblock**, then confirm.

1. **Reassign ownership.** With the agent selected, in the details pane select **Assign new owner**.

1. In the **Assign a new owner** pane, enter a user from your organization and select **Assign**.

    ![Assign a new owner pane](./media/a365-ex4-t7-02.png)

    >**Note:** After reassignment, the new owner gets full edit and delete permissions plus access to any files the previous owner uploaded, and the previous owner loses all access including read rights. This action is supported for Agent Builder and Copilot Studio agents.

1. **Uninstall the agent.** Select the agent, then select **Uninstall**.

1. In the **Remove agent** pane, select the **Remove agent** option, then select the **Uninstall Agent** button.

    ![Remove agent pane with the Uninstall Agent button](./media/a365-ex4-t7-03.png)

    >**Note:** Uninstalling affects the agent's availability and functionality in Copilot and in host products such as Outlook and Teams. It is distinct from **Delete**, which permanently removes the agent and all associated files, including the underlying SharePoint Embedded container. Deletion is irreversible and can take up to 24 hours to reach all users.

1. Return to **Agents** > **Overview** and confirm the governance cards reflect your changes.

1. Optionally, re-run the advanced hunting query from Task 6 to observe the `LifecycleStatus` column change to reflect the actions you performed:

    ```
    AgentsInfo
    | where AgentName contains "Finley"
    | project Timestamp, AgentName, LifecycleStatus, Availability
    | order by Timestamp desc
    ```

    >**Note:** Remember ingestion latency. The lifecycle change may take a few minutes to appear in advanced hunting.

## Conclusion

In this exercise you completed the agent lifecycle. You built Finley in Copilot Studio, grounded it in Microsoft IQ through a Work IQ MCP tool and proved the grounding worked by having it send a real email, confirmed that Copilot Studio had automatically issued it an Entra Agent ID subject to the policies you wrote in Exercise 2, published it to Teams through an approval flow that required an administrator to review its permissions, audited it in Defender advanced hunting, and finally suspended, reassigned, and removed it.

The thread running through all four exercises is that governance is not a gate you add at the end. Finley was governable in Exercise 4 because it had an identity in Exercise 1 and policies in Exercise 2. Had you built the agent first and tried to govern it afterwards, you would have been retrofitting controls onto something already in production, which is exactly the situation Contoso was in when this lab started.

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

You have now governed a single agent across identity, policy, compute, intelligence, and lifecycle, which is the complete Agent 365 control plane.

### You have successfully completed the lab. Click on **Next** to proceed.

![Next button in the lower right corner of the lab guide](./media/a365-gs-07.png)
