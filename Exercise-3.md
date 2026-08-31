# Exercise 3: Windows 365 for Agents

### Estimated Duration: 60 minutes

Some agents only need APIs. Others must operate a Windows desktop, clicking through applications the way a person would. In this exercise you establish the compute case for Finley, enable Windows 365 for Agents and create the billing policy that meters agent compute, give the agent a user account, define what a healthy agent device looks like, scope that definition so it never touches employee laptops, and enforce it with Conditional Access.

## Overview
**Windows 365 for Agents** provides AI agents with secure, on-demand Cloud PCs that have a managed identity, device posture, and a governed session lifecycle. It exists for **computer-using agents (CUA)**: agents that work by operating a desktop rather than calling an API.
This matters for Contoso because their legacy expense portal has no API. Finley must open it, sign in, and enter data through the interface, exactly as a person would. That requires a machine.
Buying the compute is the easy part. What decides whether it is safe is everything that must be true *before* a Cloud PC reaches an agent: someone decided the agent needs one, the cost meter is owned, "healthy" is defined for agent devices, that definition is scoped so it never touches employee laptops, and access is denied when a device fails it. You build that layer here, for the Finley agent from Exercises 1 and 2.
Four things to know before you begin:
- A **billing policy** is the meter for agent compute, created in Microsoft 365 admin center cost management and bounded by a budget. Windows 365 for Agents is **consumption-billed**, not seat-licensed.
- **Two switches** must be on before an agent pool can exist: the pay-as-you-go connection, and tenant-level enablement in Microsoft Intune.
- An **agent user account** is a user-like identity parented to an agent identity. You create one using Microsoft Graph, and it is the object **device compliance is evaluated against**.
- The **Agent execution environments** condition restricts a Conditional Access policy to agent sessions initiated from an endpoint. Without it, agents running in Microsoft infrastructure can never be compliant — they have no device at all.
>**Note:** You create the billing policy and review the Cloud PC agent pool model, but you do **not** provision Cloud PCs. Every control here works before a device exists.

## In this exercise you will

- Establish the compute case for the agent and confirm its accountability
- Create a Windows 365 for Agents billing policy and enable the service for your tenant
- Create an agent user account and review the Cloud PC agent pool configuration model
- Define the compliance baseline for agent Cloud PCs
- Scope the baseline so it applies only to agent devices
- Require a compliant device for agent sessions and validate the policy
- Apply cost governance and decommission agent compute

## Objectives

- **Task 1**: Establish the compute case for the agent and confirm its accountability
- **Task 2**: Enable Windows 365 for Agents and review the agent compute model
- **Task 3**: Define the compliance baseline for agent Cloud PCs
- **Task 4**: Scope the baseline to agent devices only
- **Task 5**: Require a compliant device for agent sessions and validate
- **Task 6**: Apply cost governance and decommission agent compute

### Task 1: Establish the compute case for the agent and confirm its accountability

Before provisioning any compute, an administrator has to answer two questions: does this agent actually need a desktop, and is someone accountable for it. In this task you answer both for Finley, and record the answer where an auditor can find it.

1. On the lab virtual machine, open **Microsoft Edge** and navigate to the Microsoft 365 admin center:

    ```
    https://admin.microsoft.com/
    ```

1. If prompted, sign in with the following credentials:

   - **Email/Username:** <inject key="AzureAdUserEmail"></inject>
   - **Password:** <inject key="AzureAdUserPassword"></inject>

1. In the left navigation pane, Select **Agents (1)** > **All agents (2)**, and make sure the **Registry (3)** tab is selected and locate the agent **finley-<inject key="DeploymentID" enableCopy="false"/> (4)** that you created in Exercise 1.

    ![](./media/ex3-01.png)

1. Select the agent to open its details pane. Review the following, exactly as you did in Exercise 1, Task 5:

   - The **Owner**
   - The agent's **Permissions**
   - The **Status** of the agent

1. While reviewing the agent's properties, tools, and permissions. Note what Finley can already reach **without** a desktop, because that is the baseline against which the Cloud PC decision is justified.

1. Recall the distinction established in Exercise 1 between the two objects that represent your agent:

    | Object | What it is | What it is used for |
    | --- | --- | --- |
    | **Agent identity** | The application-like identity of the agent itself | Permissions, API access, blueprint-derived configuration |
    | **Agent user account** | A user-like identity the agent holds | Licenses, collaboration, and **device compliance evaluation** |

1. Now apply the compute decision framework. Not every agent needs a machine, and provisioning one reflexively wastes money and adds attack surface:

    | Pattern | What it means | Does it need a Cloud PC? |
    | --- | --- | --- |
    | **API-first** | The target system exposes an API the agent can call directly | **No.** Cheapest option, smallest attack surface |
    | **CUA (computer-using agent)** | The agent reasons about a screen and adapts to what it sees | **Yes.** Requires a desktop to operate |
    | **RPA (robotic process automation)** | A fixed, scripted sequence repeated identically | Needs a runner, but no reasoning and often no full desktop |

1. Apply the framework to Contoso's scenario and confirm the conclusion:

   - Contoso's legacy expense portal has **no API**, so API-first is not available.
   - The portal's interface **changes between releases**, so a fixed RPA script would break on every release.
   - Finley must therefore reason about the screen and adapt, which makes this a **CUA scenario** that justifies a Cloud PC.

1. Also decide how the agent will run, because it changes how strictly you must govern it:

   - **Attended** execution means a human is present and can approve or intervene.
   - **Unattended** execution means the agent runs with no human present, on a schedule or in response to an event.

    >**Note:** Finley processes expenses on a schedule overnight, so it is **unattended**. Unattended agents need stricter Conditional Access than attended ones, because there is no human present to notice a mistake or a compromised session.

1. Record the decision so it is auditable rather than tribal knowledge. Switch to the Microsoft Entra admin center:

    ```
    https://entra.microsoft.com
    ```

1. In the left navigation pane, expand **Entra ID (1)** and select **Custom security attributes (2)**, then open the attribute set you created in **Exercise 2**.

    ![](./media/ex3-1.png)

1. Select **Add attribute** and create a new attribute with the following values:

   - **Attribute name:** **RequiresDesktopCompute**
   - **Description:** **Records whether this agent has an approved business case for a Cloud PC**
   - **Data type:** **String**
   - **Allow multiple values to be assigned:** **No**
   - **Only allow predefined values to be assigned:** **Yes**
   - **Predefined values:** **Yes** and **No**
     
       ![](./media/ex3-2.png)

       ![](./media/ex3-3.png)

1. Select **Save**.

1. Navigate back to **Entra ID** > **Agents** > **Agent identities**, select your agent **finley-<inject key="DeploymentID" enableCopy="false"/> Identity**, open its **Custom security attributes (1)** section, and select **Add assignment (2)**.

    ![](./media/ex3-4.png)

1. Assign the attribute you just created:

   - **Attribute set:** **AgentAttributes**
   - **Attribute name:** **RequiresDesktopCompute**
   - **Assigned value:** **Yes**

     ![](./media/ex3-5.png)

1. Select **Save**, then refresh the page and confirm the value persisted.

### Task 2: Enable Windows 365 for Agents and review the agent compute model

Agent compute is metered differently from the rest of Windows 365, and it has to be switched on in two places before it can be used. In this task you create the meter, enable the service, give Finley a user account, and then review the pool configuration model that runs on top of all of it.

1. Open a new browser tab and navigate to the Microsoft 365 admin center:

    ```
    https://admin.microsoft.com/
    ```
1. In the left navigation pane, select **Copilot (1)**, then select **Cost management (2)**.

1. At the top of the page, select the classic **Billing & usage (3)** link.

    ![](./media/ex3-6.png)

1. Select **Billing policies (1)**, then select **Add a billing policy (2)** and provide the following:

   - **Name:** **finley-plan-<inject key="DeploymentID" enableCopy="false"/> (1)**
   - **Azure subscription:** select the lab subscription **(2)**
   - **Resource group:** select the lab resource group **(3)**
   - **Region:** select the region closest to your lab, in the same country or continent group as the geography you will pick in Intune later in this task **(4)**
   - Select **Checkbox** to accept terms of service.
   - Click **Next (6)**

     ![](./media/ex3-7.png)

     ![](./media/ex3-8.png)

1. Click **Next** on choose user tab.

1. On the Budget tab, select the **Checkbox (1)** to set a budget for this policy, enter **10 (2)** in the Amount textbox, and click **Next (3)**.

     ![](./media/ex3-9.png)

1. Review the details and click **Create Policy**.

1. Return to **Billing & usage**, select **Pay-as-you-go services (1)**, then open **Windows 365 for Agents (2)**.

     ![](./media/ex3-10.png)

1. Switch its **Connection status (1)** toggle on and click **Save (2)**. Confirm the row now reads **Connected** 

    ![](./media/ex3-11.png)

1. Open a new browser tab and navigate to the Microsoft Intune admin center:

    ```
    https://intune.microsoft.com/
    ```

1. If prompted, sign in with the following credentials:

   - **Email/Username:** <inject key="AzureAdUserEmail"></inject>
   - **Password:** <inject key="AzureAdUserPassword"></inject>

1. In the left navigation pane, select **Devices (1)**, then under **Manage Windows 365 Cloud Pcs** select **Provision Cloud PCs (2)**.

    ![](./media/ex3-12.png)

1. Select the **Provisioning policies (Agents)** tab, and select **Set up Windows 365 for Agents**.

    ![](./media/ex3-14.png)

1. On the **Set up Windows 365 for Agents** screen, toggle **Turn on Windows 365 for Agents for your tenant** to **On**.
    
     ![](./media/ex3-15.png)

1. Now return to the **Windows powershell** and run the below command:

    ```
    $token = az account get-access-token --resource "https://graph.microsoft.com" --query accessToken -o tsv
    $body = @{
            "@odata.type"       = "#microsoft.graph.agentUser"
            "accountEnabled"    = $true
            "displayName"       = "Finley Cloud PC Agent"
            "mailNickname"      = "finley_agent"
            "userPrincipalName" = "finley_agent@otuwamsbxxxxxx.onmicrosoft.com"
            "identityParentId"  = "Agent Object Id"
    } | ConvertTo-Json
 
    Invoke-RestMethod -Method Post -Uri "https://graph.microsoft.com/beta/users" -Headers @{ Authorization = "Bearer $token" } -ContentType "application/json" -Body $body
    ```
    >**Note:** Replace **otuwamsbxxxxxx.onmicrosoft.com** from the **environment** tab.

    >**Note:** Replace **Agent Object Id** by navigating to **Entra admin center**, Entra ID > Agents > Agent Identities and Copy **Object Id**. 

    >![](./media/ex3-16.png)

    ![](./media/ex3-13.png)

1. Once completed, navigate back to the **Provision policies (Agents)** tab. You should now see the **+ Create policy (Agents)** option enabled.
    
     ![](./media/ex3-17.png)

1. Select **+ Create policy (Agents)** to open the wizard, and step through the pages to review what a Cloud PC agent pool defines.

    >**Note:** You are reviewing the configuration surface, not provisioning Cloud PCs. **Do not complete this wizard.** You will cancel it at the end of this task. Provisioning Cloud PCs takes 20 to 30 minutes and immediately begins consumption billing, neither of which is needed to build the controls in this exercise.

1. On the **General (1)** page, review the following settings and note what each one controls:

    | Setting | What it controls | Governance consideration |
    | --- | --- | --- |
    | **Billing plan** | Which meter the pool bills against | This is where the plan you just created is consumed |
    | **Always available Cloud PCs count** | How many Cloud PCs are provisioned and kept ready (1–200) | Each one is provisioned immediately and billed continuously. This number must reflect real concurrency, not optimism |
    | **Geography** | Where the agent's desktop and its data reside | A data residency decision, not a performance one |

    ![](./media/ex3-18.png)

    >**Note:** You may enter the details shown in the image to proceed. Make sure not to click Create policy at the end.

1. Select **Next (2)** and review the **Agents** page. Select **Add Agents** to see the agent picker.
    
    ![](./media/ex3-20.png)

    ![](./media/ex3-19.png)

1. Close the agent picker after selecting an agent, then select **Next** and review the **Image** page. Note the two image types available:

   - **Gallery image** — the default images Microsoft provides.
   - **Custom image** — images your organization has uploaded through the **Add device images** workflow.

     ![](./media/ex3-21.png)

1. Select **Next** and review the **Configuration** page. The **Language & region** value determines which language pack is installed on every Cloud PC provisioned by the policy.

    ![](./media/ex3-22.png)

1. Select **Next** and review the **Scope tags** page.

1. On **Review + Create** page, Select **Cancel** to exit the wizard **without** creating a policy.

    ![](./media/ex3-23.png)

1. Confirm the **Provisioning policies (Agents)** list is still empty, and that no Cloud PCs have been created.

    ![](./media/ex3-24.png)

1. Finally, review the **agent session lifecycle**, which explains why pool sizing works the way it does:

   - An agent **claims** a session from the pool when it begins work.
   - It **holds** the session for the duration of the task.
   - It **releases** the session back to the pool when it finishes.

    >**Note:** This is why a pool of one Cloud PC can serve many sequential agent tasks. Pool size tracks **concurrency**, not the number of agents. Ten agents that never run at the same time need one Cloud PC. Two agents that always run together need two.

### Task 3: Define the compliance baseline for agent Cloud PCs

A Cloud PC for Agents is an Intune-managed Windows device. That means Conditional Access can evaluate its compliance exactly as it would an employee's laptop — but only if you have defined what compliance means. In this task you write that definition.

1. In the Microsoft Intune admin center, in the left navigation pane select **Devices (1)**, then under **Manage devices** select **Compliance (2)**.

    ![](./media/ex3-25.png)

1. On the **Policies** tab, select **+ Create policy**.

    ![](./media/ex3-26.png)

1. In the **Create a policy** pane, select the following, then select **Create (3)**:

   - **Platform:** **Windows 10 and later (1)**
   - **Profile type:** **Windows 10/11 compliance policy (2)**

    ![](./media/ex3-27.png)

1. On the **Basics** page, enter the following values, then select **Next (3)**:

   - **Name:** **A365-Agent-CloudPC-Compliance-<inject key="DeploymentID" enableCopy="false"/> (1)**
   - **Description:** **Compliance baseline for Windows 365 Cloud PCs used by agents (2)**

    ![](./media/ex3-28.png)

1. On the **Compliance settings** page, expand **Device Health** and configure:

   - **BitLocker:** **Require**
   - **Secure Boot:** **Require**
   - **Code integrity:** **Require**

    ![](./media/ex3-29.png)

1. Expand **Device Properties** and in the **Minimum OS version** field enter the following value. Leave every other field in this section as **Not configured**:

    ```
    10.0.22000.0
    ```

    ![](./media/ex3-30.png)

1. Expand **System Security** and configure:

   - **Microsoft Defender Antimalware:** **Require**
   - **Microsoft Defender Antimalware security intelligence up-to-date:** **Require**

    ![](./media/ex3-31.png)

1. Select **Next**.

1. On the **Actions for noncompliance** page, review the single action that is already present. **Make no changes on this page.** and Select **Next**.

   - **Action:** **Mark device noncompliant**
   - **Schedule (days after noncompliance):** **Immediately** — this is fixed and cannot be edited
   - Leave the empty **Action** dropdown on the row below untouched

    ![](./media/ex3-32.png)

1. On the **Assignments** page, do **not** assign the policy yet. Select **Next**.

1. On the **Review + create** page, review your configuration and select **Create**.

    ![](./media/ex3-33.png)

1. Confirm the policy appears in the **Policies** list.

    ![](./media/ex3-34.png)

### Task 4: Scope the baseline to agent devices only

The policy you just wrote is deliberately strict. Assigned carelessly, it would measure every employee laptop in the tenant against a standard written for unattended agent compute. In this task you scope it so it governs agent devices and nothing else.

1. Consider what would happen if you assigned the Task 3 policy to **All devices**:

   - Every corporate Windows device would be evaluated against a zero-grace-period, code-integrity-required baseline.
   - Devices that fail would be marked noncompliant **immediately**, with no grace window.
   - Any Conditional Access policy requiring compliance would then start blocking real users.

    >**Note:** This is the most common way a well-intentioned compliance policy causes an outage. The policy is not wrong; the assignment is. Scoping is what makes a strict baseline politically survivable, and a baseline nobody can safely assign is a baseline that never ships.

1. Switch to the Microsoft Entra admin center tab, or navigate to:

    ```
    https://entra.microsoft.com
    ```

1. In the left navigation pane, expand **Groups (1)** and select **All groups (2)**, then select **New group (3)**.

    ![](./media/ex3-35.png)

1. On the **New Group** page, configure the following:

   - **Group type:** **Security**
   - **Group name:** **A365-Agent-CloudPCs-<inject key="DeploymentID" enableCopy="false"/>**
   - **Group description:** **Cloud PCs provisioned for agents, scoped by enrollment profile**
   - **Membership type:** **Dynamic Device**

    ![](./media/ex3-36.png)

1. Next to **Dynamic device members**, select **Add dynamic query**.

1. Select **Edit** to switch to the **rule syntax** text box, and enter the following rule:

    >device.enrollmentProfileName -eq "finley-pool-<inject key="DeploymentID" enableCopy="false"/>"

    ![](./media/ex3-37.png)

1. Select **Save**, then select **Create**.
  
    ![](./media/ex3-38.png)

    ![](./media/ex3-39.png)

1. Open the group and confirm the following:

   - The membership rule saved exactly as entered.
   - The group currently has **0 members**.

    ![](./media/ex3-44.png)

1. Switch back to the Microsoft Intune admin center tab.

1. Navigate to **Devices** > **Compliance**, and select your policy **A365-Agent-CloudPC-Compliance-<inject key="DeploymentID" enableCopy="false"/>**.

1. Select **Properties**, then next to **Assignments** select **Edit**.

   ![](./media/ex3-40.png)

1. Under **Included groups**, select **+ Add groups**, choose **A365-Agent-CloudPCs-<inject key="DeploymentID" enableCopy="false"/> (1)**, and select **Select (2)**.

    ![](./media/ex3-41.png)
 
1. Select **Review + save**, then select **Save**.

    ![](./media/ex3-42.png)

1. Confirm the assignment appears on the policy's properties page.

    ![](./media/ex3-43.png)

### Task 5: Require a compliant device for agent sessions and validate

You have defined what a healthy agent device looks like and scoped that definition correctly. It currently has no effect on access. In this task you turn it into an access decision, and then validate that the policy is actually being evaluated.

1. In the Microsoft Entra admin center, in the left navigation pane expand **Entra ID (1)**, select **Conditional Access (2)**, then select **Policies (3)**.

    ![](./media/ex3-45.png)

1. Select **+ New policy**.

1. In the **Name (1)** field, enter:

    ```
    CA-Agents-RequireCompliantDevice
    ```

1. Under **Assignments**, select **Users or agents (2)**.

1. Under **What does this policy apply to?**, select **Agents (3)**, then under **Include** select **All agent users (Preview) (4)**.

    ![](./media/ex3-46.png)

1. Under **Target resources (1)**, select **Include (2)**, then select **All resources (formerly 'All cloud apps') (3)**.

    ![](./media/ex3-47.png)

1. Under **Conditions (1)**, select **Agent execution environments (Preview) (2)**, and set **Configure** to **Yes (3)**, and under **Include**, select **Agent user sessions initiated from endpoints (4)** and select **Done (5)**.

    ![](./media/ex3-48.png)

1. Under **Access controls**, select **Grant (1)**, then select **Grant access (2)**, select **Require device to be marked as compliant (3)**, and select **Select (4)**.

    ![](./media/ex3-49.png)

1. Set **Enable policy** to **Report-only (1)**, then select **Create (2)**.

    ![](./media/ex3-50.png)

1. Now validate the policy using the **What If** tool, which models a sign-in and reports which Conditional Access policies would be evaluated for it. In the left navigation pane, under **Conditional Access**, select **Policies**, then select **What If**.

    ![](./media/ex3-51.png)

1. Under **Identity**, configure the following:

   - **Select identity type:** **Users**
   - **User:** select **Edit user**, then choose  your own lab account **<inject key="AzureAdUserEmail"></inject> (1)** and select **Select (2)**.

     ![](./media/ex3-52.png)
 
1. Under **Target resource**, set **Select target type** to **Cloud apps**, then select **+ Select cloud app**.

1. On the **Resources** pane, in the **Search** box type the following:

    ```
    Office 365
    ```

1. In the results, select the checkbox for **Office 365 Exchange Online**, then select **Select**.

    ![](./media/ex3-53.png)

1. Scroll down to **Sign-in conditions** and set the two required conditions.

   - **Device platform:** **Windows**
   - **Client app:** **Browser**

    ![](./media/ex3-54.png)

1. Leave every remaining condition unset: **Authentication flow**, **Insider risk**, **Sign-in risk**, **User risk**, **IP address**, **Country**, and **Filter for devices**.

1. Select **What if** to run the evaluation.

1. Review the **Evaluation result**. The output is split across two tabs, **Policies that will apply** and **Policies that will not apply**.

1. On the **Policies that will apply** tab, note that it reports **0 policies found**.

    ![](./media/ex3-55.png)

    >**Note:** This is the **correct** result, not a failure. You evaluated a sign-in by a **human user**, and all three of your agent policies are assigned to **Agents**. A human signing in is therefore in scope of none of them. An empty result here is positive evidence that your agent policies do not reach human accounts — which is exactly the scoping precision you want, and exactly the mistake that a policy assigned to All users would have made.

1. Navigate back to **Conditional Access** > **Policies** and confirm all three of your agent policies now appear together, all in **Report-only**.

    ![](./media/ex3-56.png)

1. Review what those three policies now cover, which closes the Zero Trust frame you opened in Exercise 2:

    | Policy | Created in | The question it answers |
    | --- | --- | --- |
    | Attribute-driven agent policy | Exercise 2 | **Is this agent approved?** |
    | Risk-based agent policy | Exercise 2 | **Is this agent behaving riskily?** |
    | **CA-Agents-RequireCompliantDevice** | Exercise 3 | **Is this agent running on a healthy device?** |


### Task 6: Apply cost governance and decommission agent compute

You created a cost meter in Task 2, so you own turning it off. Two things have to be reversed: the pay-as-you-go connection and the billing policy itself. This task is short, but it is not optional.

1. Before deleting anything, confirm you understand what actually costs money in Windows 365 for Agents:

    | Object | Does it incur cost? |
    | --- | --- |
    | **Billing plan** with no provisioning policy | **No.** It is a meter with nothing running against it |
    | **Provisioning policy** provisioning Cloud PCs | **Yes.** Continuously, from provisioning until the policy is deleted |
    | Agent **not currently running**, but a pool exists | **Yes.** Cost tracks capacity, not activity |

    >**Note:** That third row is the one that surprises people. An idle agent does not stop the meter. Cloud PC capacity is billed for as long as it exists, whether or not any agent is claiming a session. Deleting the provisioning policy is the single action that stops consumption — nothing else does, including deleting the agent.

1. In the Microsoft Intune admin center, navigate to **Devices** > **Manage Windows 365 Cloud PCs** > **Provision Cloud PCs**.

1. Confirm the **Provisioning policies (Agents)** list is still empty, so no Cloud PC consumption has accrued in this lab.

    ![](./media/ex3-24.png)

1. Navigate to **Microsoft 365 admin center**.
 
    ```
    https://admin.microsoft.com/
    ```
1. In the left navigation pane, select **Copilot (1)**, then select **Cost management (2)**.

1. At the top of the page, select the classic **Billing & usage (3)** link.

    ![](./media/ex3-6.png)

1. Select **Pay-as-you-go-services** tab, and select Windows 365 for agents.

1. Switch its **Connection status (1)** toggle off and click **Save (2)**. Confirm the row now reads **Disconnected** 

    ![](./media/ex3-57.png)

1. Navigate to **billing policies** tab and select your plan **finley-plan-<inject key="DeploymentID" enableCopy="false"/>**.

1. On Details page, select **Delete billing policy**.

    ![](./media/ex3-58.png)

1. Select Delete when prompted.

1. Confirm the plan no longer appears in the list.

    ![](./media/ex3-59.png)

## Conclusion
In this exercise you built the governance layer that must exist before agent compute is safe to grant — all of it before a single device existed.
You began with the decision rather than the mechanics: you confirmed Finley's owner, applied the API-first / CUA / RPA framework to conclude that Contoso's API-less expense portal justifies a desktop, and recorded that conclusion as a custom security attribute so it is queryable rather than remembered. You then made agent compute available — a billing policy bounded by a budget, the pay-as-you-go connection, tenant-level enablement, and an agent user account created through Microsoft Graph — and reviewed the agent pool model without provisioning a machine. Finally you defined a compliance baseline, scoped it to agent devices with a dynamic device group, enforced it with Conditional Access, and turned the meter off.
Three lessons carry forward. First, the **Agent execution environments** condition is not optional polish: without it, a well-meant compliance policy silently breaks every cloud-native agent in the tenant. Second, agent identities and agent user accounts are different objects, and a policy aimed at the wrong one enforces nothing while appearing correct. Third, consumption-billed compute has no natural end, so teardown belongs in the runbook.
Finley now has an identity from Exercise 1, policy from Exercise 2, and governed compute from this exercise. Exercise 4 grounds it in Microsoft 365 context with Microsoft IQ and publishes it to Teams and Microsoft 365 Copilot.

## Review

In this exercise, you have completed the following:

- Established the compute case for the agent and confirmed its accountability
- Created a Windows 365 for Agents billing policy with a budget, and enabled the service for the tenant
- Created an agent user account and reviewed the Cloud PC agent pool configuration model
- Defined the compliance baseline for agent Cloud PCs
- Scoped the baseline to agent devices only
- Required a compliant device for agent sessions and validated the policy
- Applied cost governance and decommissioned agent compute

## Summary

In this exercise, you established a defensible business case for granting an agent desktop compute and recorded it as an auditable custom security attribute. You created a Windows 365 for Agents billing policy backed by an Azure subscription and bounded by a budget, connected the pay-as-you-go service, enabled Windows 365 for Agents for your tenant, and created an agent user account for Finley using Microsoft Graph. You reviewed the Cloud PC agent pool configuration model and the agent session lifecycle without provisioning a machine. You then defined an Intune compliance baseline for agent Cloud PCs, scoped it precisely using an Entra dynamic device group keyed to the enrollment profile name, and enforced it with a Conditional Access policy targeting agent user accounts and conditioned on endpoint-initiated agent sessions — completing the identity, risk, and device triad you began in Exercise 2. Finally, you disconnected the service and deleted the billing policy while retaining the guardrails, demonstrating a clean agent-compute teardown.

Click **Next** from the lower right corner to move on to the next page.

![](./media/next.png)