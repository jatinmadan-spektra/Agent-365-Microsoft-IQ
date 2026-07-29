# Exercise 2: Agent 365 Policy Conformance

### Estimated Duration: 60 minutes

Finley now exists as an identity, but nothing constrains it. In this exercise you put guardrails around it: you review the protections Microsoft ships by default, configure tenant-wide agent settings, and build two Conditional Access policies that decide what Finley can reach and under what conditions.

## Overview

**Agent 365 policy templates** bundle predefined governance and security policies drawn from four different Microsoft services, so you get consistent protection without configuring each service separately:

- **Microsoft Entra** provides identity protection, lifecycle management, and network visibility
- **Microsoft Purview** provides audit logging, sensitive information detection through DSPM for AI, and compliance assessment
- **SharePoint Online** provides access control and content permission insights
- **Microsoft Defender** provides real-time protection and advanced hunting

You will review the default template rather than build one, because Microsoft's built-in policies are locked and cannot be edited, and custom policies inside an Agent 365 template are not yet supported. Instead, you will build the equivalent controls **natively in Microsoft Entra**, where they are generally available today. This gives you the same enforcement with a fully supported path.

Two Conditional Access patterns matter for agents:

- **Attribute-driven access**: You tag agent identities with **custom security attributes**, then write a policy that targets those tags. As the number of agents grows, managing each one individually in every policy becomes unsustainable. Tagging scales; individual selection does not.
- **Risk-based access**: You block agents that Microsoft Entra ID Protection flags as high risk, the same way you would block a risky user sign-in.

>**Note:** Every Conditional Access policy in this exercise is created in **Report-only** mode first. Report-only mode evaluates the policy and logs what *would* have happened without actually enforcing it. This is how you validate a policy safely before it can lock out a legitimate agent.

In this exercise you will:

- Review the Microsoft default policy template and map its protections to their source services
- Configure tenant guardrails under Agent settings
- Create custom security attributes and assign them to Finley
- Build an attribute-driven Conditional Access policy allowing only approved agents
- Build a risk-based Conditional Access policy blocking high-risk agents
- Review conformance and risk signals in the registry

## Objectives

- **Task 1**: Review the Microsoft default policy template
- **Task 2**: Configure tenant guardrails in Agent settings
- **Task 3**: Create and assign custom security attributes
- **Task 4**: Create an attribute-driven Conditional Access policy
- **Task 5**: Create a Conditional Access policy for high-risk agents
- **Task 6**: Review conformance and risk signals in the registry

### Task 1: Review the Microsoft default policy template

Before writing your own rules, understand what is already enforced. Every tenant has default policies for all agents. Some are enabled automatically; others require additional configuration based on your organization's setup.

1. In the Microsoft 365 admin center, in the left navigation pane, expand **Agents (1)** and select **Settings (2)**.

    ![Microsoft 365 admin center with Agents expanded and Settings selected](./media/a365-ex2-t1-01.png)

1. Review the configuration options available on the **Agent settings** page:

   - **Agent management rules** - run bulk governance actions across many agents at once
   - **Allowed agent types** - control which categories of agent users can install
   - **Security templates** - preset policies, rules, and allow lists for new agents
   - **Sharing** - control who can share agents and how
   - **User access** - control which users can interact with agents

    ![Agent settings page showing the five configuration areas](./media/a365-ex2-t1-02.png)

1. Select **Templates** to view the policy templates available in your tenant.

1. Select the **Default policy template for all agents except AI teammates** to open it.

    ![Templates page showing the available default policy templates](./media/a365-ex2-t1-03.png)

1. Review the bundled policies. Use the table below to map each policy to the service that enforces it, and note what problem each one solves.

    | Policy name | Source service | What it protects against |
    | --- | --- | --- |
    | Purview audit enabled | Microsoft Purview | No record of what the agent did |
    | Detect sensitive information (DSPM) in AI interaction | Microsoft Purview | Sensitive data leaking through agent prompts and responses |
    | Purview AI compliance assessment | Microsoft Purview | Undetected compliance drift over time |
    | Identity protection | Microsoft Entra | Compromised or anomalous agent identities |
    | Network visibility | Microsoft Entra | Unmonitored agent access to external resources |
    | Lifecycle management for agents | Microsoft Entra | Orphaned agents accumulating after owners leave |
    | Agent access insights | SharePoint Online | Not knowing which sites agents are reading |
    | Restrict external sharing of sites and its content | SharePoint Online | Agents discovering content they should not see |
    | Access control for sites and OneDrive | SharePoint Online | Overbroad agent access to user files |
    | Content permissions insights | SharePoint Online | Oversharing invisible to admins |
    | AI real time protection and investigation | Microsoft Defender | Malicious agent behaviour at runtime |
    | Advanced hunting | Microsoft Defender | Inability to investigate a suspected incident |

    ![Default policy template detail showing the list of bundled policies](./media/a365-ex2-t1-04.png)

    >**Note:** Microsoft's built-in default policies appear **locked** and cannot be edited. This is by design: they are the enforced baseline. Also note that policy templates apply only at **new agent activation**. You cannot retroactively apply a template to an already-approved agent.

1. Close the template pane without making changes.

### Task 2: Configure tenant guardrails in Agent settings

Policy templates govern individual agents. Agent settings govern the whole tenant: which kinds of agents are allowed to exist at all, who can share them, and who can use them. These are the broadest controls available and the fastest way to reduce your exposure.

1. On the **Agent settings** page, select **Allowed agent types**.

1. Review the three options and note which are enabled:

   - **Allow apps and agents built by Microsoft**
   - **Allow apps and agents built by your organization**
   - **Allow apps and agents built by external publishers**

    ![Allowed agent types page with the three publisher categories](./media/a365-ex2-t2-01.png)

1. Ensure **Allow apps and agents built by your organization** is enabled, because Finley is an organization-built agent and you will publish it in Exercise 4. Select **Save** if you made a change.

    >**Note:** If you disable a category, agents of that type no longer appear for users in the Agent Store. Agents built by Microsoft remain visible even when disabled, but users cannot install them.

1. Return to **Agent settings** and select **Sharing**.

1. Review the sharing options and set sharing to **Specific users**, then select the group or user permitted to share agents. Select **Save**.

    ![Sharing settings page with the Specific users option selected](./media/a365-ex2-t2-02.png)

    >**Note:** Only agents built with **Microsoft 365 Copilot Agent Builder** are governed by this sharing control. This is a common source of confusion: restricting sharing here does not restrict every agent platform.

1. Return to **Agent settings** and select **User access**.

1. Review the three options and leave **All users** selected for this lab, so you can test Finley in Exercise 4. Select **Save**.

    ![User access settings page showing the access scope options](./media/a365-ex2-t2-03.png)

1. Return to **Agent settings** and select **Agent management rules**.

1. Review the supported rule-based bulk actions. These let you apply governance across many agents at once instead of one at a time:

   - **Install Microsoft agents** - proactively deploy Microsoft first-party agents tenant-wide
   - **Reassign ownerless agents created with Agent Builder to manager** - automatically transfer ownership of orphaned agents to the previous owner's manager, based on the Microsoft Entra ID hierarchy

    ![Agent management rules page showing the available bulk governance rules](./media/a365-ex2-t2-04.png)

    >**Note:** The ownerless-agent rule directly addresses one of the governance gaps surfaced on the Agent overview dashboard. Agents become ownerless when their creator leaves the organization, and without a rule an admin has to find and fix each one manually.

### Task 3: Create and assign custom security attributes

You cannot write "only approved agents may access Finance resources" until you have a way to record which agents are approved. Custom security attributes are that mechanism: business-specific key-value pairs you attach to directory objects and then target in policy.

>**Note:** By default, Global Administrator does **not** have permission to define or assign custom security attributes. Your lab account has been granted the **Attribute Definition Administrator** and **Attribute Assignment Administrator** roles for this reason.

1. Switch to the Microsoft Entra admin center tab, or navigate to:

    ```
    https://entra.microsoft.com
    ```

1. In the left navigation pane, expand **Entra ID (1)** and select **Custom security attributes (2)**.

    ![Microsoft Entra admin center with Custom security attributes selected](./media/a365-ex2-t3-01.png)

1. Select **Add attribute set**.

    >**Note:** If **Add attribute set** is greyed out, you are missing the Attribute Definition Administrator role. Sign out and sign back in to refresh your role assignments.

1. Enter the following values, then select **Add**:

   - **Attribute set name:** **AgentAttributes**
   - **Description:** **Attributes used to classify AI agents**
   - **Maximum number of attributes:** **25**

    ![New attribute set pane with AgentAttributes entered](./media/a365-ex2-t3-02.png)

    >**Note:** Attribute set names can be up to 32 characters with no spaces or special characters, and **cannot be renamed or deleted** later. Only deactivation is possible, so name carefully.

1. Select the **AgentAttributes** attribute set you just created to open it.

1. Select **Add attribute** and enter the following:

   - **Attribute name:** **AgentApprovalStatus**
   - **Description:** **Review and approval state of the agent**
   - **Data type:** **String**
   - **Allow multiple values to be assigned:** **Yes**
   - **Only allow predefined values to be assigned:** **Yes**

    ![New attribute pane with AgentApprovalStatus configured](./media/a365-ex2-t3-03.png)

1. Because you selected **Yes** for predefined values, select **Add value** and add each of the following values, leaving **Is active?** set to **Yes** for each:

   - **New**
   - **In_Review**
   - **HR_Approved**
   - **Finance_Approved**
   - **IT_Approved**

    ![Add predefined value pane showing the approval status values](./media/a365-ex2-t3-04.png)

1. Select **Save**.

1. Now assign the attribute to Finley. In the left navigation pane, navigate to **Entra ID** > **Agents** > **Agent identities**.

1. Select the agent identity you created in Exercise 1.

1. In the left navigation pane of the agent identity, select **Custom security attributes**.

1. Select **Add assignment**, then set:

   - **Attribute set:** **AgentAttributes**
   - **Attribute name:** **AgentApprovalStatus**
   - **Assigned value:** **Finance_Approved**

    ![Add assignment pane assigning Finance_Approved to the agent identity](./media/a365-ex2-t3-05.png)

1. Select **Save**.

Finley is now tagged as an approved Finance agent. In the next task you write the policy that acts on that tag.

### Task 4: Create an attribute-driven Conditional Access policy

Now you build the rule that uses the tag. The policy blocks all agent identities *except* those carrying the `Finance_Approved` value, which means any new untagged agent that appears in the tenant is denied access by default.

1. In the Microsoft Entra admin center, in the left navigation pane, expand **Entra ID (1)**, select **Conditional Access (2)**, and then select **Policies (3)**.

    ![Microsoft Entra admin center with Conditional Access Policies selected](./media/a365-ex2-t4-01.png)

1. Select **+ New policy**.

1. In the **Name** field, enter:

    ```
    CA-Agents-BlockUnapprovedAgents
    ```

    >**Note:** Adopt a consistent naming standard for Conditional Access policies. A name that encodes scope and intent saves considerable time during incident response.

1. Under **Assignments**, select **Users, agents or workload identities**.

1. Under **What does this policy apply to?**, select **Agents**.

    ![Assignments pane with Agents selected as the policy target](./media/a365-ex2-t4-02.png)

1. Under **Include**, select **All agent identities**.

1. Select the **Exclude** tab, then select **Select agent identities based on attributes**.

1. Configure the attribute filter:

   - Set **Configure** to **Yes**
   - **Attribute:** **AgentApprovalStatus**
   - **Operator:** **Contains**
   - **Value:** **Finance_Approved**

    ![Exclude tab showing the attribute filter for Finance_Approved](./media/a365-ex2-t4-03.png)

1. Select **Done**.

1. Under **Target resources**, select **Include**, then select **All resources (formerly 'All cloud apps')**.

    ![Target resources pane with All resources selected](./media/a365-ex2-t4-04.png)

1. Under **Access controls** > **Grant**, select **Block access**, then select **Select**.

    ![Grant control pane with Block access selected](./media/a365-ex2-t4-05.png)

1. At the bottom of the page, set **Enable policy** to **Report-only**.

    ![Enable policy toggle set to Report-only](./media/a365-ex2-t4-06.png)

    >**Note:** Do not set this policy to **On**. A misconfigured block policy targeting all agent identities could immediately lock out every agent in the tenant, including Finley. Report-only mode gives you the evaluation data without the outage.

1. Select **Create**.

1. To see what the policy would do, select your new policy from the list and review the **policy impact** information. This shows which identities would have been affected had the policy been enforced.

    ![Conditional Access policy overview showing report-only impact data](./media/a365-ex2-t4-07.png)

    >**Note:** In production you would leave a policy in report-only mode for several days, confirm the affected set matches your expectation, and only then move the **Enable policy** toggle from **Report-only** to **On**.

### Task 5: Create a Conditional Access policy for high-risk agents

The previous policy answers "is this agent approved?". This one answers a different question: "has this agent started behaving suspiciously?". Microsoft Entra ID Protection generates risk signals for agent identities, and Conditional Access can act on them.

1. Still in **Entra ID** > **Conditional Access** > **Policies**, select **+ New policy**.

1. In the **Name** field, enter:

    ```
    CA-Agents-BlockHighRiskAgents
    ```

1. Under **Assignments**, select **Users, agents or workload identities**.

1. Under **What does this policy apply to?**, select **Agents**, then under **Include** select **All agent identities**.

1. Under **Target resources** > **Include**, select **All resources (formerly 'All cloud apps')**.

1. Under **Conditions**, select **Agent risk (Preview)** and set **Configure** to **Yes**.

1. Under **Configure agent risk levels needed for policy to be enforced**, select **High**.

    ![Agent risk condition configured for High risk level](./media/a365-ex2-t5-01.png)

    >**Note:** Microsoft recommends starting with **High** only. Including **Medium** increases the chance of blocking legitimate agent activity. Tune this per your organization's risk tolerance.

1. Under **Access controls** > **Grant**, select **Block access**, then select **Select**.

    >**Note:** For agents authenticating with their own identity there is no remediation path such as multifactor authentication, because no human is present to respond to a prompt. **Block** is the only meaningful control.

1. Set **Enable policy** to **Report-only**, then select **Create**.

1. Select your new policy and review its impact data.

    ![Conditional Access policies list showing both new report-only agent policies](./media/a365-ex2-t5-02.png)

    >**Note:** In a freshly provisioned lab tenant, Microsoft Entra ID Protection has not yet observed enough activity to flag any agent as high risk, so you will most likely see zero affected identities. This is expected and is not a failure. The learning objective here is the policy construction and the safe validation workflow, not observing a live detection. In production, risk detections accumulate as agents operate.

### Task 6: Review conformance and risk signals in the registry

Finally, close the loop. The Microsoft 365 admin center aggregates signals from Entra, Defender, and Purview into a single view, so an administrator does not have to check three portals to find out whether an agent is a problem.

1. Switch to the Microsoft 365 admin center tab.

1. Navigate to **Agents (1)** > **All agents (2)** and ensure the **Registry (3)** tab is selected.

1. Locate the **Risks** column in the agent list. This column aggregates high-severity risks across Microsoft Entra, Microsoft Defender, and Microsoft Purview.

    ![All agents page with the Risks column visible in the registry list](./media/a365-ex2-t6-01.png)

    >**Note:** The Risks column exists to close a specific visibility gap. Before it, an administrator governing agents had to correlate findings across three separate security portals manually.

1. Select your agent **finley-<inject key="DeploymentID" enableCopy="false"/>** to open the details pane, and review the **Permissions** section to confirm what the agent has been granted.

1. Navigate to **Agents** > **Overview** and review the governance action cards:

   - **Pending requests for agents**
   - **Agents at risk**
   - **Agents without owners**
   - **Agents with exceptions**

    ![Agent overview dashboard showing the four governance action cards](./media/a365-ex2-t6-02.png)

1. Confirm that **Agents without owners** does not include your agent, because you assigned an owner in Exercise 1.

    >**Note:** Governance actions such as approving agent requests or assigning ownership can only be performed by users in the **AI Administrator** or **Global Administrator** roles. Other roles can view governance gaps but cannot remediate them.

## Conclusion

In this exercise you moved Finley from "exists" to "constrained". You reviewed the twelve protections Microsoft enforces by default and traced each back to the service that implements it, tightened tenant-wide guardrails through Agent settings, and built the two Conditional Access patterns that matter most for agents at scale: attribute-driven approval and risk-based blocking.

The most transferable idea here is the attribute pattern. Selecting individual agents in a policy works when you have three agents and collapses when you have three hundred. Tagging identities and targeting the tag means policies automatically cover agents that do not exist yet, which is the only approach that survives growth.

## Review

In this exercise, you have completed the following:

- Reviewed the Microsoft default policy template
- Configured tenant guardrails in Agent settings
- Created and assigned custom security attributes
- Created an attribute-driven Conditional Access policy
- Created a Conditional Access policy for high-risk agents
- Reviewed conformance and risk signals in the registry

## Summary

In this exercise, you reviewed the Agent 365 default policy template and mapped its protections to Entra, Purview, SharePoint, and Defender, configured tenant-wide agent guardrails, and built attribute-driven and risk-based Conditional Access policies for agents, validating both safely in report-only mode. These policies govern the same agent identity you provisioned in Exercise 1 and will continue to apply to it in Exercises 3 and 4.

Click **Next** from the lower right corner to move on to the next page.

![Next button in the lower right corner of the lab guide](./media/a365-gs-07.png)
