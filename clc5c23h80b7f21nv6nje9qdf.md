# When gradle fails or Android's own version of the dependency hell

Today I wanted to share with you a "war story" about a problem I encountered in one of my projects. It has to do with the way gradle versions its the dependency libraries and a poor little programmer trying to fix it one late Friday evening.

## Once upon a time

Our story starts (as many such stories do) with an upgrade to the libraries. The app is experiencing a number of ANRs recently and we've managed to trace the probable cause to an old version of Firebase and Google Play Services libraries, so we decided to upgrade them before the next release (taking advantage of the regression testing that happens before it).

Things are never that simple though - after committing the changes to bitbucket I got back the dreaded `Build failed in Jenkins` email. The most baffling aspect is that for whatever reason Android Studio is able to build and run the app, it's only Jenkins (and the `gradlew clean assembleDebug` command) that fails. The reported error looks cryptic, to say the least:

```java
FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:transformClassesWithJarMergingForDevDebug'.
com.android.build.api.transform.TransformException: java.util.zip.ZipException: duplicate entry: com/google/common/base/FinalizableReference.class

* Try:
Run with --info or --debug option to get more log output.

* Exception is:
org.gradle.api.tasks.TaskExecutionException: Execution failed for task ':app:transformClassesWithJarMergingForDevDebug'.
	at org.gradle.api.internal.tasks.execution.ExecuteActionsTaskExecuter.executeActions(ExecuteActionsTaskExecuter.java:84)
	at org.gradle.api.internal.tasks.execution.ExecuteActionsTaskExecuter.execute(ExecuteActionsTaskExecuter.java:55)
	at org.gradle.api.internal.tasks.execution.SkipUpToDateTaskExecuter.execute(SkipUpToDateTaskExecuter.java:62)
	at org.gradle.api.internal.tasks.execution.ValidatingTaskExecuter.execute(ValidatingTaskExecuter.java:58)
	at org.gradle.api.internal.tasks.execution.SkipEmptySourceFilesTaskExecuter.execute(SkipEmptySourceFilesTaskExecuter.java:88)
	at org.gradle.api.internal.tasks.execution.ResolveTaskArtifactStateTaskExecuter.execute(ResolveTaskArtifactStateTaskExecuter.java:46)
	at org.gradle.api.internal.tasks.execution.SkipTaskWithNoActionsExecuter.execute(SkipTaskWithNoActionsExecuter.java:51)
	at org.gradle.api.internal.tasks.execution.SkipOnlyIfTaskExecuter.execute(SkipOnlyIfTaskExecuter.java:54)
	at org.gradle.api.internal.tasks.execution.ExecuteAtMostOnceTaskExecuter.execute(ExecuteAtMostOnceTaskExecuter.java:43)
	at org.gradle.api.internal.tasks.execution.CatchExceptionTaskExecuter.execute(CatchExceptionTaskExecuter.java:34)
	at org.gradle.execution.taskgraph.DefaultTaskGraphExecuter$EventFiringTaskWorker$1.execute(DefaultTaskGraphExecuter.java:236)
	at org.gradle.execution.taskgraph.DefaultTaskGraphExecuter$EventFiringTaskWorker$1.execute(DefaultTaskGraphExecuter.java:228)
	at org.gradle.internal.Transformers$4.transform(Transformers.java:169)
	at org.gradle.internal.progress.DefaultBuildOperationExecutor.run(DefaultBuildOperationExecutor.java:106)
	at org.gradle.internal.progress.DefaultBuildOperationExecutor.run(DefaultBuildOperationExecutor.java:61)
	at org.gradle.execution.taskgraph.DefaultTaskGraphExecuter$EventFiringTaskWorker.execute(DefaultTaskGraphExecuter.java:228)
	at org.gradle.execution.taskgraph.DefaultTaskGraphExecuter$EventFiringTaskWorker.execute(DefaultTaskGraphExecuter.java:215)
	at org.gradle.execution.taskgraph.AbstractTaskPlanExecutor$TaskExecutorWorker.processTask(AbstractTaskPlanExecutor.java:77)
	at org.gradle.execution.taskgraph.AbstractTaskPlanExecutor$TaskExecutorWorker.run(AbstractTaskPlanExecutor.java:58)
	at org.gradle.internal.concurrent.ExecutorPolicy$CatchAndRecordFailures.onExecute(ExecutorPolicy.java:54)
	at org.gradle.internal.concurrent.StoppableExecutorImpl$1.run(StoppableExecutorImpl.java:40)
Caused by: java.lang.RuntimeException: com.android.build.api.transform.TransformException: java.util.zip.ZipException: duplicate entry: com/google/common/base/FinalizableReference.class
	at com.android.builder.profile.Recorder$Block.handleException(Recorder.java:55)
	at com.android.builder.profile.ThreadRecorder.record(ThreadRecorder.java:104)
	at com.android.build.gradle.internal.pipeline.TransformTask.transform(TransformTask.java:176)
	at org.gradle.internal.reflect.JavaMethod.invoke(JavaMethod.java:73)
	at org.gradle.api.internal.project.taskfactory.DefaultTaskClassInfoStore$IncrementalTaskAction.doExecute(DefaultTaskClassInfoStore.java:163)
	at org.gradle.api.internal.project.taskfactory.DefaultTaskClassInfoStore$StandardTaskAction.execute(DefaultTaskClassInfoStore.java:134)
	at org.gradle.api.internal.project.taskfactory.DefaultTaskClassInfoStore$StandardTaskAction.execute(DefaultTaskClassInfoStore.java:123)
	at org.gradle.api.internal.tasks.execution.ExecuteActionsTaskExecuter.executeAction(ExecuteActionsTaskExecuter.java:95)
	at org.gradle.api.internal.tasks.execution.ExecuteActionsTaskExecuter.executeActions(ExecuteActionsTaskExecuter.java:76)
	... 20 more
Caused by: com.android.build.api.transform.TransformException: java.util.zip.ZipException: duplicate entry: com/google/common/base/FinalizableReference.class
	at com.android.build.gradle.internal.transforms.JarMergingTransform.transform(JarMergingTransform.java:118)
	at com.android.build.gradle.internal.pipeline.TransformTask$2.call(TransformTask.java:185)
	at com.android.build.gradle.internal.pipeline.TransformTask$2.call(TransformTask.java:181)
	at com.android.builder.profile.ThreadRecorder.record(ThreadRecorder.java:102)
	... 27 more
Caused by: java.util.zip.ZipException: duplicate entry: com/google/common/base/FinalizableReference.class
	at com.android.build.gradle.internal.transforms.JarMerger.addJar(JarMerger.java:179)
	at com.android.build.gradle.internal.transforms.JarMergingTransform.transform(JarMergingTransform.java:108)
	... 30 more
```

## The analysis

Gradle looks like it could be quite a successful politician, it managed to spit out 100 lines without saying anything useful. The only thing we have to work with is the `java.util.zip.ZipException: duplicate entry: com/google/common/base/FinalizableReference.class` error. Sadly, googling for this does not offer a solution, but at least it points to the general cause of the problem: it seems that Gradle is trying to put two versions of the same library in the APK. It is up to us however to find out which library it is and how to fix it.

Out first clue comes from the name of the class that is causing the error. `FinalizableReference.java` seems to be a class from the `guava` library (according to [Google](https://google.github.io/guava/releases/19.0/api/docs/com/google/common/base/FinalizableReference.html)). So we at least know which library is the duplicate one.

The next step is to use the `./gradlew app:dependencies` command to investigate the dependency tree of the app. For a mature project such as ours, it contains 17 references of the guava library and the output is a bit hard to follow (all 5434 lines of it). A trick to use here would be to pipe the output of the command to a text file and then open that file in your favourite editor `./gradlew app:dependencies > deps.txt`.

The file will look like a huge dependency tree, which each library listing its own dependencies which in turn have dependencies of their own. Here is a short excerpt:

```java
+--- jp.wasabeef:recyclerview-animators:2.2.6
|    +--- com.android.support:appcompat-v7:25.3.0 -> 25.3.1 (*)
|    \--- com.android.support:recyclerview-v7:25.3.0 -> 25.3.1 (*)
+--- com.facebook.fresco:fresco:1.1.0
|    +--- com.facebook.fresco:drawee:1.1.0
|    |    \--- com.facebook.fresco:fbcore:1.1.0
|    +--- com.facebook.fresco:fbcore:1.1.0
|    \--- com.facebook.fresco:imagepipeline:1.1.0
|         +--- com.parse.bolts:bolts-tasks:1.4.0
|         +--- com.facebook.fresco:fbcore:1.1.0
|         \--- com.facebook.fresco:imagepipeline-base:1.1.0
|              +--- com.parse.bolts:bolts-tasks:1.4.0
|              \--- com.facebook.fresco:fbcore:1.1.0
+--- net.hockeyapp.android:HockeySDK:3.6.2
```

After grabbing a large cup of coffee and sifting through the file for a while I notice that guava appears in several of the configurations, but in general due to two dependencies:

```java
+--- com.google.guava:guava:20.0
...
provided
+--- com.squareup.dagger:dagger-compiler:1.2.2
|    +--- com.squareup.dagger:dagger:1.2.2
|    |    \--- javax.inject:javax.inject:1
|    +--- com.squareup:javapoet:1.7.0
|    \--- com.google.guava:guava:17.0
...
```

So guava is both a direct dependency (our project uses it directly) and requested by dagger through one of its dependencies (poetically named `javapoet`). We also notice the version mismatch between the two versions of the library. We corroborate this by having a quick look at our `.gradle` file:

```java
    provided 'com.squareup.dagger:dagger-compiler:1.2.2'
    compile 'com.squareup.dagger:dagger:1.2.2'
//...
    compile 'com.google.guava:guava:20.0'
```

## Attempted fixes

With that in mind, there are a few things to try to fix this:

* ask gradle to exclude guava when importing dagger compiler:
    

```java
    provided 'com.squareup.dagger:dagger-compiler:1.2.2' {
        exclude group: 'com.google.guava`
    }
```

However, this fails with the following compilation error

```java
Error:(150, 0) Could not find method com.squareup.dagger:dagger-compiler:1.2.2() for arguments [build_2rbnqrh7lpc8m0xgzomg2xtvt$_run_closure2$_closure18@7373d1c2] on object of type org.gradle.api.internal.artifacts.dsl.dependencies.DefaultDependencyHandler.
```

* adjust the requested version of the `guava` library to 17.0 *\- does not work*
    
* upgrade the `dagger` library to 1.2.5 so that it depends on a newer version of guava *\- does not work*
    
* install the `android-apt` gradle plugin and change the scope to `apt` instead of `provided` for the dagger library (which is the correct way of including it anyway) *\- does not work*
    
* remove the explicit import for `guava` *\- does not compile anymore*
    

## The solution

The one thing that worked in the end was changing the scope of the directly requested `guava` dependency from `compile` to `provided`. This instructs gradle to keep the library around while it compiles the files project, but not bundle it in the APK. This works since it `dagger` already brings in a version of the library, we're just removing one source of it. Hence, the result of this two-hours adventure is a simple one-liner:

```java
-    compile 'com.google.guava:guava:20.0'
+    provided 'com.google.guava:guava:20.0'
```

Now, `gradle` should already be doing this automatically, and it indeed did it for the entire duration of the project. What exactly prompted it to stop doing it now will remain one of the mysteries of the universe (though I would love to know your thoughts in the comment if you can explain this).