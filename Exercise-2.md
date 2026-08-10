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

>**Note:** This exercise targets **agent identities**, which is the correct scope for Finley. Conditional Access also supports **agent users (Preview)**, a different object type used by agents that have their own mailbox and user account. Controls such as device compliance and compliant network are available **only** for agent users, not for agent identities. Finley has an agent identity and no agent user, which is why every policy here uses **Block** as its control.

In this exercise you will:

- Review the Microsoft default policy template and map its protections to their source services
- Configure tenant guardrails under Agent settings
- Grant yourself the attribute administrator roles, then create and assign custom security attributes
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

    ![](./media/a365-ex2-t1-01.png)

1. Review the configuration options available on the **Agent settings** page:

   - **Agent management rules** - set and run rules to manage or perform actions on agents
   - **Allowed agent types** - specify which categories of AI agents are permitted in the organization
   - **Policy templates** - preset policies, rules, and allow lists for new agents
   - **Sharing** - manage who can share agents and how
   - **User access** - control which users or groups can interact with agents

    ![](./media/a365-ex2-t1-02.png)

1. Select **Policy templates** to view the policy templates available in your tenant.

    >**Note:** Some Microsoft documentation refers to this area as **Security templates** or simply **Templates**. In this lab tenant it is labeled **Policy templates**. All three refer to the same place.

1. Select **Default policy templates for agents** to open it.

    ![](./media/a365-ex2-t1-03.png)

    >**Note:** Microsoft documentation describes two default templates, **Default policy template for all agents except AI teammates** and **Default policy template for AI teammates in Frontier**. This lab tenant exposes a single template named **Default policy templates for agents**. The policy-template experience is in preview and its naming changes between service builds, so work with whichever template name your tenant displays.

1. Review the bundled policies. **In this lab tenant you will see six policies**, drawn from Microsoft Purview and Microsoft Entra:

    | Policy name | Source service | What it protects against |
    | --- | --- | --- |
    | Purview audit enabled | Microsoft Purview | No record of what the agent did |
    | Detect sensitive information (DSPM) in AI interaction | Microsoft Purview | Sensitive data leaking through agent prompts and responses |
    | Purview AI compliance assessment | Microsoft Purview | Undetected compliance drift over time |
    | Identity protection | Microsoft Entra | Compromised or anomalous agent identities |
    | Network visibility | Microsoft Entra | Unmonitored agent access to external resources |
    | Lifecycle management for agents | Microsoft Entra | Orphaned agents accumulating after owners leave |

    ![Default policy template detail showing the six bundled policies](./media/a365-ex2-t1-04.png)

1. The full documented template also bundles six further policies from SharePoint Online and Microsoft Defender. **These are not present in this lab tenant.** Review them so you understand the complete protection set:

    | Policy name | Source service | What it protects against |
    | --- | --- | --- |
    | Agent access insights | SharePoint Online | Not knowing which sites agents are reading |
    | Restrict external sharing of sites and its content | SharePoint Online | Agents discovering content they should not see |
    | Access control for sites and OneDrive | SharePoint Online | Overbroad agent access to user files |
    | Content permissions insights | SharePoint Online | Oversharing invisible to admins |
    | AI real time protection and investigation | Microsoft Defender | Malicious agent behaviour at runtime |
    | Advanced hunting | Microsoft Defender | Inability to investigate a suspected incident |

    >**Note:** **Why are the SharePoint Online and Defender policies missing?** Not licensing. This tenant has `SHAREPOINTENTERPRISE`, `M365_COPILOT_SHAREPOINT`, and `DEFENDER_FOR_AI` all fully provisioned. The policy-template experience is a **preview** feature, and Microsoft documents these scenarios as available only to tenants enrolled in the **Frontier** preview program. When you ran `a365 setup all` in Exercise 1, the CLI warned that Frontier enrollment could not be verified. The reduced six-policy template is the expected result in a non-Frontier tenant, and it is **not a failure or a misconfiguration**. Continue to Task 2; every remaining task in this exercise is generally available and unaffected.

    >**Note:** Even in a tenant where they do appear, the four SharePoint Online policies require extra work before they take effect. An administrator must visit the SharePoint admin center to create the insights report or apply the **Restrict Content** discovery setting, and the tenant needs a Microsoft 365 Copilot license.

    >**Note:** Microsoft's built-in default policies appear **locked** and cannot be edited. This is by design: they are the enforced baseline. Custom policies inside a template are not yet supported.

    >**Note:** Policy templates apply only at **new agent activation**. You cannot retroactively apply a template to an already-approved agent. Editing a template affects future activations only.

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

1. Review the three sharing options:

   - **All users** - all users can share their agents with others in your tenant
   - **No users** - org-level sharing is disabled, though users can still share directly with specific individuals
   - **Specific users** - restrict broad sharing permissions to designated groups

1. Select **Specific users**, then select the group or user permitted to share agents. Select **Save**.

    ![Sharing settings page with the Specific users option selected](./media/a365-ex2-t2-02.png)

    >**Note:** Only agents built with **Microsoft 365 Copilot Agent Builder** are governed by this sharing control. This is a common source of confusion: restricting sharing here does not restrict every agent platform.

1. Return to **Agent settings** and select **User access**.

1. Review the three options:

   - **All users** - the default; all users can access agents, subject to existing app policies and user assignments
   - **No users** - no users in the organization can access agents
   - **Specific users/groups** - only the users or groups you select can use agents

1. Leave **All users** selected for this lab, so you can test Finley in Exercise 4. Select **Save**.

    ![User access settings page showing the access scope options](./media/a365-ex2-t2-03.png)

1. Return to **Agent settings** and select **Agent management rules**.

1. Review the supported rule-based bulk actions. These let you apply governance across many agents at once instead of one at a time:

   - **Install Microsoft agents** - proactively deploy Microsoft first-party agents tenant-wide
   - **Reassign ownerless agents created with Agent Builder to manager** - automatically transfer ownership of orphaned agents to the previous owner's manager, based on the Microsoft Entra ID hierarchy

    ![Agent management rules page showing the available bulk governance rules](./media/a365-ex2-t2-04.png)

    >**Note:** The reassignment rule is supported **only** for agents created with Microsoft 365 Copilot Agent Builder. It will not reassign Finley, which was provisioned through the Agent 365 CLI.

    >**Note:** The ownerless-agent rule directly addresses one of the governance gaps surfaced on the Agent overview dashboard. Agents become ownerless when their creator leaves the organization, and without a rule an admin has to find and fix each one manually.

### Task 3: Create and assign custom security attributes

You cannot write "only approved agents may access Finance resources" until you have a way to record which agents are approved. Custom security attributes are that mechanism: business-specific key-value pairs you attach to directory objects and then target in policy.

**Global Administrator does not include permission to define or assign custom security attributes.** These permissions are deliberately separated from tenant administration, so even a Global Administrator sees the attribute controls greyed out until the correct roles are assigned. Your first job in this task is to grant yourself those roles.

>**Note:** This role separation is intentional. Custom security attributes can drive access control decisions, so Microsoft keeps their management behind dedicated roles rather than bundling them into Global Administrator. A Global Administrator **can** assign these roles to themselves, which is what you do next.

1. Switch to the Microsoft Entra admin center tab, or navigate to:

    ```
    https://entra.microsoft.com
    ```

1. In the left navigation pane, expand **Entra ID (1)** and select **Roles & admins (2)**.

    ![Microsoft Entra admin center with Roles and admins selected](./media/a365-ex2-t3-01.png)

1. In the search box, type **Attribute Definition Administrator** and select the role from the results.

1. Select **+ Add assignments**.

1. Select your lab user account <inject key="AzureAdUserEmail"></inject>, then select **Add**.

    ![Add assignments pane with the lab user selected for the Attribute Definition Administrator role](./media/a365-ex2-t3-02.png)

1. Return to **Roles & admins**, search for **Attribute Assignment Administrator**, and repeat the assignment for the same account.

    >**Note:** You need both roles. **Attribute Definition Administrator** lets you create attribute sets and attribute definitions. **Attribute Assignment Administrator** lets you assign attribute values to objects such as Finley, and also lets the Conditional Access policy builder in Task 4 read your attribute names so you can select them.

1. **Sign out of the Microsoft Entra admin center and sign back in** with the same credentials.

    >**Note:** This step is required, not optional. Role assignments are written into your access token when you sign in. Until you get a fresh token, the attribute controls remain greyed out even though the roles are assigned. If **Add attribute set** is unavailable in the next steps, this sign-out is the step you skipped.

1. In the left navigation pane, expand **Entra ID (1)** and select **Custom security attributes (2)**.

    ![Microsoft Entra admin center with Custom security attributes selected](./media/a365-ex2-t3-03.png)

1. Select **Add attribute set**.

1. Enter the following values, then select **Add**:

   - **Attribute set name:** **AgentAttributes**
   - **Description:** **Attributes used to classify AI agents**
   - **Maximum number of attributes:** **25**

    ![New attribute set pane with AgentAttributes entered](./media/a365-ex2-t3-04.png)

    >**Note:** Attribute set names can be up to 32 characters with no spaces or special characters, and **cannot be renamed or deleted** later. Only deactivation is possible, so type the name carefully. The same applies to attribute names in the next step.

1. Select the **AgentAttributes** attribute set you just created to open it.

1. Select **Add attribute** and enter the following:

   - **Attribute name:** **AgentApprovalStatus**
   - **Description:** **Review and approval state of the agent**
   - **Data type:** **String**
   - **Allow multiple values to be assigned:** **Yes**
   - **Only allow predefined values to be assigned:** **Yes**

    ![New attribute pane with AgentApprovalStatus configured](./media/a365-ex2-t3-05.png)

1. Because you selected **Yes** for predefined values, select **Add value** and add each of the following values, leaving **Is active?** set to **Yes** for each:

   - **New**
   - **In_Review**
   - **HR_Approved**
   - **Finance_Approved**
   - **IT_Approved**

    ![Add predefined value pane showing the approval status values](./media/a365-ex2-t3-06.png)

1. Select **Save**.

1. Now assign the attribute to Finley. In the left navigation pane, navigate to **Entra ID** > **Agents** > **Agent identities**.

1. Select the agent identity **finley-<inject key="DeploymentID" enableCopy="false"/> Identity** that you created in Exercise 1.

1. In the left navigation pane of the agent identity, select **Custom security attributes**.

1. Select **Add assignment**, then set:

   - **Attribute set:** **AgentAttributes**
   - **Attribute name:** **AgentApprovalStatus**
   - **Assigned value:** **Finance_Approved**

    ![Add assignment pane assigning Finance_Approved to the agent identity](./media/a365-ex2-t3-07.png)

1. Select **Save**, then confirm the assignment now appears in the agent identity's **Custom security attributes** list.

Finley is now tagged as an approved Finance agent. In the next task you write the policy that acts on that tag.

>**Note:** In a fuller production design you would also create a second attribute set, for example **ResourceAttributes** with a **Department** attribute, and tag the resources themselves. That lets you write policies such as "only `HR_Approved` agents may reach resources tagged `HR`". This lab keeps a single attribute set and targets all resources, which demonstrates the same pattern in the time available.

### Task 4: Create an attribute-driven Conditional Access policy

Now you build the rule that uses the tag. The policy blocks all agent identities *except* those carrying the `Finance_Approved` value, which means any new untagged agent that appears in the tenant is denied access by default.

>**Note:** Creating this policy requires **Conditional Access Administrator** to build the policy and **Attribute Assignment Administrator** so the attribute picker can read your attribute definitions. Global Administrator covers the first, and you granted the second in Task 3. If the attribute dropdown in step 8 is empty, you either skipped the role assignment or the sign-out and sign-in.

1. In the Microsoft Entra admin center, in the left navigation pane, expand **Entra ID (1)**, select **Conditional Access (2)**, and then select **Policies (3)**.

    ![Microsoft Entra admin center with Conditional Access Policies selected](./media/a365-ex2-t4-01.png)

1. Select **+ New policy**.

1. In the **Name** field, enter:

    ```
    CA-Agents-BlockUnapprovedAgents
    ```

    >**Note:** Adopt a consistent naming standard for Conditional Access policies. A name that encodes scope and intent saves considerable time during incident response.

1. Under **Assignments**, select **Users, agents or workload identities**.

    >**Note:** Depending on your tenant build this may read **Users, agents (Preview) or workload identities**. Agent targeting is a preview capability, so the **(Preview)** suffix appears in some tenants and not others.

1. Under **What does this policy apply to?**, select **Agents**.

    ![Assignments pane with Agents selected as the policy target](./media/a365-ex2-t4-02.png)

    >**Note:** Four options appear here: **All agent identities**, **All agent users (Preview)**, **Select agent identities**, and **Select agent users (Preview)**. Finley has an agent identity, so this exercise uses the agent identity options. Note also that a policy targeting **All users** does **not** include agent user accounts, which is a common and dangerous assumption.

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

    >**Note:** A target resource must have a service principal in your tenant to be protected by Conditional Access. This applies to Microsoft Graph, MCP servers, OpenAPI tools, and any custom tool you build.

1. Under **Access controls** > **Grant**, select **Block access**, then select **Select**.

    ![Grant control pane with Block access selected](./media/a365-ex2-t4-05.png)

    >**Note:** This control appears as **Block access** in most tenants and simply **Block** in some builds. **Block** is the only available grant control for agent identities, because there is no interactive remediation such as multifactor authentication available to a non-human identity.

1. At the bottom of the page, set **Enable policy** to **Report-only**.

    ![Enable policy toggle set to Report-only](./media/a365-ex2-t4-06.png)

    >**Note:** Do not set this policy to **On**. It blocks **every** agent identity in the tenant except those tagged `Finance_Approved`. If you enable it, any agent you add later without that tag stops working immediately, and no agent can prompt a human to fix it. Report-only mode gives you the evaluation data without the outage.

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

    >**Note:** When a policy targets agent identities, **Agent risk (Preview)** is the **only** condition available. Conditions such as device platforms, filter for devices, agent execution environments, and network apply only to agent **user** accounts, because they depend on signals that only an endpoint can provide.

1. Under **Configure agent risk levels needed for policy to be enforced**, select **High**.

    ![Agent risk condition configured for High risk level](./media/a365-ex2-t5-01.png)

    >**Note:** Microsoft recommends starting with **High** only. Including **Medium** increases the chance of blocking legitimate agent activity. Tune this per your organization's risk tolerance.

1. Under **Access controls** > **Grant**, select **Block access**, then select **Select**.

    >**Note:** For agents authenticating with their own identity there is no remediation path such as multifactor authentication, because no human is present to respond to a prompt. **Block** is the only meaningful control.

1. Set **Enable policy** to **Report-only**, then select **Create**.

1. Select your new policy and review its impact data.

    ![Conditional Access policies list showing both new report-only agent policies](./media/a365-ex2-t5-02.png)

    >**Note:** Agent risk requires Microsoft Entra ID Protection, which is included with the Microsoft Entra ID P2 license present in this lab tenant.

    >**Note:** In a freshly provisioned lab tenant, Microsoft Entra ID Protection has not yet observed enough activity to flag any agent as high risk, so you will most likely see zero affected identities. This is expected and is not a failure. The learning objective here is the policy construction and the safe validation workflow, not observing a live detection. In production, risk detections accumulate as agents operate.

1. Confirm both policies now appear in the **Policies** list with a state of **Report-only**.

### Task 6: Review conformance and risk signals in the registry

Finally, close the loop. The Microsoft 365 admin center aggregates signals from Entra, Defender, and Purview into a single view, so an administrator does not have to check three portals to find out whether an agent is a problem.

1. Switch to the Microsoft 365 admin center tab.

1. Navigate to **Agents (1)** > **All agents (2)** and ensure the **Registry (3)** tab is selected.

1. Locate the **Risks** column in the agent list. This column aggregates high-severity risks across Microsoft Entra, Microsoft Defender, and Microsoft Purview.

    ![All agents page with the Risks column visible in the registry list](./media/a365-ex2-t6-01.png)

    >**Note:** The Risks column exists to close a specific visibility gap. Before it, an administrator governing agents had to correlate findings across three separate security portals manually.

    >**Note:** The Risks column shows **high severity** risks only. A count of `0` means no high-severity risks, not that the agent is free of all risk. Lower-severity findings remain visible in the individual security portals. There can also be a delay of up to an hour between a detection appearing in a security portal and the count updating here.

1. Select your agent **finley-<inject key="DeploymentID" enableCopy="false"/>** to open the details pane, and review the **Permissions** section to confirm what the agent has been granted.

1. Navigate to **Agents** > **Overview** and review the governance action cards, which surface items such as **Pending requests**, **Agents at risk**, and **Agents without owners**.

    ![Agent overview dashboard showing the governance action cards](./media/a365-ex2-t6-02.png)

    >**Note:** Card names and layout on this dashboard change frequently as the preview evolves. Focus on locating the governance cards rather than matching exact tile titles.

1. Confirm that **Agents without owners** does not include your agent, because you assigned owners in Exercise 1.

    >**Note:** Selecting the **Agents without owners** card filters the agent list to ownerless agents so you can triage them directly. Because you closed Finley's accountability gap in Exercise 1, Finley should not appear in that filtered view.

    >**Note:** Governance actions such as approving agent requests or assigning ownership can only be performed by users in the **AI Administrator** or **Global Administrator** roles. Other roles can view governance gaps but cannot remediate them.

## Conclusion

In this exercise you moved Finley from "exists" to "constrained". You reviewed the protections Microsoft enforces by default and traced each back to the service that implements it, tightened tenant-wide guardrails through Agent settings, granted yourself the attribute administrator roles that even a Global Administrator lacks by default, and built the two Conditional Access patterns that matter most for agents at scale: attribute-driven approval and risk-based blocking.

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
