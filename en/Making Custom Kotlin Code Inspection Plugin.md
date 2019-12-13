> source:[https://medium.com/@elye.project/making-custom-kotlin-code-inspection-plugin-ec1aedeffa48](https://medium.com/@elye.project/making-custom-kotlin-code-inspection-plugin-ec1aedeffa48)
> author: [Elye](https://medium.com/@elye.project)
> md:[]()

![head](https://miro.medium.com/max/4200/1*A2mpuxHpmmUDQSQk59cAfA.png)
Ever wish to have your write custom code inspection? And have the warning shown even before you compile? Check out the below GIF, where I made a non-CamelCase name on the fly.
![result](https://miro.medium.com/max/1200/1*ih0-iy_BSaloJNBiK0dXKw.gif)

### Code Inspection vs Linting
First of all, let me make sure we have the same term understood.
> Lint — a static code analysis perform during COMPILATION
> Code Inspection — a static code analysis while CODING
So when I say Code Inspection, I literally mean something that check your code while you CODE! So much better than Lint. (though lint does have some advantage, e.g. provide you ability to have a report and also used in Continuous Integration)<br/>

If you are looking to write custom lint for Kotlin code, checkout the below <br/>
[Making Custom Lint for Kotlin Code](https://medium.com/@elye.project/making-custom-lint-for-kotlin-code-8a6c203bf474)
To understand further about Code Inspection you could refers to<br/>

### Let’s start the work
To have code inspection, one would need to make an plugin for Android Studio. The site below is an excellent reference
[Developing Android Studio plugins with Gradle](https://medium.com/groupon-eng/developing-android-studio-plugins-with-gradle-7597d8825d76)

Unfortunately it only show how to print a system out log for the plugin, and it is missing what is needed for Kotlin.

Nonetheless a great reference where I’ll follow the initial step (with some modification) for our Kotlin Code Inspection plugin.
#### Create the below folders
Other than `build.gradle` and `plugin.xml`files, create the folders as shown below. This is because in Android Studio, we can’t create a plugin project. So we have do so manually.
![path](https://miro.medium.com/max/1952/1*q_p15uSQeMNB52-yTrMrYA.png)

#### Create the build.gradle
Below is just getting the right repository and classpath, and I have include what is needed for Kotlin shown in bold.
``` gradle
buildscript {
  ext.kotlin_version = '1.2.50'
  repositories {
    maven {
      url "https://plugins.gradle.org/m2/"
    }
    maven {
      url 'http://dl.bintray.com/jetbrains/intellij-plugin-service'
    }
  }
  dependencies {
    classpath 
      "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
    classpath 
      "gradle.plugin.org.jetbrains:gradle-intellij-plugin:0.1.10"
  }
}
```
Then follow by the needed plugin and dependencies
``` gradle
apply plugin: 'kotlin'
apply plugin: 'org.jetbrains.intellij'

sourceCompatibility = 1.8

repositories {
    mavenCentral()
}

intellij {
    version '2018.1.4'
    pluginName 'Custom Plugin'
    plugins 'kotlin'
    updateSinceUntilBuild false
    //alternativeIdePath "/Applications/Android App Path/"
}

dependencies {
    testCompile group: 'junit', name: 'junit', version: '4.12'
    implementation
      "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
}

group 'com.personal'
version '1.0-SNAPSHOT'
```
> Note: we have alterantiveIdePath there commented. With it commented, when you run, it will open up the IntelliJ Sandbox to test your plugin. If you uncomment it out, and put your Android App’s path there, it will open up Android Studio instead as your Sandbox to test your plugin.

#### Create plugin.xml
This is used to configure your plugin
``` xml
<idea-plugin>
    <id>com.custom.plugin</id>
    <name>Custom Plugin</name>
    <vendor email="test@example.com" 
              url="http://www.example.com">Example</vendor>

    <description><![CDATA[
      Demo plugin written in Kotlin for Kotlin syntax check to 
      ensure camelcase naming and also a notification plugin
    ]]></description>

    <change-notes><![CDATA[
      Release notes : Camelcase naming check and Notification plugin
    ]]>
    </change-notes>

    <depends>org.jetbrains.kotlin</depends>
    <!-- Code Inspection Component  -->
    <extensions defaultExtensionNs="com.intellij">
        <inspectionToolProvider implementation=
            "com.plugin.inspection.CustomInspectionProvider"/>
    </extensions>

    <depends>com.intellij.modules.lang</depends>
    <!-- Notification Component  -->
    <project-components>
        <component>
            <implementation-class>
                com.plugin.inspection.CustomNotificationComponent
            </implementation-class>
        </component>
    </project-components>

</idea-plugin>
```
From here, you could see there’s two components I add to my plugin
- Notification Component.
- Code Inspection Component.
I will describe the two components below

### Notification Component
This is just to show that the plugin is loaded. It’s a simple notification as below
![notification](https://miro.medium.com/max/1476/1*o4_2lvkWZ9JaMchk04LYhg.png)
I extracted this code sample from the below.
[Android Studio Creating Plugins Easy](http://myhexaville.com/2017/03/06/android-studio-creating-plugins-easy/?source=post_page-----ec1aedeffa48----------------------)
> Note: Don’t follow the entire step in the blog above, as it start off downloading the entire IntelliJ source and build from there.

I change it to Kotlin version of it though
``` kotlin
class CustomNotificationComponent : ProjectComponent {

    override fun projectOpened() {}

    override fun projectClosed() {}

    override fun initComponent() {
        ApplicationManager.getApplication()
             .invokeLater({
                  Notifications.Bus.notify(NOTIFICATION_GROUP.value
                      .createNotification(
                          "Testing Personal Plugin",
                          "Check for Kotlin non CamelCase usage",
                          NotificationType.INFORMATION,
                          null))
             }, ModalityState.NON_MODAL)
    }

    override fun disposeComponent() {}

    override fun getComponentName(): String {
        return CUSTOM_NOTIFICATION_COMPONENT
    }

    companion object {
        private const val CUSTOM_NOTIFICATION_COMPONENT = 
            "CustomNotificationComponent"
        private val NOTIFICATION_GROUP = object : 
             NotNullLazyValue<NotificationGroup>() {
               override fun compute(): NotificationGroup {
                 return NotificationGroup(
                        "Motivational message",
                        NotificationDisplayType.STICKY_BALLOON,
                        true)
            }
        }
    }
}
```
### Code Inspection Component
This is the main part of the plugin that will perform the inspection.

#### The Provider
First of all, we need to have a provider class that provides a list of custom inspection classes.

``` kotlin
class CustomInspectionProvider : InspectionToolProvider {
    override fun getInspectionClasses(): Array<Class<*>> {
        return arrayOf(CamelcaseInspection::class.java)
    }
}
```
#### The Inspection (in Java)
The IntelliJ did have some initial guide of how to write it as below
[Code Inspections](https://www.jetbrains.org/intellij/sdk/docs/tutorials/code_inspections.html)

To see how it is implemented, there’s quite a bit of example on the Android code found in this [link](https://android.googlesource.com/platform/tools/adt/idea/+/studio-2.2/android/src/org/jetbrains/android/inspections/lint)<br/>

Unfortunately they are all in Java, and doesn’t show how to get the needed PSI (Program Structure Interface) for Kotlin.<br/>

#### The Inspection (in Kotlin)
For Kotlin, we’ll need to have access to AbstractKotlinInspection.<br/>
> I search the entire [Kotlin Library](https://mvnrepository.com/artifact/org.jetbrains.kotlin?p=3), but can’t find it easily. Later found that we could get that included by adding **plugins ‘kotlin’** in the intellij section of build.gradle.

Using that `AbstractKotlinInspection` class, we could easily get access to various method that extract your Kotlin code for analyzing it.<br/>

In my case, I’m accessing all the name extracted, and check if it is CamelCase to decide if we could register it as a problem.<br/>

In my case, I’m accessing all the name extracted, and check if it is CamelCase to decide if we could register it as a problem.<br>

``` kotlin
class CamelcaseInspection : AbstractKotlinInspection() {

    override fun getDisplayName(): String {
        return "Use CamelCase naming"
    }

    override fun getGroupDisplayName(): String {
        return GroupNames.STYLE_GROUP_NAME
    }

    override fun getShortName(): String {
        return "Camelcase"
    }

    override fun buildVisitor(holder: ProblemsHolder, 
        isOnTheFly: Boolean): KtVisitorVoid {
        return namedDeclarationVisitor { declaredName ->
            if (declaredName.name?.isDefinedCamelCase() == false) {
                System.out.println(
                    "Non CamelCase Name Detected for 
                         ${declaredName.name}")
                holder.registerProblem(
                    declaredName.nameIdentifier as PsiElement, 
                    "Please use CamelCase for #ref #loc")
            }
        }
    }

    private fun String.isDefinedCamelCase(): Boolean {
        val toCharArray = toCharArray()
        return toCharArray
                .mapIndexed { index, current -> 
                     current to toCharArray.getOrNull(index + 1) }
                .none { it.first.isUpperCase() && 
                     it.second?.isUpperCase() ?: false }
    }

    override fun isEnabledByDefault(): Boolean {
        return true
    }
}
```
Refer to the above code, would give you some idea of how to write an Inspection Code in Kotlin.<br/>
For more code reference, you could check the below link (actual Kotlin code inspection codes)<br/>
[kotlin](https://github.com/JetBrains/kotlin/tree/master/idea/src/org/jetbrains/kotlin/idea/inspections)

In it also, other than just detecting the issue, you could also code to fix the issue. Check the code out!<br>

#### The inspection description
The description of the inspection issue detected could is defined through XML file.

Under the `resources` folder, create `inspectionDescriptions` folder.

Then create a html file with the name that is name after the Custom Inspection you made, which is in our case `CamelCase.html`

In it, just write the description of the Inspection message as below.

``` html
<html>
<body>
The name detected is not in CamelCase format. <br>
Consider changing it to CamelCase format.
</body>
</html>
```
The description of official Kotlin code inspection could be found in 
[kotlin](https://github.com/JetBrains/kotlin/tree/master/idea/resources/inspectionDescriptions)


### Deploying the Plugin
Now with the above, you should be all set for deploying.
#### Sandbox testing
Once you get all that, to deploy it on Sandbox and test it out, just type the following command
``` sh
./gradlew runIdea
```
It will launch the sandBox IntelliJ that you could check out if your plugin works.
If you like to launch other IDE (e.g. Android Studio), just use the `alternativeIdePath` as mentioned above.

#### Applying on actual Android Studio.
To apply on Android Studio, follow the following steps
- 1. run ./gradlew buildPlugin
- 2. Go to Android Studio->Preferences->Plugin->Install plugin from disk
- 3. Go to your plugin project’s build->distributions
- 4. You’ll find a zip file there, and you could install it.

#### Publishing to IntelliJ Plugin Repository
If you like to share with everyone, you could use the command
``` sh
./gradlew publishPlugin
```
You’ll need to have your account ready though, and follow the tutorial below
[Publishing a plugin](https://www.jetbrains.org/intellij/sdk/docs/basics/getting_started/publishing_plugin.html)
### Some hiccups along the way
The above shared the common flow. Along the way, I do encountered issue, which is shared below.
#### 1. bash: ./gradlew: Permission denied
When trying to run `./gradlew` for the first time after creating it manually, it doesn’t allow you. Just change the permission as below
``` sh
chmod 777 ./gradlew
```
#### 2. Need to have one Java Class
If your entire plugin code is in Kotlin, it will fail as below
``` 
* What went wrong:
Execution failed for task ‘:classes’.
> destination directory “/demo_kotlin_inspection_plugin/build/classes/java/main” does not exist or is not a directory
```
Just create a dummy Java class, or make one class as Java class instead.
#### 3. Some odd issue
If you do encounter some odd issue, just perform a `./gradlew clean`, and remove the build folder entire. Then rerun it again.

> The issue I got is, my inspectionDescriptions is not showing, due to initial error. After correcting it, it is still not okay until I clear my build and reset everything.

You could get my working code here
[demo_android_kotlin_custom_inspection_plugin](https://github.com/elye/demo_android_kotlin_custom_inspection_plugin)

Hopes this provides you some good pointer on writing your own custom Inspection rules for Kotlin. Let me know if you publish anything up on IntelliJ Plugin Repository one day. Would love to know this blog helps people contribute better to everyone.
I hope this post is helpful to you. You could check out my other interesting topics [here](https://medium.com/@elye.project).











