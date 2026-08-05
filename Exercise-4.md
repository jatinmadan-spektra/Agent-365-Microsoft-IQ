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

1. Under **Turn on Microsoft 365**, ensure **Make agent available in Microsoft 365 Copilot (1)** is selected.

1. Select **Save and publish (2)**.

    ![](./media/ex4-14.png)

1. Select **Edit details** and provide the information users will see in the app store:

   - Confirm the agent **short description** and **Long Description (1)**
   
   - Select **More** and add a **Developer name (2)**.

1. Select **Save and publish (3)**.

    ![](./media/ex4-15.png)

    ![](./media/ex4-16.png)

1. After publishing click back, now install the agent for yourself. In the configuration panel, select **See agent in Teams**.

    ![](./media/ex4-17.png)

    >Note: If prompted to open Teams App, select **open the web App instead**.

1. In the Teams dialog that opens, select **Add**.

    ![](./media/ex4-18.png)

1. Added successfully message appears, now,click open to send it a test message to verify it responds.

    ![](./media/ex4-20.png)

    ![](./media/ex4-19.png)

1. Now make it available to the organization. Return to the **Teams and Microsoft 365 Copilot** configuration panel in Copilot Studio and select **Availability options**.
  
    ![](./media/ex4-21.png) 

1. Select **Submit to org catalog**.
    
    ![](./media/ex4-22.png)

1. Review the submission requirements, then select **Submit to org catalog**.

    ![](./media/ex4-23.png)

1. At the confirmation prompt, select **Yes,submit**.

    ![](./media/ex4-24.png)

### Task 5: Approve the agent in the Microsoft 365 admin center

You now change hats. You submitted the agent as a maker; you approve it as the administrator. This separation is the whole point of governed deployment: nothing reaches the organization without an IT review of what it is asking for.

1. Switch to the Microsoft 365 admin center tab, or navigate to:

    ```
    https://admin.microsoft.com/
    ```

1. In the left navigation pane, expand **Agents (1)** and select **Overview (2)**.

1. On the **Agent overview** dashboard, locate the **Pending requests** card and select the **manage requests (3)**.

    ![](./media/ex4-25.png)

1. On **All agents** > **Requests**, locate your agent **Finley Expense Assistant** in the list.

    ![](./media/ex4-26.png)

1. Select the agent to review its details.

1. Select **Publish to store** to make the agent available to members of your organization.

    ![](./media/ex4-27.png)

1. In Publish agent to selected users screen, select users or groups who can install the agent. For this lab, select **specific users/groups** and choose your own account.

    ![](./media/ex4-28.png)

1. For select users or groups who will have the agent pre-installed. For this lab, select **specific users/groups**, choose your own account, and click **Next**.

1. Click **Next** on both the Apply template pane and Accept permissions keeping the settings as default.

1. On Review and finish pane, review the details and select **Publish**.

    ![](./media/ex4-30.png)

1. Once published, click Done.

1. Verify the agent now appears with an Available status in the registry.

    ![](./media/ex4-31.png)

### Task 6: Trace agent activity in Microsoft Defender advanced hunting

Governance without observability is a promise you cannot verify. In this task you query the actual record of what your agent is and what it can do.

1. Open a new browser tab and navigate to the Microsoft Defender portal:

    ```
    https://security.microsoft.com/
    ```

1. If prompted, sign in with the following credentials:

   - **Email/Username:** <inject key="AzureAdUserEmail"></inject>

   - **Temporary Access Pass:** <inject key="AzureAdUserPassword"></inject>

1. In the left navigation pane, expand **Investigation & response (1)**, then expand **Hunting (2)** and select **Advanced hunting (3)**.

    ![](./media/ex4-32.png)

1. In the query editor, enter the following KQL query to list the agents in your tenant, then select **Run query**:

    ```
    AgentsInfo
    | summarize arg_max(Timestamp, *) by AgentId
    | project Timestamp, Name, Platform, PublishedStatus, LifecycleStatus, Owners
    | order by Timestamp desc
    ```

    ![](./media/ex4-33.png)

1. Narrow the query to your own agent:

    ```
    AgentsInfo
    | where Name contains "Finley"
    | summarize arg_max(Timestamp, *) by AgentId
    | project Timestamp, Name, Platform, LifecycleStatus, Availability, Owners
    ```
    
    ![](./media/ex4-34.png)

1. Now inspect what tools and MCP servers the agent has been given. Run the following query:

    ```
    AgentsInfo
    | where Name contains "Finley"
    | summarize arg_max(Timestamp, *) by AgentId
    | project Name, McpServers, DeclaredTools, DeclaredDataSources, Permissions
    ```

    ![](./media/ex4-35.png)

1. Review the agent's system prompt and model, which reveal what the agent was instructed to do:

    ```
    AgentsInfo
    | where Name contains "Finley"
    | summarize arg_max(Timestamp, *) by AgentId
    | project Name, Model, Instructions, Channels, Capabilities, Guardrails
    ```

1. Optionally, inspect the agent's runtime surface and multi-agent connections:

    ```
    AgentsInfo
    | where Name contains "Finley"
    | summarize arg_max(Timestamp, *) by AgentId
    | project Name, Endpoints, ConnectedAgents, Triggers, Memory, InstanceCount
    ```
1. Finally, run a tenant-wide posture query that would be useful in a real environment, summarising agents by platform and lifecycle state:

    ```
    AgentsInfo
    | summarize arg_max(Timestamp, *) by AgentId
    | summarize AgentCount = count() by Platform, LifecycleStatus
    | order by AgentCount desc
    ```

    ![](./media/ex4-36.png)

### Task 7: Exercise lifecycle governance actions

Finally, prove you can control the agent after deployment. An agent you cannot suspend, reassign, or remove is not governed.

1. Switch to the Microsoft 365 admin center tab.

1. Navigate to **Agents (1)** > **All agents (2)** and ensure the **Registry** tab is selected.

1. Locate and select your agent **Finley Expense Assistant (3)**.

    ![](./media/ex4-37.png)

1. **Suspend the agent.**  On the top right, select **Block (1)**, then on Block agent screen, select **Block agent (2)** checkbox, and click **Save (3)**.

    ![](./media/ex4-38.png)

    ![](./media/ex4-39.png)

1. Verify the agent's status now shows as blocked in the registry.
   
   ![](./media/ex4-40.png)

   >Note: If you see an error, go back and select the correct agent with the Uninstall option on the left side of the block icon as shown in the previous image.
         
1. **Restore the agent.** Select **Unblock (1)**, then on Unblock agent screen, select **Unblock agent (2)** checkbox, and click **Save (3)**.

    ![](./media/ex4-41.png)

    ![](./media/ex4-42.png)

1. **Uninstall the agent.** Select **Uninstall (1)**, then on Remove agent screen, select **Remove agent (2)** checkbox, and click **Uninstall Agent (3)**.

    ![](./media/ex4-43.png)

    ![](./media/ex4-44.png)

1. Re-run the advanced hunting query from Task 6 to observe the `LifecycleStatus` column change:

    ```
    AgentsInfo
    | where Name contains "Finley"
    | summarize arg_max(Timestamp, *) by AgentId
    | project Timestamp, Name, LifecycleStatus, Availability
    ```

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

### You have successfully completed the lab. 
