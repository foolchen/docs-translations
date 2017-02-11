# 建立Instrumented单元测试

Instrumented单元测试是运行在物理设备或模拟器上的测试,它们可以使用Android框架的API和Support Library中相关的测试用API.如果单元测试需要访问一些真实的信息(例如目标应用程序的[Context](https://developer.android.com/reference/android/content/Context.html)),或者需要真实的Android框架组件的实现(如[Parcelable](https://developer.android.com/reference/android/os/Parcelable.html)或者[SharedPreferences](https://developer.android.com/reference/android/content/SharedPreferences.html)对象)时,则应该创建Instrumented单元测试.

> 使用Instrumented单元测试,还有助于减少编写和维护模拟代码所需的工作量.同时,你仍然可以在需要的时候使用模拟框架来模拟任何依赖关系.

## 配置您的测试环境

在您的Android Studio项目中,必须将本地单元测试的源文件保存在`*module-name*/src/androidTest/java/`目录.当您在创建新项目时,该目录已自动创建.

在开始之前,您应该首先[设置测试支持库](https://developer.android.com/tools/testing-support-library/index.html#setup),它提供的API允许您为您的应用程序快速构建和运行测试代码.测试之处库包括JUnit 4测试运行器([AndroidJUnitRunner](https://developer.android.com/tools/testing-support-library/index.html#AndroidJUnitRunner))和UI相关功能测试的API([Espresso](https://developer.android.com/tools/testing-support-library/index.html#Espresso) and [UI Automator](https://developer.android.com/tools/testing-support-library/index.html#UIAutomator)).

你还应该为您的项目增加这些测试支持库的依赖,以使用它提供的测试运行期和标准API.为了简化测试代码的编写,您可能还需要添加[Hamcrest](https://github.com/hamcrest)库的依赖,它允许您使用Hamcrest mathcer API创建更灵活的断言.

在您的应用程序的顶级`build.gradle`文件中,您需要添加如下依赖:

```groovy
dependencies {
    androidTestCompile 'com.android.support:support-annotations:24.0.0'
    androidTestCompile 'com.android.support.test:runner:0.5'
    androidTestCompile 'com.android.support.test:rules:0.5'
    // Optional -- Hamcrest library
    androidTestCompile 'org.hamcrest:hamcrest-library:1.3'
    // Optional -- UI testing with Espresso
    androidTestCompile 'com.android.support.test.espresso:espresso-core:2.2.2'
    // Optional -- UI testing with UI Automator
    androidTestCompile 'com.android.support.test.uiautomator:uiautomator-v18:2.1.2'
}
```

>**注意:**如果您添加了`support-annotations`库的`compile`依赖,并且添加了`expresso-core`库的`androidTestCompile`依赖,在构建的时候可能会出现依赖冲突.要解决这个问题,你应该讲`expresso-core`的依赖更新如下:
>
>```groovy
>androidTestCompile('com.android.support.test.espresso:espresso-core:2.2.2', {
>    exclude group: 'com.android.support', module: 'support-annotations'
>})
>```

要使用JUnit 4测试类,请确保在项目中将[AndroidJUnitRunner](https://developer.android.com/reference/android/support/test/runner/AndroidJUnitRunner.html)指定为默认Instrumentation运行器,方法是在项目的模块级`build.gradle`文件中增加如下配置:

```groovy
android {
    defaultConfig {
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
    }
}
```

## 创建一个Instrumented单元测试类

您的Instrumented单元测试的定义方式应与JUnit 4测试类的定义方式一致.要了解有关创建JUnit 4测试类和使用JUnit 4断言和注解的更多信息,请参考[建立本地单元测试](建立本地单元测试.md).

要创建Instrumented JUnit 4测试类,需要在定义测试类时添加类注解`@RunWith(AndroidJUnit4.class)`.同样,您还需要指定Android测试支持库中提供的[`AndroidJUnitRunner`](https://developer.android.com/reference/android/support/test/runner/AndroidJUnitRunner.html)类作为默认的测试运行器.具体的配置步骤请参考[Getting Started with Testing](https://developer.android.com/training/testing/start/index.html#run-instrumented-tests).

以下示例演示了如何编写一个Instrumented单元测试来验证`LogHistory`类是否正确实现了[Parcelable](https://developer.android.com/reference/android/os/Parcelable.html)接口:

```java
import android.os.Parcel;
import android.support.test.runner.AndroidJUnit4;
import android.util.Pair;
import org.junit.Test;
import org.junit.runner.RunWith;
import java.util.List;
import static org.hamcrest.Matchers.is;
import static org.junit.Assert.assertThat;

@RunWith(AndroidJUnit4.class)
@SmallTest
public class LogHistoryAndroidUnitTest {

    public static final String TEST_STRING = "This is a string";
    public static final long TEST_LONG = 12345678L;
    private LogHistory mLogHistory;

    @Before
    public void createLogHistory() {
        mLogHistory = new LogHistory();
    }

    @Test
    public void logHistory_ParcelableWriteRead() {
        // Set up the Parcelable object to send and receive.
        mLogHistory.addEntry(TEST_STRING, TEST_LONG);

        // Write the data.
        Parcel parcel = Parcel.obtain();
        mLogHistory.writeToParcel(parcel, mLogHistory.describeContents());

        // After you're done with writing, you need to reset the parcel for reading.
        parcel.setDataPosition(0);

        // Read the data.
        LogHistory createdFromParcel = LogHistory.CREATOR.createFromParcel(parcel);
        List<Pair<String, Long>> createdFromParcelData = createdFromParcel.getData();

        // Verify that the received data is correct.
        assertThat(createdFromParcelData.size(), is(1));
        assertThat(createdFromParcelData.get(0).first, is(TEST_STRING));
        assertThat(createdFromParcelData.get(0).second, is(TEST_LONG));
    }
}
```

### 创建测试套件(Test Suite)

为了同时执行多条相关测试用例,你可以在测试套件类中对一组测试类进行分组,并且让这些测试类一起运行.测试套件可以再嵌套测试套件,您的测试套件可以将其他测试套件分组,并将这些测试套件一直运行.

测试套件位于测试包中,就像主应用程序包.按照惯例,测试套件包名通常以`.suite`结尾(例如:`com.example.android.testing.mysample.suite`).

要为单元测试创建测试套件,您需要导入[`RunWith`](http://junit.sourceforge.net/javadoc/org/junit/runner/RunWith.html)和[`Suite`](http://junit.sourceforge.net/javadoc/org/junit/runners/Suite.html)两个类.在定义测试套件类时,需要添加`@RunWith(Suite.class)`和`@Suite.SuiteClasses()`注解.在`@Suite.SuiteClasses()`注解中,需要将测试类或者测试套件类作为参数.

以下实例展示了如何实现名为`UnitTestSuite`的测试套件,它将组合运行`CalculatorInstrumentationTest`和`CalculatorAddParameterizedTest`:

```java
import com.example.android.testing.mysample.CalculatorAddParameterizedTest;
import com.example.android.testing.mysample.CalculatorInstrumentationTest;
import org.junit.runner.RunWith;
import org.junit.runners.Suite;

// Runs all unit tests.
@RunWith(Suite.class)
@Suite.SuiteClasses({CalculatorInstrumentationTest.class,
        CalculatorAddParameterizedTest.class})
public class UnitTestSuite {}
```

## 运行Instrumented单元测试

要运行Instrumented测试,请按照如下步骤操作:

1. 点击工具栏中的**Sync Project** ![img](https://developer.android.com/images/tools/sync-project.png)按钮来进行项目同步,保证您的项目与Gradle同步;
2. 使用以下方法中的一种运行测试:
   * 要运行单个测试,请打开**Project**窗口,然后右键单击测试并且点击 **Run** ![img](https://developer.android.com/images/tools/as-run.png);
   * 要测试类中的所有方法,右键单击测试类或者测试文件并且点击**Run** ![img](https://developer.android.com/images/tools/as-run.png);
   * 要运行目录中的所有测试,请右键单击目录并选择**Run tests** ![img](https://developer.android.com/images/tools/as-run.png).

[Android Plugin for Gradle](https://developer.android.com/tools/building/plugin-for-gradle.html)会便于位于默认目录(`src/androidTest/java/`)中的测试代码,构建一个测试APK和一个生产APK,在已连接的设备上安装这两个APK,并运行测试.然后,在Android Studio的*Run*窗口中会展示Instrumented测试的执行结果.

>**注意:**在运行或者调试Instrumented测试时,Android Studio不会注入[Instant Run](https://developer.android.com/tools/building/building-studio.html#instant-run)所需的其他方法,并且关闭此功能.

### 使用Firebase Test Lab运行测试

使用[Firebase Test Lab](https://firebase.google.cn/docs/test-lab/),您可以针对主流的不同Android设备和不同的设置(区域设置,方向,屏幕尺寸和平台版本)进行测试.这些测试运行在Google数据中心的物理和虚拟设备上.您可以直接从Android Studio中直接部署应用程序到测试实验室或者通过命令行完成这一点.测试实验室将提供测试日志,该日志包含任何导致测试失败的细节.

要使用Firebase测试实验室,您需要执行如下操作.如果您已经拥有了Google账号和Firebase项目,则请忽略:

1. 如果您还没有Google账号,则请[创建一个Google账号](https://accounts.google.com/);

2. 在[Firebase console](https://console.firebase.google.com/)中,点击**Create New Project**.

   在[Spark Plan的每日免费限额](https://firebase.google.cn/docs/test-lab/overview#quota_for_spark_and_flame_plans)中通过测试实验室测试您的应用程序并不需要任何费用.

#### 配置测试矩阵并执行测试

Android Studio提供了集成工具,允许您将测试程序部署到Firebase测试实验室中.在您通过Blaze计费方式创建了Firebase项目后,您就可以创建测试程序并执行:

1. 在主菜单中点击**Run** > **Edit Configurations**;

2. 点击**Add New Configuration** ![img](https://developer.android.google.cn/studio/images/buttons/ic_plus.png)并且选择**Android Tests**;

3. 在Android Test配置对话框中,可执行如下操作:

   a. 输入或者选择要执行的测试的详细信息;例如测试名称,模块的类型,测试的类型和要测试的类;

   b. 从**Deployment Target Options**的**Target**下拉菜单中,选择**Firebase Test Lab Device Matrix**;

   c. 如果您尚未登录,请点击**Connect to Google Cloud Platform**,并允许Android Studio访问您的账号;

   d. 点击*Cloud Project*旁边的![img](https://developer.android.google.cn/images/tools/as-wrench.png)按钮,并从列表中选择您的Firebase项目.

4. 创建并配置测试矩阵:

   a. 点击*Matrix Configuration*下拉菜单旁边的click **Open Dialog** ![img](https://developer.android.google.cn/images/tools/as-launchavdm.png);

   b. 点击**Add New Configuration (+)**;

   c. 在**Name**输入框中,为您的配置输入一个名称;

   d. 选择您测试应用时需要使用的设备,Android版本,区域设置和屏幕方向.Firebase测试实验室将根据您的选择进行测试并且声称测试报告;

   e. 点击**OK**来保存您的设置.

5. 点击**OK**来退出*Run/Debug*配置对话框;

6. 点击**Run** ![img](https://developer.android.google.cn/images/tools/as-run.png)来运行您的测试.

![img](https://developer.android.google.cn/images/training/ctl-config.png)

**图1**,为Firebase测试实验室添加一个配置.

#### 分析测试结果

当Firebase测试实验室完成测试时,*Run*窗口会打开并显示测试结果,如图2所示.您可能需要点击**Show Passed** ![img](https://developer.android.google.cn/images/tools/as-ok.png)来查看所有已执行的测试.

![图2](https://developer.android.google.cn/images/training/ctl-test-results.png)

**图2**,查看Firebase执行的Instrumented测试的结果.

您也可以通过*Run*窗口中测试结果开头显示的链接来打开Web页面来进行结果的分析.

要了解更多关于Web中查看结果的详细信息,请参阅[Analyze Firebase Test Lab for Android Results](https://firebase.google.cn/docs/test-lab/analyzing-results).