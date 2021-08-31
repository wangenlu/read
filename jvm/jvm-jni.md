# jni

The following code example illustrates how to use functions in the Invocation API. 

In this example, the C++ code creates a Java VM and invokes a static method, called Main.test. For clarity, we omit error checking.

```c
#include <jni.h>       /* where everything is defined */
...
JavaVM *jvm;       /* denotes a Java VM */
JNIEnv *env;       /* pointer to native method interface */
JavaVMInitArgs vm_args; /* JDK/JRE 6 VM initialization arguments */
JavaVMOption* options = new JavaVMOption[1];
options[0].optionString = "-Djava.class.path=/usr/lib/java";
vm_args.version = JNI_VERSION_1_6;
vm_args.nOptions = 1;
vm_args.options = options;
vm_args.ignoreUnrecognized = false;
/* load and initialize a Java VM, return a JNI interface
 * pointer in env */
JNI_CreateJavaVM(&jvm, &env, &vm_args);
delete options;
/* invoke the Main.test method using the JNI */
jclass cls = env->FindClass("Main");
jmethodID mid = env->GetStaticMethodID(cls, "test", "(I)V");
env->CallStaticVoidMethod(cls, mid, 100);
/* We are done. */
jvm->DestroyJavaVM();
```

## Links

- [https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/jniTOC.html](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/jniTOC.html)
- [https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/invocation.html#wp15899](https://docs.oracle.com/javase/6/docs/technotes/guides/jni/spec/invocation.html#wp15899)
