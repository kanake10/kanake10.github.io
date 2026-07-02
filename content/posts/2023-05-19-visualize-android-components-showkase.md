---
title: "Visualize Android Components with Airbnb's ShowKase."
date: 2023-05-19
tags: ["android", "jetpack-compose", "showkase", "airbnb", "kotlin", "ui"]
categories: ["Android", "UI"]
draft: false
description: "Learn how to use Airbnb's Showkase library to visualize, organize, discover and search Jetpack Compose UI elements in your Android app."
---

Hello guys,hope you all good and welcome back.In today's article we shall learn how to use Airbnb's [Showkase](https://github.com/airbnb/Showkase) library to visualize android components.This is more of a browser in our android app to help engineers organize, discover, search and visualize [Jetpack Compose](https://developer.android.com/jetpack/compose) UI elements. To jump straight to the codebase, check the repo on my [github](https://github.com/kanake10/AirbnbShowkase).

## Setting up Showkase

To set up showkase lib in our android app, we shall be required to add dependencies in our build gradle as follows.

```groovy
// root build.gradle
buildscript {
    dependencies {
        classpath 'com.airbnb.android:showkase-processor:VERSION'
    }
}

// app/build.gradle
dependencies {
    implementation 'com.airbnb.android:showkase:VERSION'
    kapt 'com.airbnb.android:showkase-processor:VERSION'
}
```

In order to visualize our android ui elements created in android app, we are required to add certain annotations provided by the showkase lib.We shall start with annotation used while creating composables.In such a scenario, we use the `@Preview` or `@ShowkaseComposable` annotation. Check this example below.

```kotlin
@ShowkaseComposable(name = "Sample Card", group = "Cards")
@Composable
fun SampleCardPreview() {
    SampleCard(
        title = "Hello Showkase",
        subtitle = "Visualize your components"
    )
}
```

Whilst to search, visualize, discover etc Color properties, we use the `@ShowkaseColor` annotation. Example,

```kotlin
@ShowkaseColor(name = "Primary", group = "Colors")
val PrimaryColor = Color(0xFF6200EE)

@ShowkaseColor(name = "Secondary", group = "Colors")
val SecondaryColor = Color(0xFF03DAC5)
```

And the last property being the TextStyle, we use the `@ShowkaseTypography` annotation. Example,

```kotlin
@ShowkaseTypography(name = "H1", group = "Typography")
val H1 = TextStyle(
    fontSize = 24.sp,
    fontWeight = FontWeight.Bold
)

@ShowkaseTypography(name = "Body", group = "Typography")
val Body = TextStyle(
    fontSize = 14.sp,
    fontWeight = FontWeight.Normal
)
```

Our last usecase while using showkase involves creating a component that supports multiple styles.A good example was provided by Airbnb library documentation where we have a custom button with multiple styles.The library luckily have these extra properties, the `styleName` and `defaultStyle`.The usage for such a circumstance is well captured in this [code](https://github.com/airbnb/Showkase/blob/master/sample/src/main/java/com/airbnb/android/showkasesample/CustomButton.kt).

This the code in general for that,

```kotlin
@ShowkaseComposable(name = "Custom Button", group = "Buttons", styleName = "Primary", defaultStyle = true)
@Composable
fun PrimaryButtonPreview() {
    CustomButton(style = ButtonStyle.PRIMARY)
}

@ShowkaseComposable(name = "Custom Button", group = "Buttons", styleName = "Secondary")
@Composable
fun SecondaryButtonPreview() {
    CustomButton(style = ButtonStyle.SECONDARY)
}
```

Our second last step is adding the Showkase root class to our android app. ie 👇,

```kotlin
@ShowkaseRoot
class MyShowkaseRootModule : ShowkaseRootModule
```

After this, do not forget to rebuild your app, this is critical to make it possible for the library to add packages required and finally call the helper function provided by Showkase in our starter activity, eg,

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                Button(onClick = {
                    startActivity(Showkase.getBrowserIntent(this))
                }) {
                    Text("Open Showkase")
                }
            }
        }
    }
}
```

And thats all guys, thanks for reading this.
The final source code can be found [here](https://github.com/kanake10).By any chance you have a query