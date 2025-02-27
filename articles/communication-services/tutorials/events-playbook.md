---
title: Build a custom event management platform with Microsoft Teams, Graph and Azure Communication Services
titleSuffix: An Azure Communication Services tutorial
description: Learn how to use Microsoft Teams, Graph and Azure Communication Services to build a custom event management platform.
author: ddematheu2
manager: chpalm
services: azure-communication-services

ms.author: dademath
ms.date: 03/31/2022
ms.topic: tutorial
ms.service: azure-communication-services
ms.subservice: teams-interop
---

# Build a custom event management platform with Microsoft Teams, Graph and Azure Communication Services

The goal of this document is to reduce the time it takes for Event Management Platforms to apply the power of Microsoft Teams Webinars through integration with Graph APIs and ACS UI Library. The target audience is developers and decision makers. To achieve the goal, this document provides the following two functions: 1) an aid to help event management platforms quickly decide what level of integration would be right for them, and 2) a step-by-step end-to-end QuickStart to speed up implementation. 

## What are virtual events and event management platforms?

Microsoft empowers event platforms to integrate event capabilities using [Microsoft Teams](https://docs.microsoft.com/microsoftteams/quick-start-meetings-live-events), [Graph](https://docs.microsoft.com/graph/api/application-post-onlinemeetings?view=graph-rest-beta&tabs=http) and [Azure Communication Services](https://docs.microsoft.com/azure/communication-services/overview). Virtual Events are a communication modality where event organizers schedule and configure a virtual environment for event presenters and participants to engage with content through voice, video, and chat. Event management platforms enable users to configure events and for attendees to participate in those events, within their platform, applying in-platform capabilities and gamification. Learn more about[ Teams Meetings, Webinars and Live Events](https://docs.microsoft.com/microsoftteams/quick-start-meetings-live-events) that are used throughout this article to enable virtual event scenarios. 

## What are the building blocks of an event management platform?

Event platforms require three core building blocks to deliver a virtual event experience. 

### 1. Event Scheduling and Management

To get started, event organizers must schedule and configure the event. This process creates the virtual container that event attendees and presenters will enter to interact. As part of configuration, organizers might choose to add registration requirements for the event. Microsoft provides two patterns for organizers to create events:

- Teams Client (Web or Desktop): Organizers can directly create events using their Teams client where they can choose a time and place, configure registration, and send to a list of attendees.

- Microsoft Graph: Programmatically, event platforms can schedule and configure a Teams event on behalf of a user by using their Microsoft 365 license. 

### 2. Attendee experience

For event attendees, they are presented with an experience that enables them to attend, participate, and engage with an event’s content. This experience might include capabilities like watching content, sharing their camera stream, asking questions, responding to polls, and more. Microsoft provides two options for attendees to consume events powered by Teams and Azure Communication Services:

- Teams Client (Web or Desktop): Attendees can directly join events using a Teams Client by using a provided join link. They get access to the full Teams experience.
  
- Azure Communication Services: Attendees can join events through a custom client powered by [Azure Communication Services](https://docs.microsoft.com/azure/communication-services/overview) using [Teams Interoperability](https://docs.microsoft.com/azure/communication-services/concepts/join-teams-meeting). This client can be directly embedded into an Event Platform so that attendees never need to leave the experience. This experience can be built from the ground up using Azure Communication Services SDKs for [calling](https://docs.microsoft.com/azure/communication-services/quickstarts/voice-video-calling/get-started-teams-interop?pivots=platform-web) and [chat](https://docs.microsoft.com/azure/communication-services/quickstarts/chat/meeting-interop?pivots=platform-web) or by applying our low-code [UI Library](https://docs.microsoft.com/azure/communication-services/quickstarts/ui-library/get-started-composites?tabs=kotlin&pivots=platform-web).

### 3. Host & Organizer experience

Event hosts and organizers require the ability to present content, manage attendees (mute, change roles, etc.) and manage the event (start, end, etc.).

- Teams Client (Web or Desktop): Presenters can join using the fully fledged Teams client for web or mobile. The Teams client provides presenters a full set of capabilities to deliver their content. Learn more about [presenter capabilities for Teams](https://support.microsoft.com/office/present-in-a-live-event-in-teams-d58fc9db-ff5b-4633-afb3-b4b2ddef6c0a). 

## Building a custom solution for event management with Azure Communication Services and Microsoft Graph

Throughout the rest of this tutorial, we will focus on how using Azure Communication Services and Microsoft Graph to build a custom event management platform. We will be using the sample architecture below. Based on that architecture we will be focusing on setting up scheduling and registration flows and embedding the attendee experience right on the event platform to join the event.

:::image type="content" source="./media/event-management-platform-architecture.svg" alt-text="Diagram showing sample architecture for event management platform":::

## Leveraging Microsoft Graph to schedule events and register attendees

Microsoft Graph enables event management platforms to empower organizers to schedule and manage their events directly through the event management platform. For attendees, event management platforms can build custom registration flows right on their platform that registers the attendee for the event and generates unique credentials for them to join the Teams hosted event.

>[!NOTE]
>For each required Graph API has different required scopes, ensure that your application has the correct scopes to access the data.

### Scheduling registration-enabled events with Microsoft Graph

1.	Authorize application to use Graph APIs on behalf of service account. This authorization is required in order to have the application use credentials to interact with your tenant to schedule events and register attendees. 

    1. Create an account that will own the meetings and is branded appropriately. This is the account that will create the events and which will receive notifications for it. We recommend to not user a personal production account given the overhead it might incur in the form of remainders.

    1. As part of the application setup, the service account is used to login into the solution once. With this permission the application can retrieve and store an access token on behalf of the service account that will own the meetings. Your application will need to store the tokens generated from the login and place them in a secure location such as a key vault. The application will need to store both the access token and the refresh token. Learn more about [auth tokens](https://docs.microsoft.com/azure/active-directory/develop/access-tokens). and [refresh tokens](https://docs.microsoft.com/azure/active-directory/develop/refresh-tokens).

    1. The application will require "on behalf of" permissions with the [offline scope](https://docs.microsoft.com/azure/active-directory/develop/v2-permissions-and-consent#offline_access) to act on behalf of the service account for the purpose of creating meetings. Individual Graph APIs require different scopes, learn more in the links detailed below as we introduce the required APIs.

    1. Refresh tokens can be revoked in the event of a breach or account termination

  >[!NOTE]
  >Authorization is required by both developers for testing and organizers who will be using your event platform to set up their events.

2.	Organizer logins to Contoso platform to create an event and generate a registration URL. To enable these capabilities developers should use:

      1.	The [Create Calendar Event API](https://docs.microsoft.com/graph/api/user-post-events?view=graph-rest-1.0&tabs=http) to POST the new event to be created. The Event object returned will contain the join URL required for the next step. Need to set the following parameter: `isonlinemeeting: true` and `onlineMeetingProvider: "teamsForBusiness"`. Set a time zone for the event, using the `Prefer` header.

     1.	Next, use the [Create Online Meeting API](https://docs.microsoft.com/graph/api/application-post-onlinemeetings?view=graph-rest-beta&tabs=http) to `GET` the online meeting information using the join URL generated from the step above. The `OnlineMeeting` object will contain the `meetingId` required for the registration steps.

      1.	By using these APIs, developers are creating a calendar event to show up in the Organizer’s calendar and the Teams online meeting where attendees will join.

>[!NOTE]
>Known issue with double calendar entries for organizers when using the Calendar and Online Meeting APIs.

3.	To enable registration for an event, Contoso can use the [External Meeting Registration API](https://docs.microsoft.com/graph/api/resources/externalmeetingregistration?view=graph-rest-beta) to POST. The API requires Contoso to pass in the `meetingId` of the `OnlineMeeting` created above. Registration is optional. You can set options on who can register.

### Register attendees with Microsoft Graph

Event management platforms can use a custom registration flow to register attendees. This flow is powered by the [External Meeting Registrant API](https://docs.microsoft.com/graph/api/externalmeetingregistrant-post?view=graph-rest-beta&tabs=http). By using the API Contoso will receive a unique `Teams Join URL` for each attendee.  This URL will be used as part of the attendee experience either through Teams or Azure Communication Services to have the attendee join the meeting.

### Communicate with your attendees using Azure Communication Services

Through Azure Communication Services, developers can use SMS and Email capabilities to send remainders to attendees for the event they have registered. Communication can also include confirmation for the event as well as information for joining and participating. 
- [SMS capabilities](https://docs.microsoft.com/azure/communication-services/quickstarts/sms/send) enable you to send text messages to your attendees. 
- [Email capabilities](https://docs.microsoft.com/azure/communication-services/quickstarts/email/send-email) support direct communication to your attendees using custom domains.

### Leverage Azure Communication Services to build a custom attendee experience

>[!NOTE]
> Limitations when using Azure Communication Services as part of a Teams Webinar experience. Please visit our [documentation for more details.](https://docs.microsoft.com/azure/communication-services/concepts/join-teams-meeting#limitations-and-known-issues)

Attendee experience can be directly embedded into an application or platform using [Azure Communication Services](https://docs.microsoft.com/azure/communication-services/overview) so that your attendees never need to leave your platform. It provides low-level calling and chat SDKs which support [interoperability with Teams Events](https://docs.microsoft.com/azure/communication-services/concepts/teams-interop), as well as a turn-key UI Library which can be used to reduce development time and easily embed communications. Azure Communication Services enables developers to have flexibility with the type of solution they need. Review [limitations](https://docs.microsoft.com/azure/communication-services/concepts/join-teams-meeting#limitations-and-known-issues) of using Azure Communication Services for webinar scenarios.

1.	To start, developers can leverage Microsoft Graph APIs to retrieve the join URL. This URL is provided uniquely per attendee during [registration](https://docs.microsoft.com/graph/api/externalmeetingregistrant-post?view=graph-rest-beta&tabs=http). Alternatively, it can be [requested for a given meeting](https://docs.microsoft.com/graph/api/onlinemeeting-get?view=graph-rest-beta&tabs=http).

2.	Before developers dive into using [Azure Communication Services](https://docs.microsoft.com/azure/communication-services/overview), they must [create a resource](https://docs.microsoft.com/azure/communication-services/quickstarts/create-communication-resource?tabs=windows&pivots=platform-azp).

3.	Once a resource is created, developers must [generate access tokens](https://docs.microsoft.com/azure/communication-services/quickstarts/access-tokens?pivots=programming-language-javascript) for attendees to access Azure Communication Services. We recommend using a [trusted service architecture](https://docs.microsoft.com/azure/communication-services/concepts/client-and-server-architecture).

4.	Developers can leverage [headless SDKs](https://docs.microsoft.com/azure/communication-services/concepts/teams-interop) or [UI Library](https://azure.github.io/communication-ui-library/) using the join link URL to join the Teams meeting through [Teams Interoperability](https://docs.microsoft.com/azure/communication-services/concepts/teams-interop). Details below:

|Headless SDKs                           | UI Library                            |
|----------------------------------------|---------------------------------------|
| Developers can leverage the [calling](https://docs.microsoft.com/azure/communication-services/quickstarts/voice-video-calling/get-started-teams-interop?pivots=platform-javascript) and [chat](https://docs.microsoft.com/azure/communication-services/quickstarts/chat/meeting-interop?pivots=platform-javascript) SDKs to join a Teams meeting with your custom client | Developers can choose between the [call + chat](https://azure.github.io/communication-ui-library/?path=/docs/composites-meeting-basicexample--basic-example) or pure [call](https://azure.github.io/communication-ui-library/?path=/docs/composites-call-basicexample--basic-example) and [chat](https://azure.github.io/communication-ui-library/?path=/docs/composites-chat-basicexample--basic-example) composites to build their experience. Alternatively, developers can leverage [composable components](https://azure.github.io/communication-ui-library/?path=/docs/quickstarts-uicomponents--page) to build a custom Teams interop experience.|


>[!NOTE]
>Azure Communication Services is a consumption-based service billed through Azure. For more information on pricing visit our resources.


