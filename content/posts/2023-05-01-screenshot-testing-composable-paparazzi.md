---
title: "Screenshot Testing Composables with Paparazzi Library 📸"
date: 2023-05-01
tags: ["android", "jetpack-compose", "testing", "paparazzi", "kotlin"]
categories: ["Android", "Testing"]
draft: false
description: "Learn how to test your Jetpack Compose UI components using the Paparazzi screenshot testing library by CashApp — no emulator needed."
---

Hello guys, in today's article we are going to see how we can test our composable in Android development using the [Paparazzi](https://github.com/cashapp/paparazzi) library by CashApp.

## Pre-requisite

Paparazzi does not run on monolith applications and hence we shall have to modularise our app.

Our first step will be creating a module in Android Studio. In this module we are going to create the composable we shall test. Before this, remember to add the Paparazzi dependency to your root module inclusive of the Paparazzi plugin.

> **NB:** Paparazzi plugin is added to the module with composables to be tested.

```groovy
classpath "app.cash.paparazzi:paparazzi-gradle-plugin:VERSION"
```

## Creating the Composable

Onto the next step which is creating our composable. Our view will be a simple card with two text views and one Icon.

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

## Testing the Composable

Finally to the interesting bit — testing the composable. Since Paparazzi does not require an emulator, we are going to write our code in the **test** folder of the module with our composable.

```kotlin
class SampleCardTest {

    @get:Rule
    val paparazzi = Paparazzi(
        deviceConfig = DeviceConfig.PIXEL_5,
        theme = "android:Theme.Material.Light.NoActionBar"
    )

    @Test
    fun snapshot_sampleCard() {
        paparazzi.snapshot {
            SampleCard(
                title = "Hello Paparazzi",
                subtitle = "Screenshot testing made easy"
            )
        }
    }
}
```

## Running the Tests

Paparazzi provides commands to run in order to take a snapshot of your composable and also to verify if the images match.

**Generate snapshots:**
```bash
./gradlew recordPaparazziDebug
```

**Verify your composable hasn't broken any tests:**
```bash
./gradlew verifyPaparazziDebug
```

After generating our snapshots, Android Studio will create a `snapshots/` folder with an image of the composable. This is the image that will be used as the reference while verifying future changes.

## Take Away 🥡

In the JVM rule, we can specify a specific device you want to use, a default theme, and more. In such cases the **TestParameterInjector** library by Google comes in handy.

```kotlin
@RunWith(TestParameterInjector::class)
class SampleCardParameterizedTest {

    @get:Rule
    val paparazzi = Paparazzi()

    @Test
    fun snapshot_devices(
        @TestParameter deviceConfig: MyDeviceConfig
    ) {
        paparazzi.unsafeUpdateConfig(deviceConfig = deviceConfig.value)
        paparazzi.snapshot {
            SampleCard(title = "Test", subtitle = "Parameterized")
        }
    }
}
```

Have a read of the [TestParameterInjector docs](https://github.com/google/TestParameterInjector) for more.

It's also possible to create a helper function where you can render your components — for example, in a Box with certain dimensions — and replace the `.snapshot{}` Paparazzi function with it.

---

## Further Reading

- [Sanely Test Your Android UI Libraries with Paparazzi](https://betterprogramming.pub/sanely-test-your-android-ui-libraries-with-paparazzi-b6d46c55f6b0)
- [Snapshot Testing and More with Paparazzi — Droidcon](https://www.droidcon.com/2022/09/29/snapshot-testing-and-more-with-paparazzi/)
- [How Screenshot Tests Elevate Our Android Testing Strategy — GetYourGuide](https://www.getyourguide.careers/posts/how-screenshot-tests-elevate-our-android-testing-strategy)
- [An Introduction to Effective Snapshot Testing on Android — Droidcon](https://www.droidcon.com/2021/11/17/an-introduction-to-effective-snapshot-testing-on-android-2/)