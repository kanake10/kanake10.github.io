---
title: "Shoot your Composables with Shot Library."
date: 2023-05-05
tags: ["android", "jetpack-compose", "testing", "shot", "kotlin"]
categories: ["Android", "Testing"]
draft: false
description: "Learn how to test your Jetpack Compose UI components using the Shot screenshot testing library."
---

Hello guys, welcome back .In the previous [article](https://medium.com/@10kanake/screenshot-testing-composable-with-paparazzi-library-e766006d0705), we learnt how we can test our composables using paparazzi library in android development.In this article, weare going to test our android composables using [shot](https://github.com/pedrovgs/Shot) library.A little difference between paparazzi and shot is that shot library uses emulator/device to run while paparazzi does not need emulator.Shot library can also be run in single module app unlike paparazzi.

Keeping this short and sweet, let's jump straight to integrating shot library in our android app.

First thing as usual is adding dependencies to our app and plugins provided by shot library.

```groovy
// root build.gradle
buildscript {
    dependencies {
        classpath 'com.karumi:shot:VERSION'
    }
}

// app/build.gradle
dependencies {
    androidTestImplementation 'com.karumi:shot-android:VERSION'
}
```

```groovy
// app/build.gradle
plugins {
    id 'shot'
}

shot {
    applicationId = 'com.your.app'
}
```

NB: Make sure you configure the instrumentation test runner in your module too .

```groovy
// app/build.gradle
android {
    defaultConfig {
        testInstrumentationRunner "com.karumi.shot.ShotTestRunner"
    }
}
```

Onto the next part which is writing code for our composable.We shall test the same composable we tested earlier with paparazzi in our previous blog.

```kotlin
@Composable
fun SampleCard(
    title: String,
    subtitle: String
) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Icon(
                imageVector = Icons.Default.Star,
                contentDescription = null,
                tint = MaterialTheme.colorScheme.primary
            )
            Spacer(modifier = Modifier.width(12.dp))
            Column {
                Text(text = title, style = MaterialTheme.typography.titleMedium)
                Text(text = subtitle, style = MaterialTheme.typography.bodySmall)
            }
        }
    }
}
```

Since the test we shall be writing needs a device to run, we are going to write these tests in the androidTest folder.

This is how our test code looks

```kotlin
@RunWith(AndroidJUnit4::class)
class SampleCardTest : ScreenshotTest {

    @get:Rule
    val composeRule = createComposeRule()

    @Test
    fun snapshots_sampleCard() {
        composeRule.setContent {
            SampleCard(
                title = "Hello Shot",
                subtitle = "Screenshot testing with device"
            )
        }
        compareScreenshot(composeRule)
    }
}
```

## Whats the next step

The next step is generating the composable using command line.Remember this the same image that shall be used to verify our test,so before generating the particular image make sure you are positive on your designs.This is to ensure that our terminal doesn't shout at us when our test fails, nobody likes dealing with test fails or do you? 😂😂😂

To generate the image, run the following command in your terminal.

```bash
./gradlew executeScreenshotTests -Precord
```

and for verification purposes run the following command

```bash
./gradlew executeScreenshotTests
```

In case your test fails, good thing , shot will show the differences between the two images for easier debugging. Thanks, see you all next time.