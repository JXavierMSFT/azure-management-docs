---
title: Troubleshoot your apps faster with App Service using Microsoft Copilot in Azure
description: Learn how Microsoft Copilot in Azure can help you troubleshoot your web apps hosted with App Service.
ms.date: 04/08/2025
ms.topic: how-to
ms.service: copilot-for-azure
ms.custom:
  - build-2024
ms.author: jenhayes
author: JnHs
---

# Troubleshoot your apps faster with App Service using Microsoft Copilot in Azure

Microsoft Copilot in Azure can act as your expert companion for [Azure App Service](/azure/app-service/overview) and [Azure Functions](/azure/azure-functions/functions-overview) diagnostics and solutions.

Azure offers many troubleshooting tools for different types of issues with web apps and function apps. Rather than figure out which tool to use, you can ask Microsoft Copilot in Azure about the problem you're experiencing. Microsoft Copilot in Azure determines which tool is best suited to your question, whether it's related to high CPU usage, networking issues, or other issues. These tools provide diagnostics and suggestions to help you resolve problems you're experiencing.

Copilot in Azure can also help you understand diagnostic information in the Azure portal. For example, when you're looking at the **Diagnose and solve** page for a resource, or viewing diagnostics provided by a troubleshooting tool, you can ask Copilot in Azure to summarize the page, or to explain what an error means.

When you ask Microsoft Copilot in Azure for troubleshooting help, it automatically pulls context when possible, based on the current conversation or the app you're viewing in the Azure portal. If the context isn't clear, you'll be prompted to specify the resource for which you want information.

[!INCLUDE [scenario-note](includes/scenario-note.md)]

>[!Note]
>If you prefer to perform these prompts in your development environment, you can now do so using [GitHub Copilot for Azure (Preview)](/azure/developer/github-copilot-azure/introduction), an extension for Visual Studio Code. Specifically, you can [write prompts to troubleshoot your application in Azure](/azure/developer/github-copilot-azure/troubleshoot-examples) and more.

## Sample prompts

Here are a few examples of the kinds of prompts you can use to access troubleshooting tools and understand diagnostic information. Modify these prompts based on your real-life scenarios, or try additional prompts to get help with different types of issues.

Troubleshooting:

- "My web app is down"
- "Why is my web app slow?"
- "Enable auto heal"
- "High CPU issue"
- "Troubleshoot performance issues with my app"
- "Analyze app latency?"

Understanding available tools:

- "Can I track uptime and downtime of my web app over a specific time period?"
- "Is there a tool that can help me view event logs for my web app?"

Proactive practices:

- "Risk alerts for my app"
- "Are there any best practices for availability for this app?"
- "How can I make my app future-proof"

Summarization and explanation:

- "Give me a summary of these diagnostics."
- "Summarize this page."
- "What does this error mean?"
- "Can you tell me more about the 3rd diagnostic on this page?"
- "What are the next steps to resolve this error?"

## Examples

You can tell Microsoft Copilot in Azure "**my web app is down**." After you select the resource that you want to troubleshoot, Copilot in Azure runs diagnostic checks to identify the cause of the problem.

:::image type="content" source="media/troubleshoot-app-service/app-service-down.png" alt-text="Screenshot of Copilot in Azure responding to a prompt that a web app is down.":::

After the diagnostic checks are finished, Copilot in Azure shares potential causes and solutions for the problem. In some cases, Copilot will suggest a specific diagnostic tool that can help fix the issue.

:::image type="content" source="media/troubleshoot-app-service/app-service-down-results.png" lightbox="media/troubleshoot-app-service/app-service-down-results.png" alt-text="Screenshot of Copilot in Azure sharing a potential solution to a web app problem.":::

While troubleshooting your issues, you can say "**Give me a summary of this page.**" Copilot in Azure summarizes the insights and provides some recommended solutions.

:::image type="content" source="media/troubleshoot-app-service/explain-diagnostics.png" alt-text="Screenshot of Copilot in Azure summarizing the insights and solutions on the Web App Down page." lightbox="media/troubleshoot-app-service/explain-diagnostics.png":::

## Next steps

- Explore [capabilities](capabilities.md) of Microsoft Copilot in Azure.
- Learn more about [Azure App Service](/azure/app-service/overview) and [Azure Functions](/azure/azure-functions/functions-overview).
