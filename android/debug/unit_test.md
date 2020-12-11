# Robolectric

## 修改仓库地址，解决下载jar包慢问题：

```java
public class AliyunRobolectricRunner extends RobolectricTestRunner {

    static {
        System.setProperty("robolectric.dependency.repo.id", "alimaven");
        System.setProperty("robolectric.dependency.repo.url", "https://maven.aliyun.com/repository/public");
    }

    /**
     * Creates a runner to run {@code testClass}. Looks in your working directory for your AndroidManifest.xml file
     * and res directory by default. Use the {@link org.robolectric.annotation.Config} annotation to configure.
     *
     * @param testClass the test class to be run
     * @throws InitializationError if junit says so
     */
    public AliyunRobolectricRunner(Class<?> testClass) throws InitializationError {
        super(testClass);
    }
}
```

在测试类中使用`@RunWith(AliyunRobolectricRunner.class)`：

```java
@RunWith(AliyunRobolectricRunner.class)
@Config(sdk = Build.VERSION_CODES.N)
public class ExampleUnitTest {
    private static final String TAG = "ExampleUnitTest";

    @Test
    public void addition_isCorrect() {
        Log.d(TAG, "addition_isCorrect: ");
    }
}
```

