**gradle版本**

com.android.tools.build:gradle:3.0.1

关键代码：

```gr
apply plugin: 'com.android.application'
```

查看application插件代码，查看gradle-3.0.1.jar中META-INF目录下com.android.application.properties文件

```groovy
implementation-class=com.android.build.gradle.AppPlugin
```

↓

```java
public class AppPlugin extends BasePlugin implements Plugin<Project>{
	....
    @Override
    public void apply(@NonNull Project project) {
        super.apply(project);
    }
}
```

↓

```java
protected void apply(@NonNull Project project) {
    ...
    checkPluginVersion();
    ...
    checkPathForErrors();
    checkModulesForErrors();
    ...
    threadRecorder = ThreadRecorder.get();
    ...
    threadRecorder.record(
    	ExecutionType.BASE_PLUGIN_PROJECT_CONFIGURE,
    	project.getPath(),
    	null,
    	this::configureProject);
    
    threadRecorder.record(
    	ExecutionType.BASE_PLUGIN_PROJECT_BASE_EXTENSION_CREATION,
    	project.getPath(),
    	null,
    	this::configureExtension);
    
    threadRecorder.record(
    	ExecutionType.BASE_PLUGIN_PROJECT_BASE_EXTENSION_CREATION,
    	project.getPath(),
    	null,
    	this::createTasks);
    
}
```

↓

```java
// 配置project，对构建project进行一些初始化
private void configureProject() {
    ...
    // 代码在编译时，需要用到android.jar,这个SdkHandler就是与我们的framework源码进行链接的
	sdkHandler = new SdkHandler(project, getLogger());
    ...
    androidBuilder = new AndroidBuilder(...);
    dataBindingBuilder = new DataBindingBuilder();
    ...
    project.getPlugins().apply(JavaBasePlugin.class);
    //单元测试相关
    project.getPlugins().apply(JacocoPlugin.class);
    project.getTasks().getByName("assemble").setDescription(
                "Assembles all variants of all applications and secondary packages.");
    //清除缓存操作
    ...
}
```

↓

```java
// 配置扩展，根据build.gradler(:app)中android{}结点配置获得一些特殊的配置
private void configureExtension() {
    // BuildType对应build.gradler(:app)中android{}结点buildTyps配置
    final NamedDomainObjectContainer<BuildType> buildTypeContainer = project.container(
                BuildType.class,
                new BuildTypeFactory(instantiator, project, project.getLogger()));
    //ProductFlavor对应android{}结点productFlavors配置
        final NamedDomainObjectContainer<ProductFlavor> productFlavorContainer = project.container(
                ProductFlavor.class,
                new ProductFlavorFactory(instantiator, project, project.getLogger()));
    	//SigningConfig对应android{}结点SigningConfig配置    
    	final NamedDomainObjectContainer<SigningConfig>  signingConfigContainer = project.container(
                SigningConfig.class,
                new SigningConfigFactory(instantiator));
    	final NamedDomainObjectContainer<BaseVariantOutput> buildOutputs = 
            project.container(BaseVariantOutput.class);
    ...
    extension = createExtension(...);
    ...
    variantFactory = createVariantFactory(globalScope, instantiator, androidBuilder, extension);
    taskManager = createTaskManager(globalScope,
                                   project,
                                   projectOptions,
                                   androidBuilder,
                                   extension,
                                   sdkHandler,
                                   ndkHandler,
                                   registry,
                                   threadRecorder);
    variantManager = new VariantManager(
                globalScope,
                project,
                projectOptions,
                androidBuilder,
                extension,
                variantFactory,
    			taskManager,
    			threadRecorder);
    ...
    // create default Objects, signingConfig first as its used by the BuildTypes.
    variantFactory.createDefaultComponents(
             buildTypeContainer, productFlavorContainer, signingConfigContainer);
}
```

↓

查看createExtension方法

```java
protected BaseExtension createExtension(Project project...) {
    return project.getExtensions().create("android", AppExtension.class, project,...);
}
```

↓

查看variantFactory.createDefaultComponents()方法

```java
public class ApplicationVariantFactory implements VariantFactory {
    ...
    @Override
    public void createDefaultComponents(
        @NonNull NamedDomainObjectContainer<BuildType> buildTypes,
        @NonNull NamedDomainObjectContainer<ProductFlavor> productFlavors,
        @NonNull NamedDomainObjectContainer<SigningConfig> signingConfigs) {
        // must create signing config first so that build type 'debug' can be initialized
        // with the debug signing config.
        signingConfigs.create(DEBUG);
        buildTypes.create(DEBUG);
        buildTypes.create(RELEASE);
    }
}
```

↓

```java
// BasePlugin.java
private void createTasks() {
    ThreadRecorder.get().record(ExecutionType.TASK_MANAGER_CREATE_TASKS,
                                new Recorder.Block<Void>() {
                                    @Override
                                    public Void call() throws Exception {
                                        taskManager.createTasksBeforeEvaluate(
                                            new TaskContainerAdaptor(project.getTasks()));
                                        return null;
                                    }
                                },
                                new Recorder.Property("project", project.getName()));

    project.afterEvaluate(new Action<Project>() {
        @Override
        public void execute(Project project) {
            ThreadRecorder.get().record(ExecutionType.BASE_PLUGIN_CREATE_ANDROID_TASKS,
                                        new Recorder.Block<Void>() {
                                            @Override
                                            public Void call() throws Exception {
                                                createAndroidTasks(false);
                                                return null;
                                            }
                                        },
                                        new Recorder.Property("project", project.getName()));
        }
    });
}
```

↓

createAndroidTasks(false)，由于该方法是在project.afterEvaluate方法中执行，说明此时回调的project已经成功配置好了，createAndroidTasks方法需要的各种参数，特别是Variant变体都可以拿到了，这样就可以根据不同的Variant变体创建出不同的task了，比如说默认的两个Variant变体：debug与release，如果有16种变体，那么就会有16种task都会被创建出来。每种Variant变体对应一组tasks，为什么这么做呢？因为每一种Variant变体需要的资源都是不一样的，所以针对每一个Variant变体的task都不能互相共用，所以每一种Variant变体都会有一套完整的构建流程。

可查看Android Studio 右侧Gradle窗口AndroidBuildGradle -> app -> Tasks -> other->assembleFreeDebug、assemblePaidDebug

```java
// BasePlugin.java
final void createAndroidTasks(boolean force) {
    ...
	ThreadRecorder.get().record(ExecutionType.VARIANT_MANAGER_CREATE_ANDROID_TASKS,
                new Recorder.Block<Void>() {
                    @Override
                    public Void call() throws Exception {
                        variantManager.createAndroidTasks();
                        ApiObjectFactory apiObjectFactory = new ApiObjectFactory(
                                androidBuilder, extension, variantFactory, instantiator);
                        for (BaseVariantData variantData : variantManager.getVariantDataList())  {
                            apiObjectFactory.create(variantData);
                        }
                        return null;
                    }
                }, new Recorder.Property("project", project.getName()));
}
```

↓

variantManager.createAndroidTasks();

```java
/**
* Variant/Task creation entry point.
*/
public void createAndroidTasks() {
    variantFactory.validateModel(this);
    variantFactory.preVariantWork(project);

    final TaskFactory tasks = new TaskContainerAdaptor(project.getTasks());
    if (variantDataList.isEmpty()) {
        ThreadRecorder.get().record(ExecutionType.VARIANT_MANAGER_CREATE_VARIANTS,
                                    new Recorder.Block<Void>() {
                                        @Override
                                        public Void call() throws Exception {
                                            populateVariantDataList();
                                            return null;
                                        }
                                    });
    }

    // Create top level test tasks.
    ThreadRecorder.get().record(ExecutionType.VARIANT_MANAGER_CREATE_TESTS_TASKS,
                                new Recorder.Block<Void>() {
                                    @Override
                                    public Void call() throws Exception {
                                        taskManager.createTopLevelTestTasks(tasks, !productFlavors.isEmpty());
                                        return null;
                                    }
                                });

    for (final BaseVariantData<? extends BaseVariantOutputData> variantData : variantDataList) {

        SpanRecorders.record(project, ExecutionType.VARIANT_MANAGER_CREATE_TASKS_FOR_VARIANT,
                             new Recorder.Block<Void>() {
                                 @Override
                                 public Void call() throws Exception {
                                     createTasksForVariantData(tasks, variantData);
                                     return null;
                                 }
                             },
                             new Recorder.Property(SpanRecorders.VARIANT, variantData.getName()));
    }

    taskManager.createReportTasks(variantDataList);
}
```

第一个方法（重点）：populateVariantDataList(); 创建出所有的variant变体

第二个方法（重点）：createTasksForVariantData(tasks, variantData); 根据variant变体创建出一组task

第二个方法：createReportTasks(variantDataList); 创建依赖关系和签名信息的报告

↓

```java
/**
    * Create tasks for the specified variantData.
    */
public void createTasksForVariantData(
        final TaskFactory tasks,
        final BaseVariantData<? extends BaseVariantOutputData> variantData) {

    // Add dependency of assemble task on assemble build type task.
    tasks.named("assemble", new Action<Task>() {
        @Override
        public void execute(Task task) {
            BuildTypeData buildTypeData = buildTypes.get(
                            variantData.getVariantConfiguration().getBuildType().getName());
            task.dependsOn(buildTypeData.getAssembleTask());
        }
    });


    VariantType variantType = variantData.getType();

    createAssembleTaskForVariantData(tasks, variantData);
    if (variantType.isForTesting()) {// 是否为测试task
        ....
    } else {
        taskManager.createTasksForVariantData(tasks, variantData);
    }
}
```

↓

【**重点**】taskManager.createTasksForVariantData(tasks, variantData); 

**各种task创建的具体位置**

```java
/**
 * TaskManager for creating tasks in an Android application project.
 */
public class ApplicationTaskManager extends TaskManager {
    ...
    @Override
    public void createTasksForVariantData(
            @NonNull final TaskFactory tasks,
            @NonNull final BaseVariantData<? extends BaseVariantOutputData> variantData) {
        assert variantData instanceof ApplicationVariantData;
        final ApplicationVariantData appVariantData = (ApplicationVariantData) variantData;

        final VariantScope variantScope = variantData.getScope();

        createAnchorTasks(tasks, variantScope);
        createCheckManifestTask(tasks, variantScope);

        handleMicroApp(tasks, variantScope);

        // Add a task to process the manifest(s)
        ThreadRecorder.get().record(ExecutionType.APP_TASK_MANAGER_CREATE_MERGE_MANIFEST_TASK,
                new Recorder.Block<Void>() {
                    @Override
                    public Void call() {
                        createMergeAppManifestsTask(tasks, variantScope);
                        return null;
                    }
                });

        // Add a task to create the res values
        ThreadRecorder.get().record(ExecutionType.APP_TASK_MANAGER_CREATE_GENERATE_RES_VALUES_TASK,
                new Recorder.Block<Void>() {
                    @Override
                    public Void call() {
                        createGenerateResValuesTask(tasks, variantScope);
                        return null;
                    }
                });

        // Add a task to compile renderscript files.
        ThreadRecorder.get().record(ExecutionType.APP_TASK_MANAGER_CREATE_CREATE_RENDERSCRIPT_TASK,
                new Recorder.Block<Void>() {
                    @Override
                    public Void call() {
                        createRenderscriptTask(tasks, variantScope);
                        return null;
                    }
                });

        // Add a task to merge the resource folders
        ThreadRecorder.get().record(ExecutionType.APP_TASK_MANAGER_CREATE_MERGE_RESOURCES_TASK,
                new Recorder.Block<Void>() {
                    @Override
                    public Void call() {
                        createMergeResourcesTask(tasks, variantScope);
                        return null;
                    }
                });

        // Add a task to merge the asset folders
        ThreadRecorder.get().record(ExecutionType.APP_TASK_MANAGER_CREATE_MERGE_ASSETS_TASK,
                new Recorder.Block<Void>() {
                    @Override
                    public Void call() {
                        createMergeAssetsTask(tasks, variantScope);
                        return null;
                    }
                });

        // Add a task to create the BuildConfig class
        ThreadRecorder.get().record(ExecutionType.APP_TASK_MANAGER_CREATE_BUILD_CONFIG_TASK,
                new Recorder.Block<Void>() {
                    @Override
                    public Void call() {
                        createBuildConfigTask(tasks, variantScope);
                        return null;
                    }
                });

        ThreadRecorder.get().record(ExecutionType.APP_TASK_MANAGER_CREATE_PROCESS_RES_TASK,
                new Recorder.Block<Void>() {
                    @Override
                    public Void call() {
                        // Add a task to process the Android Resources and generate source files
                        createProcessResTask(tasks, variantScope, true /*generateResourcePackage*/);

                        // Add a task to process the java resources
                        createProcessJavaResTasks(tasks, variantScope);
                        return null;
                    }
                });

        ThreadRecorder.get().record(ExecutionType.APP_TASK_MANAGER_CREATE_AIDL_TASK,
                new Recorder.Block<Void>() {
                    @Override
                    public Void call() {
                        createAidlTask(tasks, variantScope);
                        return null;
                    }
                });

        // Add a compile task
        ThreadRecorder.get().record(ExecutionType.APP_TASK_MANAGER_CREATE_COMPILE_TASK,
                new Recorder.Block<Void>() {
                    @Override
                    public Void call() {
                        AndroidTask<JavaCompile> javacTask = createJavacTask(tasks, variantScope);

                        if (variantData.getVariantConfiguration().getUseJack()) {
                            createJackTask(tasks, variantScope);
                        } else {
                            setJavaCompilerTask(javacTask, tasks, variantScope);
                            createJarTask(tasks, variantScope);
                            createPostCompilationTasks(tasks, variantScope);
                        }
                        return null;
                    }
                });

        // Add NDK tasks
        if (isNdkTaskNeeded) {
            ThreadRecorder.get().record(ExecutionType.APP_TASK_MANAGER_CREATE_NDK_TASK,
                    new Recorder.Block<Void>() {
                        @Override
                        public Void call() {
                            createNdkTasks(variantScope);
                            return null;
                        }
                    });
        } else {
            if (variantData.compileTask != null) {
                variantData.compileTask.dependsOn(getNdkBuildable(variantData));
            } else {
                variantScope.getCompileTask().dependsOn(tasks, getNdkBuildable(variantData));
            }
        }
        variantScope.setNdkBuildable(getNdkBuildable(variantData));

        if (variantData.getSplitHandlingPolicy().equals(
                BaseVariantData.SplitHandlingPolicy.RELEASE_21_AND_AFTER_POLICY)) {
            if (getExtension().getBuildToolsRevision().getMajor() < 21) {
                throw new RuntimeException("Pure splits can only be used with buildtools 21 and later");
            }

            ThreadRecorder.get().record(ExecutionType.APP_TASK_MANAGER_CREATE_SPLIT_TASK,
                    new Recorder.Block<Void>() {
                        @Override
                        public Void call() {
                            createSplitResourcesTasks(variantScope);
                            createSplitAbiTasks(variantScope);
                            return null;
                        }
                    });
        }

        ThreadRecorder.get().record(ExecutionType.APP_TASK_MANAGER_CREATE_PACKAGING_TASK,
                new Recorder.Block<Void>() {
                    @Override
                    public Void call() {
                        createPackagingTask(tasks, variantScope, true /*publishApk*/);
                        return null;
                    }
                });

        // create the lint tasks.
        ThreadRecorder.get().record(ExecutionType.APP_TASK_MANAGER_CREATE_LINT_TASK,
                new Recorder.Block<Void>() {
                    @Override
                    public Void call() {
                        createLintTasks(tasks, variantScope);
                        return null;
                    }
                });
    }
}
```



**创建各种Task的位置**

gradle-core-3.0.1-sources.jar

↓

ApplicationTaskManager.java

↓

createTaskForVariantScope 

↓

createBuildConfigTask

↓

通过JavaWriter将一个字符串生成BuildConfig.class

**输出所有Task的名字**

在build.gradle(:app) android {}模块后添加代码，然后build：

```groovy
gradle.taskGraph.whenReady {
    it.allTasks.each { task ->
        println "${task.name} : ${task.class.name - 'Decorated'}"
    }
}
```

**如何debug Gradle代码？**

* 首先，在build.gradle(:app)中通过compileOnly依赖源码
  * compileOnly 'com.android.tools.build:gradle:3.0.1'
* 其次，在Edit Configurations中添加一个Remote模式
* 然后，执行相关命令
  * ./gradlew --no-daemon --no-build-cache -return-tasks -Dorg.gradle.debug=true assembleDebug
* 最后，打好断点，通过Debug模式运行

注意：需要将根目录下build.gradle中使用的版本与导入的源码版本一致

```groovy
dependencies {
    classpath 'com.android.tools.build:gradle:3.0.1'
}
```

**将.class文件编译为DEX文件的Task**

TransformTask，transform的包装，帮我们将class文件转换为dex的tansform实际为DexArchiveBuilderTransform

