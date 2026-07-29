# Exercise 3: Windows 365 for Agents

### Estimated Duration: 60 minutes

Some agents only need APIs. Others need to operate a Windows desktop, clicking through applications the way a person would. In this exercise you provision a Cloud PC agent pool for Finley, require device compliance for agent sessions, and then decommission the pool cleanly.

## Overview

**Windows 365 for Agents** provides AI agents with secure, on-demand Cloud PCs that have a managed identity, device posture, and a governed agent session lifecycle. It exists to support **computer-using agents (CUA)**: agents that complete tasks by operating a desktop environment rather than calling an API.

This matters for Contoso because their legacy expense portal has no API. Finley has to open it, log in, and enter data through the interface, exactly as a human would. That requires a machine.

Two concepts to understand before you begin:

- A **provisioning policy (agents)** in Microsoft Intune defines the configuration used to create Cloud PCs for Agents. In Intune, a provisioning policy (agents) *is* a **Cloud PC agent pool**. The policy and the pool are the same object viewed two ways.
- The **Agent execution environments** Conditional Access condition restricts a policy so it only applies when an agent session is initiated **from an endpoint**. This is essential: agents running directly in Microsoft infrastructure have no associated device, so without this condition a device-compliance policy would block them with no possible path to compliance.

>**Note:** Cloud PC provisioning takes **20 to 30 minutes**. Task 3 is deliberately placed to run during that wait. Do not reorder the tasks, or you will spend twenty minutes watching a progress indicator.

>**Note:** Windows 365 for Agents is **consumption-billed**. Cloud PCs continue to incur cost until the provisioning policy is deleted. Task 5 is a mandatory teardown step, not an optional cleanup.

In this exercise you will:

- Verify the Windows 365 for Agents prerequisites and billing plan
- Create a provisioning policy (agents) and assign Finley to it
- Create a Conditional Access policy requiring a compliant device for agent sessions
- Verify the provisioned Cloud PCs and review session usage
- Delete the provisioning policy to stop consumption

## Objectives

- **Task 1**: Verify prerequisites and the Windows 365 for Agents billing plan
- **Task 2**: Create a provisioning policy (agents) in Microsoft Intune
- **Task 3**: Require a compliant device for agent user accounts
- **Task 4**: Verify Cloud PCs and review session usage
- **Task 5**: Delete the provisioning policy (mandatory teardown)

### Task 1: Verify prerequisites and the Windows 365 for Agents billing plan

Three things must be in place before a provisioning policy can be created. If any is missing, the **Provisioning policies (Agents)** option will either be hidden or the policy creation will fail at the billing plan step.

1. On the lab virtual machine, open a new browser tab and navigate to the Microsoft Intune admin center:

    ```
    https://intune.microsoft.com/
    ```

1. If prompted, sign in with the following credentials:

   - **Email/Username:** <inject key="AzureAdUserEmail"></inject>
   - **Password:** <inject key="AzureAdUserPassword"></inject>

1. Confirm the three prerequisites for creating a provisioning policy (agents) are met:

    | Prerequisite | Why it is needed | Pre-staged in this lab |
    | --- | --- | --- |
    | Microsoft Intune license | Provisioning policies and device compliance live in Intune | Yes |
    | Agent 365 license in the tenant | Agents must exist in the Agent 365 platform to be assignable | Yes |
    | Active Windows 365 for Agents billing plan | Cloud PCs are consumption-billed against this plan | Yes |

1. In the left navigation pane, select **Devices (1)**, then under **Device onboarding** select **Provision Cloud PCs (2)**.

    ![Microsoft Intune admin center with Devices and Provision Cloud PCs selected](./media/a365-ex3-t1-01.png)

1. Select the **Provisioning policies (Agents)** tab and confirm it is available.

    ![Provision Cloud PCs page showing the Provisioning policies (Agents) tab](./media/a365-ex3-t1-02.png)

    >**Note:** If this tab is missing, the tenant is missing either the Agent 365 license or the Windows 365 for Agents billing plan. Contact lab support before continuing.

1. Before creating the policy, understand when a Cloud PC is the right choice. Review the distinction below:

   - **Computer-using agent (CUA)** - the agent reasons about a screen and adapts to what it sees. Suited to applications with no API, or interfaces that change.
   - **Robotic process automation (RPA)** - a scripted sequence repeated identically. Suited to stable, high-volume, deterministic tasks.

    >**Note:** Finley is a CUA scenario because Contoso's legacy expense portal has no API and its interface changes between releases. Choosing a Cloud PC for a task that a simple API call could handle wastes money and adds attack surface.

1. Also note the difference between execution modes, because it affects how you design the agent:

   - **Attended** execution means a human is present and can approve or intervene.
   - **Unattended** execution means the agent runs with no human present, on a schedule or in response to an event.

    >**Note:** Unattended agents need stricter Conditional Access, because there is no human to catch a mistake. This is exactly why Task 3 enforces device compliance.

### Task 2: Create a provisioning policy (agents) in Microsoft Intune

Now you create the pool. Think of this as IT filling out an equipment order form: which image, which region, how many machines, and who they are for.

1. On the **Provision Cloud PCs** page, select the **Provisioning policies (Agents)** tab, then select **Create policy**.

    ![Provisioning policies (Agents) tab with the Create policy button highlighted](./media/a365-ex3-t2-01.png)

1. On the **General** page, enter the following values:

   - **Name:** **finley-pool-<inject key="DeploymentID" enableCopy="false"/>**
   - **Description:** **Cloud PC agent pool for the Finley expense-processing agent**
   - **Billing plan:** select the available Windows 365 for Agents billing plan
   - **Always available Cloud PCs count:** **1**
   - **Geography:** select the geography closest to your lab region

    ![General page of the provisioning policy wizard with name and billing plan configured](./media/a365-ex3-t2-02.png)

    >**Note:** The **Always available Cloud PCs count** accepts a value between **1** and **200**. Use **1** for this lab. Each Cloud PC in this count is provisioned immediately and billed continuously, so in production this number should reflect your actual concurrency requirement.

1. Select **Next**.

1. On the **Agents** page, select **Add Agents**.

1. Select the agent **finley-<inject key="DeploymentID" enableCopy="false"/>** that you created in Exercise 1, then select **Save**.

    ![Agents page showing the Finley agent selected for assignment to the pool](./media/a365-ex3-t2-03.png)

    >**Note:** If your agent does not appear in this picker, its blueprint is missing the `managerApplications` property, which happens when a blueprint is created through the Microsoft Entra portal wizard instead of the Agent 365 CLI. Return to Exercise 1 and re-run `a365 setup all`.

1. Select **Next**.

1. On the **Image** page, for **Image type**, select **Gallery image**, then select **Select**, choose an available gallery image, and select **Select** again.

    ![Image page with a gallery image selected](./media/a365-ex3-t2-04.png)

    >**Note:** Gallery images are the default images Microsoft provides. **Custom image** lists images your organization has uploaded through the Add device images workflow, which is how you would pre-install a line-of-business application the agent needs.

1. Select **Next**.

1. On the **Configuration** page, under **Windows settings**, select a **Language & region** value. The matching language pack is installed on every Cloud PC provisioned by this policy.

    ![Configuration page with language and region selected](./media/a365-ex3-t2-05.png)

1. Select **Next**.

1. On the **Scope tags** page, leave the default and select **Next**.

    >**Note:** Scope tags are used in larger organizations to delegate administration, so that a regional IT team can only see and manage the resources tagged for their region.

1. On the **Review + create** page, review your configuration and select **Create**.

    ![Review and create page showing the completed provisioning policy configuration](./media/a365-ex3-t2-06.png)

1. Windows 365 automatically begins provisioning the Cloud PCs. **Provisioning takes approximately 20 to 30 minutes.**

    >**Note:** Do not wait here. Continue directly to Task 3, which is designed to be completed while provisioning runs in the background. You will return to verify the result in Task 4.

### Task 3: Require a compliant device for agent user accounts

While the Cloud PC provisions, you add the control that makes it trustworthy. A Cloud PC for Agents is an Intune-managed Windows device, which means Conditional Access can evaluate its compliance status exactly as it would an employee's laptop.

1. Switch to the Microsoft Entra admin center tab, or navigate to:

    ```
    https://entra.microsoft.com
    ```

1. In the left navigation pane, expand **Entra ID (1)**, select **Conditional Access (2)**, then select **Policies (3)**.

1. Select **+ New policy**.

1. In the **Name** field, enter:

    ```
    CA-Agents-RequireCompliantDevice
    ```

1. Under **Assignments**, select **Users, agents or workload identities**.

1. Under **What does this policy apply to?**, select **Agents**, then under **Include** select **All agent users (Preview)**.

    ![Assignments pane with All agent users selected](./media/a365-ex3-t3-01.png)

    >**Note:** This policy targets **agent users**, not agent identities. These are different objects. An agent's user account is a user-like identity that can hold licenses and participate in collaboration, and device compliance is evaluated against it. A policy targeting agent identities does **not** apply to agent user accounts, and vice versa.

1. Under **Target resources** > **Include**, select **All resources (formerly 'All cloud apps')**.

1. Under **Conditions**, select **Agent execution environments (Preview)** and set **Configure** to **Yes**.

1. Under **Include**, select **Agent user sessions initiated from endpoints**.

    ![Agent execution environments condition configured for endpoint-initiated sessions](./media/a365-ex3-t3-02.png)

    >**Note:** This condition is the most important setting in this task. Device compliance requires Intune enrollment, which today is only supported on Windows 365 Cloud PCs for Agents. Without this condition scoping the policy to endpoint sessions, every agent running in Microsoft cloud infrastructure would be blocked with **no possible path to compliance**, because those agents have no device at all.

1. Under **Access controls** > **Grant**, select **Grant access**, then select **Require device to be marked as compliant**, and select **Select**.

    ![Grant control pane with Require device to be marked as compliant selected](./media/a365-ex3-t3-03.png)

1. Set **Enable policy** to **Report-only**, then select **Create**.

    ![Enable policy toggle set to Report-only before creating the policy](./media/a365-ex3-t3-04.png)

1. Confirm the policy appears in your policy list alongside the two policies you created in Exercise 2.

    ![Conditional Access policies list showing all three agent policies in report-only mode](./media/a365-ex3-t3-05.png)

    >**Note:** You now have three report-only agent policies covering three distinct questions: is the agent approved, is the agent risky, and is the agent running on a compliant device. Together these are the core Zero Trust controls for agents.

1. Optionally, review the related network control. Under **Access controls** > **Grant**, a **Require compliant network** option exists, which uses the Global Secure Access client on the endpoint to supply a network location signal. This requires Microsoft Entra Internet Access.

### Task 4: Verify Cloud PCs and review session usage

By now provisioning should be complete or close to it. In this task you confirm the Cloud PCs exist, are enrolled in Intune, and have sessions available for the agent to consume.

1. Switch to the Microsoft Intune admin center tab.

1. In the left navigation pane, select **Devices**, then select **All devices**.

1. Locate the devices provisioned by your policy. The **device enrollment profile name** matches your provisioning policy name, **finley-pool-<inject key="DeploymentID" enableCopy="false"/>**.

    ![All devices page in Intune showing the provisioned Cloud PC for Agents](./media/a365-ex3-t4-01.png)

    >**Note:** If no devices appear yet, provisioning is still in progress. Wait a few more minutes and refresh. Provisioning Cloud PCs for Agents takes around 20 to 30 minutes from policy creation.

1. Select the provisioned device and review its properties, confirming it is enrolled and managed by Intune. This Intune enrollment is what produces the compliance signal that the Conditional Access policy in Task 3 evaluates.

1. Navigate back to **Devices** > **Provision Cloud PCs** > **Provisioning policies (Agents)**.

1. Select your policy **finley-pool-<inject key="DeploymentID" enableCopy="false"/>**.

1. Locate the **Session Usage** section and review the number of **Available sessions**.

    ![Provisioning policy detail page showing the Session Usage section with available sessions](./media/a365-ex3-t4-02.png)

    >**Note:** Available sessions represent capacity the agent can consume. When an agent starts work it claims a session from the pool; when it finishes, the session is released back. This is the **agent session lifecycle**, and it is why a pool of one Cloud PC can serve many sequential agent tasks.

1. Review the management options available for this pool. You can **edit** the policy to change image or capacity, and **delete** it to decommission the pool entirely.

### Task 5: Delete the provisioning policy (mandatory teardown)

This step is required. Cloud PCs continue to bill against the Windows 365 for Agents consumption plan until the provisioning policy is deleted. Leaving the pool running after the lab generates ongoing cost with no benefit.

1. In the Microsoft Intune admin center, navigate to **Devices** > **Provision Cloud PCs** > **Provisioning policies (Agents)**.

1. Select your policy **finley-pool-<inject key="DeploymentID" enableCopy="false"/>**.

1. Select **Delete**.

    ![Provisioning policy page with the Delete option highlighted](./media/a365-ex3-t5-01.png)

1. Confirm the deletion when prompted.

1. Navigate to **Devices** > **All devices** and confirm the Cloud PCs associated with the policy are being removed.

    ![All devices page confirming the Cloud PCs are being deprovisioned](./media/a365-ex3-t5-02.png)

    >**Note:** Treat this as a governance habit, not a lab formality. Orphaned Cloud PC pools are a recurring source of unexplained cloud spend, in the same way that orphaned virtual machines are. Deleting the provisioning policy is the single action that stops consumption.

## Conclusion

In this exercise you gave Finley a machine to work on and made that machine trustworthy. You confirmed the licensing and billing prerequisites, created a Cloud PC agent pool through an Intune provisioning policy and assigned your agent to it, and used the wait time productively by building a device-compliance Conditional Access policy scoped correctly with the Agent execution environments condition.

Two lessons are worth carrying forward. First, that scoping condition is not optional polish; without it, a well-intentioned compliance policy silently breaks every cloud-native agent in the tenant. Second, consumption-billed infrastructure needs an explicit teardown step in every runbook, because nothing else will stop the meter.

## Review

In this exercise, you have completed the following:

- Verified prerequisites and the Windows 365 for Agents billing plan
- Created a provisioning policy (agents) in Microsoft Intune
- Required a compliant device for agent user accounts
- Verified Cloud PCs and reviewed session usage
- Deleted the provisioning policy

## Summary

In this exercise, you provisioned a Cloud PC agent pool using Windows 365 for Agents, assigned your agent identity to it, and enforced device compliance for endpoint-initiated agent sessions using Conditional Access. You then decommissioned the pool to stop consumption billing. Finley now has governed compute available for computer-using scenarios.

Click **Next** from the lower right corner to move on to the next page.

![Next button in the lower right corner of the lab guide](./media/a365-gs-07.png)
