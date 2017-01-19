# Java classes can be changed at runtime
Class redefinition is the act of replacing the bytecode of a class after the class has already been loaded.

A common example is changing code while debugging. The debugger may recompile the class, then have the JVM's debug agent replace the class bytecode while the application is running. This way the programmer can immediately see the effect of the change they made.

This repo is about how to programmatically redefine classes by a fully supported method that isn't well documented online. The rest of this README describes the process and [`RedefineClassAgent.java`](https://github.com/turn/RedefineClassAgent/blob/master/RedefineClassAgent.java) completely implements it.

# Programmatically redefining classes using the debug agent

[`javassist`](http://jboss-javassist.github.io/javassist/) provides a class called [`HotSwapper`](https://jboss-javassist.github.io/javassist/html/javassist/util/HotSwapper.html) that is able to use the debug agent to redefine arbitrary classes. You pass it a class and the class' new bytecode.

The downside of this approach is that a) the debug agent must be running and listening on a port, and b) the debug agent is buggy.

We've noticed that the debug agent can freeze the JVM if the JVM is under heavy load on a large multithreaded system. We've even had to disable the debug agent on one class of machines because the agent would cause a freeze due to a thread safety problem in its class loading hooks. Those machines do a lot of online class loading of generated code.

# Programmatically redefining classes using the Instrumentation class

Java 1.6 introduced the [`Instrumentation`](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/Instrumentation.html) class that has a handy [`redefineClasses`](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/Instrumentation.html#redefineClasses-java.lang.instrument.ClassDefinition...-) method. The trick is how to obtain an instance of `Instrumentation`.

When an agent is loaded by the JVM `agentmain()` is passed an instance of `Instrumentation`. If the agent's `MANIFEST` contains `Can-Redefine-Classes: true` then that instance of `Instrumentation` can be used to redefine classes.

[`RedefineClassAgent`](https://github.com/turn/RedefineClassAgent/blob/master/RedefineClassAgent.java) does everything you need to do:

* Agents are specified as JARs. For simplicity we can programmatically generate such a JAR in a temporary location. We set the appropriate properties.
* Connect to the JVM. One way to do so is by PID. We attempt to parse the PID out of an MXBean. Yes this is platform dependent, but it appears to work well.
* Load the agent into the JVM.
* The agent saves the `Instrumentation` instance into a `static` variable for later use. The agent's job is done; it exists.
* The cached `Instrumentation` instance is used for class redefinition.

`RedefineClassAgent.redefineClasses()` does all that for you.

# Requirements

* `javassist` is a dependency. It's used to dump `RedefineClassAgent`'s bytecode into the temporary JAR. You'll probably use `javassist` to construct your own new bytecode anyway.
* `tools.jar` is a dependency. It's bundled with the JDK. Add it to your classpath if it's not there already.
* PID detection may break on your platform. It works on HotSpot on Linux.
* Temporary directory must be writeable.

# Usage

    Class clazz; // class to redefine
    byte[] bytecode; // new class bytecode

    ClassDefinition definition = new ClassDefinition(clazz, bytecode));
    RedefineClassAgent.redefineClasses(definition);

Note that you can't change the schema of the class. You can change method bodies, but you can't add or remove methods.

# Examples

## Reload from .class file
Handy if you're testing a change by a hot patch and you'd like to apply it live without having to restart the JVM. Useful if, for example, your service must load a lot of state before it can start serving requests (a restart is expensive) or if you wish to not break existing network connections or internal state.

    Class clazz = Class.forName(className); // the class to reload

    // load the bytecode from the .class file
    URL url = new URL(clazz.getProtectionDomain().getCodeSource().getLocation(),
            className.replace(".", "/") + ".class");
    InputStream classStream = url.openStream();
    byte[] bytecode = IOUtils.toByteArray(classStream);

    ClassDefinition definition = new ClassDefinition(clazz, bytecode));
    RedefineClassAgent.redefineClasses(definition);

## Inject a bit of code into a method
Use `javassist` to find the method, compile and inject new code, then dump new bytecode of the entire class.

        // find a reference to the class and method you wish to inject
        ClassPool classPool = ClassPool.getDefault();
        CtClass ctClass = classPool.get(className);
        ctClass.stopPruning(true);

        // javaassist freezes methods if their bytecode is saved
        // defrost so we can still make changes.
        if (ctClass.isFrozen()) {
            ctClass.defrost();
        }

        CtMethod method; // populate this from ctClass however you wish

        method.insertBefore("{ System.out.println(\"Wheeeeee!\"); }");
        byte[] bytecode = ctClass.toBytecode();

        ClassDefinition definition = new ClassDefinition(Class.forName(className), bytecode);
        RedefineClassAgent.redefineClasses(definition);





