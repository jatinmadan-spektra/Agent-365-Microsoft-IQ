# Agent 365 & Microsoft IQ: Enterprise AI Governance for M365

### Expected Duration : 4 hours

## Overview

**Contoso Ltd.**, a global IT consultancy, has seen AI agents appear across the business faster than IT can track them. Individual teams have built agents in Copilot Studio, Foundry, and SharePoint. Nobody can answer three basic questions: *how many agents do we have, who is accountable for each one, and what can they reach?*

Contoso has decided to bring every agent under a single control plane using **Microsoft Agent 365**, the enterprise control plane for observing, securing, and governing agents, built on the identity foundation of **Microsoft Entra Agent ID**.

In this lab you take on the role of Contoso's AI Administrator. You will provision a governed agent named **Finley**, an expense-processing agent, and carry that single agent through its entire lifecycle: giving it an enterprise identity, writing and validating the policies that constrain it, provisioning a compliant Cloud PC for it to operate, grounding it in real Microsoft 365 context using **Microsoft IQ**, publishing it to Microsoft Teams through an IT-approved flow, and finally auditing and retiring it.

By the end of this lab you will have governed one agent end to end, using the same tools and controls Contoso would use across thousands.

## Objective

Learn how to govern enterprise AI agents across their full lifecycle. By the end of this lab you will be able to:

- **Provision an agent identity** : Create an agent identity blueprint and agent identity in Microsoft Entra using the Agent 365 CLI, and establish accountability with owners and sponsors.
- **Enforce policy conformance** : Review the Agent 365 default policy template, configure tenant guardrails, and build attribute-driven and risk-based Conditional Access policies for agents.
- **Provision agent compute** : Create a Cloud PC agent pool in Microsoft Intune using Windows 365 for Agents, and require device compliance for agent sessions.
- **Ground an agent in Microsoft IQ** : Extend an agent with Work IQ MCP tools so it can act on real Microsoft 365 context.
- **Deploy through governance** : Publish an agent to Microsoft Teams and Microsoft 365 Copilot with admin approval.
- **Observe and retire** : Trace agent tool calls in Microsoft Defender advanced hunting and run lifecycle actions from the Agent 365 registry.

## Prerequisites

Participants should have:

- **Basic Microsoft 365 administration knowledge**: Familiarity with the Microsoft 365 admin center and license assignment.
- **Basic Microsoft Entra knowledge**: Understanding of users, groups, app registrations, and Conditional Access concepts.
- **Comfort running commands**: You will run a small number of copy-paste PowerShell and CLI commands. No programming is required.
- **Understanding of least privilege**: Familiarity with why agents should be granted only the permissions they need.

>**Note:** All licenses, directory roles, and the Windows 365 for Agents billing plan have been pre-staged in your lab environment. You do not need to purchase or assign anything.

## Architecture

This lab builds a single governed agent across four layers of the Microsoft stack, and each exercise adds one layer.

The **identity layer** (Microsoft Entra Agent ID) issues the agent an enterprise identity derived from an agent identity blueprint, with an owner and a sponsor for accountability. The **governance layer** (Agent 365 in the Microsoft 365 admin center, plus Entra Conditional Access) applies policy templates and access rules that decide what the agent can reach and under what conditions. The **compute layer** (Windows 365 for Agents) provides a compliant, Intune-managed Cloud PC for agents that need to operate a desktop. The **intelligence and surface layer** (Microsoft IQ Work IQ MCP tools, Copilot Studio, and Microsoft Teams) gives the agent real organizational context and a place to do its work, while Microsoft Defender and the Agent 365 registry provide observability and lifecycle control over everything above.

Because one agent flows through all four layers, the policy you write in Exercise 2 is the same policy that governs the Cloud PC in Exercise 3 and the Teams-published agent in Exercise 4.

## Architecture Diagram

![Architecture diagram showing the identity, governance, compute, and intelligence layers of Agent 365](./media/a365-gs-architecture.png)

## Explanation of Components

- **Microsoft Entra Agent ID**: Issues and governs identities for AI agents. Provides agent identity blueprints, agent identities, owners and sponsors, and Conditional Access for agents.
- **Microsoft Agent 365**: The control plane in the Microsoft 365 admin center for observing, securing, and governing every agent in the tenant. Includes the agent registry, agent settings, and policy templates.
- **Agent 365 CLI**: Cross-platform command-line tool that provisions the Azure infrastructure and registers the agent blueprint so the agent is accepted by the Agent 365 platform.
- **Microsoft Intune**: Manages Cloud PCs for Agents through provisioning policies, and supplies the device compliance signal used by Conditional Access.
- **Windows 365 for Agents**: Provides secure, on-demand Cloud PCs with managed identity and a governed agent session lifecycle, for computer-using agents.
- **Microsoft IQ / Work IQ**: The intelligence layer that grounds agents in real-time organizational context. Work IQ MCP servers expose governed tools for Mail, Calendar, Teams, SharePoint, and more.
- **Microsoft Copilot Studio**: Low-code environment for authoring agents and attaching tools.
- **Microsoft Defender XDR**: Provides advanced hunting over agent activity through the `AgentsInfo` table for auditing and investigation.

## Getting Started with the Lab

1. Once the environment is provisioned, a lab guide appears on the right side and the lab virtual machine on the left. Use the lab guide to work through the exercises.

    ![Lab environment showing the lab guide on the right and the virtual machine on the left](./media/a365-gs-01.png)

1. To get the lab environment details, select the **Environment Details** tab. You can also open the **Lab Validation** tab to validate your work as you progress.

    ![Environment Details tab showing lab credentials and resource names](./media/a365-gs-02.png)

1. You can start, stop, and restart the virtual machine from the **Resources** tab if needed.

    ![Resources tab with virtual machine power controls](./media/a365-gs-03.png)

### Signing in to Microsoft 365

1. On the lab virtual machine, open **Microsoft Edge** and navigate to the Microsoft 365 admin center.

    ```
    https://admin.microsoft.com/
    ```

1. On the **Sign in** page, enter the following credentials:

   - **Email/Username:** <inject key="AzureAdUserEmail"></inject>

      ![Microsoft sign-in page with the username field filled in](./media/a365-gs-04.png)

1. Next, provide your password:

   - **Password:** <inject key="AzureAdUserPassword"></inject>

      ![Microsoft sign-in page with the password field filled in](./media/a365-gs-05.png)

1. If prompted with **Stay signed in?**, select **Yes**.

    ![Stay signed in prompt with the Yes button highlighted](./media/a365-gs-06.png)

1. If a **Welcome to Microsoft 365** or first-run dialog appears, close it.

>**Note:** Your lab account has been assigned the Global Administrator, Attribute Definition Administrator, and Attribute Assignment Administrator roles. The last two are required because, by default, Global Administrator alone cannot define or assign custom security attributes.

### Lab naming convention

Throughout this lab you will name resources using a unique suffix so they do not collide with other tenants or participants. Wherever you see **finley-<inject key="DeploymentID" enableCopy="false"/>**, use that exact value including the suffix.

## Support Contact

The CloudLabs support team is available 24/7/365 to help resolve any issues you encounter during the lab.

- **Email Support:** cloudlabs-support@spektrasystems.com
- **Live Chat Support:** https://cloudlabs.ai/labs-support

Now click on **Next** from the lower right corner to move on to the next page.

![Next button in the lower right corner of the lab guide](./media/a365-gs-07.png)
