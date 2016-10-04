---
title: Spying on Android events without modifying source code
date: 2016-10-03T10:10:46-04:00
---

Let's say you want to intercept events in an Android app, but don't want to modify your source code. As a simple example, you want to add logging whenever the user clicks on any button in your app. I got a bit curious about whether it was possible so I spent some time last week figuring out how this might work. This was extra fun because I've never written and Android app, and it's been years since I did anything in the Java ecosystem.


## What's hard about intercepting events on Android?

There are a few things going on that make this harder to do on Android than on the web. JavaScript and the DOM let you do all kinds of things at runtime—which is the only time! Adding `<script src="blah.js">` can pretty much catch anything you want without modifying any other code.

In Java, once a class is loaded it's fixed. You can do tricksy things to it via reflection, but that's all you can do. You can't get it to call custom code in response to a method call. On the JVM, you can provide a custom class loader and do even trickier things before a class is loaded, including modifying the bytecode. This would allow you to insert the custom code at the start of the method.

But Android doesn't run the JVM, and it doesn't run JVM bytecode. Instead the code is translated to DEX, another bytecode format. In current Android versions, [even that isn't what is actually run][art]. Instead the DEX gets compiled further at install time, this time to native code for the device. This makes the modify-at-load-time approach seem somewhere between daunting and impossible.

So, runtime modification and instrumentation isn't going to work. We can drop back to compile-time instrumentation instead. We could modify either the JVM bytecode before DEX translation, or the DEX bytecode. There's a lot of support for JVM bytecode modification, and JVM bytecode is easier to modify[^ft-jvm-easier] so we'll go with that.

[^ft-jvm-easier]:
    In the JVM's stack-based bytecode, inserted code just needs to ensure it doesn't pop anything off the stack, or leave anything on the stack. Otherwise you can more or less insert any code with net-zero stack change. In the register-based DEX bytecode, some instructions can only work with a specific subset of the available registers, so you'll almost certainly have to move data around and restore it afterwards.

As one last thing that makes things more difficult, the base SDK classes are completely off limits. For example, instrumenting the [base `View` class][view] might be a useful thing to do, but we just can't. They are preloaded in the [zygote], which is sort of the primordial goop of a process that all apps launch from. This is to speed up app launch, and to save on some memory by allowing all apps to share those pages.

[art]: http://source.android.com/devices/tech/dalvik/index.html
[view]: https://developer.android.com/reference/android/view/View.html
[zygote]: https://developer.android.com/topic/performance/memory-overview.html#SharingRAM


## Let's modify the bytecode

So now we know we want to modify JVM bytecode, let's actually do it. There are a bunch of libraries that help with Java bytecode instrumentation. I went with [ASM] for this experiment because I'd heard of it before.

[asm]: http://asm.ow2.org/

We only need to intercept one type of event to prove that this approach works. In this post we'll just look at spying on clicks. There's a [`View.OnClickListener`][onclick] interface that any listener implements. We need to check if a class implements that interface, and if so instrument its `onClick(View)` method[^ft-onclick].

[onclick]: https://developer.android.com/guide/topics/ui/controls/button.html#ClickListener

[^ft-onclick]:
    Tracking clicks on Android turns out to be fairly complicated. There are a bunch of ways a click handler can be registered. I think the most obvious ones are `setOnClickListener`, and setting the name of a method on the `Activity` in the the `android:onclick` XML attribute. The method I'll outline only catches the `setOnClickListener` ones. The XML attirbutes are a bit trickier as they result in using an `OnClickListener` defined in the base SDK's `View` class. That's not something we can instrument. Instead, the way to handle those is to get them from the XML layout files, and instrument the named methods.


ASM presents a streaming view of the contents of a class, and calls methods on classes you define as it encounters the bits of a class file. For example, it calls `visit` with the class name and some other stuff when it starts a new class, and it calls `visitMethod` on each method.

Because we only have this one-way streaming view of a class, we have to we'll need to check the list of interfaces at the start and track whether we're meant to do any work in an instance variable.

~~~
@Override
public void visit(
    int version,
    int access,
    String name,
    String signature,
    String superName,
    String[] interfaces) {

  // Call down the class visitor chain.
  cv.visit(version, access, name, signature, superName, interfaces);

  shouldInstrumentOnClick =
      Arrays.asList(interfaces).contains("android/view/View$OnClickListener");
}
~~~

Later, when we're inspecting a method, we check to see if its name is `onClick`. If so, we make sure to instrument it. The way this works is by returning a modified `MethodVisitor` from `visitMethod`.

~~~
@Override
public MethodVisitor visitMethod(
    int access, String name, String desc, String signature, String[] exceptions) {
  // Get a method visitor from further down the class visitor chain.
  MethodVisitor mv = cv.visitMethod(access, name, desc, signature, exceptions);

  if (shouldInstrumentOnClick && name.equals("onClick")) {
    // Add our method visitor to the chain.
    mv = new LogClickAdapter(mv);
  }
  return mv;
}
~~~

Now the part that actually adds the code! We just want to insert some code at the start of the method, which we can do in `visitCode` in our `MethodVisitor`. ASM calls this library just before it goes through any bytecode instructions, so adding code here puts it at the start of the method. This isn't the nicest thing to look at, as we're mirroring the bytecode we're adding:

~~~
@Override
public void visitCode() {
  mv.visitCode();

  mv.visitLdcInsn("SPY");
  mv.visitTypeInsn(NEW, "java/lang/StringBuilder");
  mv.visitInsn(DUP);
  mv.visitMethodInsn(INVOKESPECIAL, "java/lang/StringBuilder", "<init>", "()V", false);
  mv.visitLdcInsn("saw click on ");
  mv.visitMethodInsn(
      INVOKEVIRTUAL,
      "java/lang/StringBuilder",
      "append",
      "(Ljava/lang/String;)Ljava/lang/StringBuilder;",
      false);
  mv.visitVarInsn(ALOAD, 1);
  mv.visitMethodInsn(
      INVOKEVIRTUAL,
      "java/lang/StringBuilder",
      "append",
      "(Ljava/lang/Object;)Ljava/lang/StringBuilder;",
      false);
  mv.visitMethodInsn(
      INVOKEVIRTUAL, "java/lang/StringBuilder", "toString", "()Ljava/lang/String;", false);
  mv.visitMethodInsn(
      INVOKESTATIC, "android/util/Log", "d", "(Ljava/lang/String;Ljava/lang/String;)I", false);
  mv.visitInsn(POP);
}
~~~

That's the ASM way to write out the bytecode corresponding to this Java code:

~~~
Log.d("SPY", "saw a click on " + view);
~~~

Luckily, there's a nifty utility called ASMifier which outputs the ASM code necessary to generate a class file. Even if you are familiar with JVM bytecode, the ASMified version isn't going to be fun to write, so that's really handy.


## Making it super easy: adding a plugin to the build

Great! Now we can modify classes to add our custom spying code. But the aim here is to make this require as little modification to the application as possible. Android uses Gradle as its standard app build tool, so we can wrap the whole thing up in a Gradle plugin. This would reduce it down to just adding a couple of lines to the `build.gradle` file for the app.

This actually turned out to be the most time consuming part! I already knew enough about JVM bytecode and compilation to make the ASM part fairly straightforward. All I had to do was learn enough about the ASM API to do what I needed to do. For the plugin, I had to get my head around some Gradle architecture, and the overall Android build system so that I could decide where to slot the instrumentation in.

In the end, it turned out that the Android build system folks did a great job of providing an API to transform classes before they get compiled to DEX. This is the well-named [Transform API][transform-api]. After I defined the transform, the plugin's body was pretty much just a call to `registerTransform` on the Android plugin. That handled setting up the Gradle task, its inputs and outputs and whatever else is involved.

[transform-api]: http://google.github.io/android-gradle-dsl/javadoc/current/com/android/build/api/transform/package-summary.html


## What it looks like

This isn't exactly exciting, perhaps unless you're me and you just got it to work. But here's 
 [one of the Android samples][sample] with and without instrumentation.


### Without instrumentation


<center>
<video width="100%" src="/static/android-no-spy.webm" autoplay loop></video>
</center>
<br>


### With instrumentation

As I click on the buttons, it logs lines with `D/SPY` and details on the thing I clicked on.

<center>
<video width="100%" src="/static/android-with-spy.webm" autoplay loop></video>
</center>
<br>

### The diff

The only differences between the two are in the `build.gradle` file:

~~~
diff --git a/Application/build.gradle b/Application/build.gradle
index 990c615..8d05e53 100644
--- a/Application/build.gradle
+++ b/Application/build.gradle
@@ -1,15 +1,18 @@
 
 buildscript {
     repositories {
+        flatDir dirs: "/home/kamal/projects/asm/build/libs"
         jcenter()
     }
 
     dependencies {
+        classpath 'me:plugin:1'
         classpath 'com.android.tools.build:gradle:2.2.0'
     }
 }
 
 apply plugin: 'com.android.application'
+apply plugin: 'track-plugin'
 
 repositories {
     jcenter()
~~~


[sample]: https://developer.android.com/samples/BorderlessButtons/index.html


## Exploring unfamiliar ecosystems can be fun

I had a bunch of fun doing this. For one thing, spying on programs is generally a fun activity. Especially if they don't know you're doing it. I rarely write programs that respond to user input. While I only tested this on a couple of Android samples, I still saw direct feedback from me clicking on things—even if that feedback was just a log line in the Android Studio console.

It was also rewarding to go from here's-an-idea-I-have-no-idea-how-to-implement to totally-working-proof-of-concept in just a few days. Even more so because this was an entire code universe I hadn't really set foot in before. I can write passable Java, but I've never touched a Java build system. My experience with Android is limited to having owned two or three Nexus devices over the years.

This makes me curious about when it's possible to dive into an unfamiliar area and do something non-trivial in a short period of time. I think there has to be a degree of familiarity with at least some of what you're trying to do. In this instance, I wasn't stumbling with Java syntax, and I'd done some stuff with JVM bytecode before. The unfamiliar areas were Android, and the build tooling.

At the other end, when I first tried writing Rust, I was trying to do something I didn't know how to do in a language I didn't know. That turned out to be quite frustrating, and it was hard to make progress. It ended up delaying me learning Rust by a few months, because I gave up on the project and Rust along with it.

This makes me think there's an analog of [‘innovation tokens’][innovation-tokens] at play here. Except instead of optimizing for reliability, we're optimizing for a balance between challenge and fun. I'm really curious to hear other people's experiences, so please [get in touch][me-twitter]!

[innovation-tokens]: http://mcfunley.com/choose-boring-technology
[me-twitter]: https://twitter.com/kamalmarhubi


<br>

---
