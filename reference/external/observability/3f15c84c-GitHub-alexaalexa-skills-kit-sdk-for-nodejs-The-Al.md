---
title: GitHub - alexa/alexa-skills-kit-sdk-for-nodejs: The Alexa Skills Kit SDK for Node.js helps you get a skill up and running quickly, letting you focus on skill logic instead of boilerplate code.
source: https://github.com/alexa/alexa-skills-kit-sdk-for-nodejs
kind: external
domain: observability
author: Alexa
original_date: 2016-06-24
fetched_at: 2026-05-16
bookmark_title: alexa/alexa-skills-kit-sdk-for-nodejs: The Alexa Skills Kit SDK for Node.js helps you get a skill up and running quickly, letting you focus on skill logic instead of boilerplate code.
tags: [external, observability]
---

> [!info] 外部文章 · 自动导入
> 来源：[github.com](https://github.com/alexa/alexa-skills-kit-sdk-for-nodejs)
> 作者：Alexa
> 原始日期：2016-06-24
> 抓取日期：2026-05-16

# GitHub - alexa/alexa-skills-kit-sdk-for-nodejs: The Alexa Skills Kit SDK for Node.js helps you get a skill up and running quickly, letting you focus on skill logic instead of boilerplate code.

The **ASK SDK v2 for Node.js** makes it easier for you to build highly engaging skills by allowing you to spend more time on implementing features and less on writing boilerplate code.

The **ASK SDK Controls Framework (Beta)** builds on the ASK SDK v2 for Node.js, providing a scalable solution for creating large, multi-turn skills in code with reusable components called *controls*.

The **ASK SMAPI SDK for Node.js** provides developers a library for easily interacting with all Skill Management APIs (SMAPI), including interaction model, intent request history, and in-skill purchasing APIs.

| Package | NPM |
|---|---|
| ask-sdk | |
| ask-sdk-core | |
| ask-sdk-dynamodb-persistence-adapter | |
| ask-sdk-runtime | |
| ask-sdk-s3-persistence-adapter | |
| ask-sdk-v1adapter | |
| ask-sdk-express-adapter | |
| ask-smapi-sdk | |
| ask-sdk-local-debug |

- Amazon Pay
- Audio Player
- Display – Body templates for devices with a screen
- Gadgets Game Engine – Echo Buttons
- Directive Service (Progressive Response)
- Messaging
- Monetization
- Video
- Device Address
- Lists
- Request for customer contact information
- Obtain customer settings information
- Account Linking
- Entity Resolution
- Dialog Management
- Location Services
- Reminders

The following features are released as public preview. The interfaces might change in future releases.

The SDK works on model classes rather than native Alexa JSON requests and responses. These model classes are generated using the Request, Response JSON schemas from the developer docs. The source code for the model classes can be found here.

Sample that familiarizes you with the Alexa Skills Kit and AWS Lambda by allowing you to hear a response from Alexa when you trigger the sample.

Sample that familiarizes you with the Controls framework, allowing you to hear a response from Alexa when you trigger the skill.

Template for a basic fact skill. You’ll provide a list of interesting facts about a topic, Alexa will select a fact at random and tell it to the user when the skill is invoked.

Template for a parameter-based skill called 'Minecraft Helper'. When the user asks how to craft an item in the game Minecraft, the skill provides instructions.

Template for a trivia-style game with score keeping. Alexa asks the user multiple-choice questions and seeks a response. Correct and incorrect answers to questions are recorded.

Template for a basic quiz game skill. Alexa quizzes the user with facts from a list you provide.

Template for a local recommendations skill. Alexa uses the data that you provide to offer recommendations according to the user's stated preferences.

Sample skill that matches the user with a pet. Alexa prompts the user for the information it needs to determine a match. Once all of the required information is collected, the skill sends the data to an external web service that processes the data and returns the match.

Template for a basic high-low game skill. When the user guesses a number, Alexa tells the user whether the number she has in mind is higher or lower.

Template for a basic decision tree skill. Alexa asks the user a series of questions to get to a career suggestion.

Build a multi-modal grocery shopping skill using custom and library controls for item lists, shopping cart management, and checkout.

Sample skill that shows how to request and access the configured address in the user’s device settings.

Project that demonstrates how to use the audio player for skills.

Alexa Skills Kit SDK for Python

Request and vote for Alexa features here!