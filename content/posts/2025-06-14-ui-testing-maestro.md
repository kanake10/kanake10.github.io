---
title: "UI Testing Made Easy with Maestro: A Guide with a Sample Script"
date: 2025-06-14
tags: ["android", "maestro", "ui-testing", "kotlin", "compose", "yaml"]
categories: ["Android", "Testing"]
draft: false
description: "Maestro simplifies mobile UI testing with readable YAML scripts. Walk through a sample flow that tests core app functionality on Android."
---

Mobile UI testing is often seen as a complex and flaky part of app development, especially when writing instrumentation tests with Espresso. Maestro simplifies this by providing a straightforward way to write and run UI flows using readable YAML scripts. In this post, we'll walk through Maestro basics and break down a sample script that tests the core flow of a fictional app called [Task App](https://github.com/kanake10/android-task).

## What is Maestro?

[Maestro](https://maestro.mobile.dev/) is an open-source mobile UI testing framework designed to be easy to use, fast, and highly maintainable. It runs tests on Android and iOS devices/emulators by interpreting simple YAML instructions that simulate user actions.

## Setting Up Maestro

Before writing your first test, you need to install Maestro.

```bash
# CLI
brew install maestro

# Verify Installation
maestro --version
```

Check out this [doc from Maestro page](https://docs.maestro.dev/getting-started/installing-maestro) on different options for installing Maestro on your device, even for Windows users.

## Writing a Maestro Flow

Maestro test flows are written in `.yaml` files. Here's a breakdown of a sample flow for the app with ID `com.example.vero`.

```yaml
appId: com.example.vero
---
- launchApp
- assertVisible: Task App
- assertVisible: Search Tasks Here…
- assertVisible: Scan QR Code
- assertNotVisible: No internet Connection!!
- assertNotVisible: Retry
- assertVisible: 100 Aufbau

- tapOn: Search Tasks Here…
- inputText: "100 Aufbau"
- assertVisible: 100 Aufbau

- eraseText
- inputText: "nonExistentinputText"
- assertVisible: No tasks found
```

## Step-by-Step Breakdown

Let's walk through what this flow does:

### 1. Launch the app

```yaml
- launchApp
```

Starts the application under test using the given `appId`.

### 2. Check initial UI elements

```yaml
- assertVisible: Task App
- assertVisible: Search Tasks Here…
- assertVisible: Scan QR Code
```

These steps confirm that the essential UI components are visible after launch.

### 3. Ensure error messages are NOT visible

```yaml
- assertNotVisible: No internet Connection!!
- assertNotVisible: Retry
```

Checks that the app is connected and not showing error prompts.

### 4. Confirm task item appears

```yaml
- assertVisible: 100 Aufbau
```

Ensures that a task named "100 Aufbau" is visible by default.

### 5. Search functionality test

```yaml
- tapOn: Search Tasks Here…
- inputText: "100 Aufbau"
- assertVisible: 100 Aufbau
```

Simulates user typing in the search bar and confirms the item is still shown.

### 6. Empty result test

```yaml
- eraseText
- inputText: "nonExistentinputText"
- assertVisible: No tasks found
```

Tests what happens when a user types a term that doesn't match any tasks.

## Running the Test

To run the flow:

```bash
maestro test search_tasks_flow.yaml
```

`search_tasks_flow` is the name of your yaml file.

You can run it on an emulator or real device. Maestro will simulate the flow and print test results in the terminal.

Huge shout-out to the [DuckDuckGo Android app](https://github.com/duckduckgo/Android/tree/develop/.maestro), which still writes Maestro tests using XML layouts. It's a great reference for developers still working with XML-based UIs.

Check out their Maestro flow implementation — it's a perfect guide on how to structure tests with multiple or complex views in the same script for cleaner, more maintainable code.

To dive deeper into Maestro, explore the official documentation and examples here:

- [Maestro + Jetpack Compose UI Tests](https://composables.com/jetpack-compose-tutorials/maestro)

## Compose UI Tests Included!

This repository also includes UI tests written using Jetpack Compose's `createComposeRule`.

📁 Check out those tests [here](https://github.com/kanake10/android-task/tree/master/app/src/androidTest/java/com/example/vero/components).

✍️ A detailed blog post on how to test Jetpack Compose UIs with `composeTestRule` is coming out soon — stay tuned!