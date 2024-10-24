# Java on Graviton

Java is a general-purpose programming language. Compiled Java code can run on all platforms that support Java, without the need for recompilation. Java applications are typically compiled to bytecode that can run on any Java virtual machine (JVM) regardless of the underlying computer architecture. Note: there are multiple programming languages such as Kotlin, Scala, and Groovy that compile to byte code and run on top of the JVM, so those also tend to be highly portable across architectures. _[Wikipedia](https://en.wikipedia.org/wiki/Java_(programming_language))_

Java is well supported and generally performant out-of-the-box on arm64. [Amazon Corretto](https://aws.amazon.com/corretto/), a no-cost, multiplatform, production-ready distribution of the Open Java Development Kit (OpenJDK) supports Graviton-powered instances.
While Java 8 is fully supported on Arm
processors, some customers haven't been able to obtain Graviton's full
performance benefit until they switched to Java 11.

This page includes specific details about building and tuning Java application on Graviton.

### Java Graviton Migration Checklist
This checklist summarizes portions of the Java on Graviton section and can be helpful for getting started.  It is not a comprehensive summary, so we also recommend reading through all the content below.

* Check AMI & Kernel Version support
    * Pre-2020 Linux distributions are unlikely to contain the right optimizations.
    * Amazon Linux:  AL2023 is ideal.  AL2 is fine with a recent kernel (i.e. not 4.14).  AL2 is EOL in June 2025.
    * Ubuntu:  Use at least Ubuntu 20.04.  More recent versions are even better.
    * Red Hat Linux:  RHEL9 is ideal.  Use at least RHEL8.2 (be aware kernel uses unusual 64KB memory pages).
    * For the full list, see [Operating Systems available for Graviton based instances](https://github.com/aws/aws-graviton-getting-started/blob/main/os.md).


* Check JVM Version & Flavor
    * Java:  The more recent the better. AWS recommends at least JDK11, but ideally JDK17 or newer.  JDK8 is the minimum version supporting Arm64 but likely won’t provide the best performance.
    * [Amazon Corretto](https://aws.amazon.com/corretto) (Amazon’s distribution of OpenJDK):  Typically provides the best performance and is recommended for Graviton workloads.  Corretto 11 and above support LSE extensions, a set of atomic memory operations that improve performance for lock-contended workloads and reduce garbage collection time.  Some customers have seen even better performance on Corretto 17 and Corretto 21.
* Check JARs and shared objects for architecture specific code (compiled other than Java byte code)
    * See guidance for [manual scanning process](https://github.com/aws/aws-graviton-getting-started/blob/main/java.md#looking-for-x86-shared-objects-in-jars).
    * [Porting Advisor for Graviton](https://github.com/aws/porting-advisor-for-graviton) can scan/flag them and is useful for [Maven](https://maven.apache.org/index.html)-based projects.
    * JNI extensions usually exist to implement performance critical functions in a language other than Java.  Without an Arm64 version the code may not work or can fall back on a slower pure Java implementation.  Check for Arm64 versions or later versions of the package to see if the JNI has been superseded by more performant native Java implementations.
    * Follow [these instructions](https://github.com/aws/aws-graviton-getting-started/blob/main/java.md#building-multi-arch-jars) for building multi-arch JAR’s that support both x86 and Arm64/Graviton.
* Java Crypto operations:
    * Review [Java JVM options](https://github.com/aws/aws-graviton-getting-started/blob/main/java.md#java-jvm-options) for Crypto optimizations and recommendations.
    * AES/GCM benefits when using AES hardware instructions, which can improve performance by up to 5x for this algorithm.  Corretto & OpenJDK 18 support this by default and have been back-ported to Corretto & OpenJDK 11 and 17 which can be enabled using `XX:+UnlockDiagnosticVMOptions -XX:+UseAESCTRIntrinsics`.
    * [Amazon-corretto-crypto-provider](https://github.com/corretto/amazon-corretto-crypto-provider) is another option that offers optimizations for a large number of cryptographic operations.
* Application Testing & Performance Evaluation
    * Be sure to run Graviton instances “hotter”: vCPUs are mapped to physical cores instead of Hyperthreads and performance often flatlines at a much higher CPU utilization than with x86 based instances.  Testing at low levels of load can lead to misleading results.  The most realistic test results are usually achieved when testing close to breaking latency.
    * See the [Graviton Performance Runbook](https://github.com/aws/aws-graviton-getting-started/blob/main/perfrunbook/README.md) for more info.
    * [Aperf](https://github.com/aws/aperf) is a CLI tool for gathering & visualizing performance data that can be helpful.

### Java versions
JDK binaries for arm64 are available from a number of
different sources.  [Amazon Corretto](https://aws.amazon.com/corretto/) is
continuing to improve performance of Java workloads running on Graviton processors and
if you have the choice of a JDK to use we recommend using Corretto as it
provides the fastest way to get access to the performance improvements AWS is making.

Versions of Corretto released since October 2020 are built to use the
most optimal atomic operations within the JVM: Corretto11 (all
variants); Correto8 (on Amazon Linux 2 only). This has shown to reduce
GC time in some workloads, and avoids contention in net-intensive workloads like Apache Kafka.

Versions of Corretto11 (>=11.0.12) come with additional enhancements to improve
performance on workloads with light to moderate lock-contention: improved spin-lock behavior inside the JVM,
enhanced implementation of `Thread.onSpinWait()` on Graviton.

### Java JVM Options
There are numerous options that control the JVM and may lead to better performance.

- Flags `-XX:-TieredCompilation -XX:ReservedCodeCacheSize=64M -XX:InitialCodeCacheSize=64M`
have shown large (1.5x) improvements in some Java workloads. Corretto 17 needs two additional flags:
`-XX:CICompilerCount=2 -XX:CompilationMode=high-only`. `ReservedCodeCacheSize`/`InitialCodeCacheSize`
should be equal and can be in range: 64M...127M.
The JIT compiler stores generated code in the code cache. The flags change the size of the code cache
from the default 240M to the smaller one. The smaller code cache may help CPU
to improve the caching and prediction of jitted code. The flags disable the tiered compilation
to make the JIT compiler able to use the smaller code cache.
These are helpful on some workloads but can hurt on others so testing with and without
them is essential.

- Crypto algorithm AES/GCM used by TLS has been optimized for Graviton.
On Graviton2 GCM encrypt/decrypt
[performance improves by 3.5x to 5x](https://github.com/aws/aws-graviton-getting-started/issues/110#issuecomment-989442948).
The optimization is enabled by default in Corretto and OpenJDK 18 and later.
The optimization has been backported to Corretto and OpenJDK 11 and 17
and can be enabled with the flags `-XX:+UnlockDiagnosticVMOptions -XX:+UseAESCTRIntrinsics`.
As an alternative, you can use [Amazon Corretto Crypto Provider](https://github.com/corretto/amazon-corretto-crypto-provider) JNI libraries.

### Java Stack Size
The default stack size for Java threads (i.e. `ThreadStackSize`) is 2mb on aarch64 (compared to 1mb on x86_64). You can check the default with:
```
$ java -XX:+PrintFlagsFinal -version | grep ThreadStackSize
     intx CompilerThreadStackSize = 2048  {pd product} {default}
     intx ThreadStackSize         = 2048  {pd product} {default}
     intx VMThreadStackSize       = 2048  {pd product} {default}
```
The default can be easily changed on the command line with either `-XX:ThreadStackSize=<kbytes>` or `-Xss<bytes>`. Notice that `-XX:ThreadStackSize` interprets its argument as kilobytes whereas `-Xss` interprets it as bytes. So `-XX:ThreadStackSize=1024` and `-Xss1m` will both set the stack size for Java threads to 1 megabyte:
```
$ java -Xss1m -XX:+PrintFlagsFinal -version | grep ThreadStackSize
     intx CompilerThreadStackSize                  = 2048                                   {pd product} {default}
     intx ThreadStackSize                          = 1024                                   {pd product} {command line}
     intx VMThreadStackSize                        = 2048                                   {pd product} {default}
```

Usually, there's no need to change the default, because the thread stack will be committed lazily as it grows. So no matter what's the default, the thread will always only commit as much stack as it really uses (at page size granularity). However there's one exception to this rule if [Transparent Huge Pages](https://www.kernel.org/doc/html/latest/admin-guide/mm/transhuge.html) (THP) are turned on by default on a system. In such a case the THP page size of 2mb matches exactly with the 2mb default stack size on aarch64 and most stacks will be backed up by a single huge page of 2mb. This means that the stack will be completely committed to memory right from the start. If you're using hundreds or even thousands of threads, this memory overhead can be considerable.

To mitigate this issue, you can either manually change the stack size on the command line (as described above) or you can change the default for THP from `always` to `madvise` on Linux distributions like AL2 (with Linux kernel 5 and higher) on which the setting defaults to `always`:
```
# cat /sys/kernel/mm/transparent_hugepage/enabled
[always] madvise never
# echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
# cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never
```

Notice that even if the the default is changed from `always` to `madvise`, the JVM can still use THP for the Java heap and code cache if you specify `-XX:+UseTransparentHugePages` on the command line.

### Looking for x86 shared-objects in JARs
Java JARs can include shared-objects that are architecture specific. Some Java libraries check
if these shared objects are found and if they are they use a JNI to call to the native library
instead of relying on a generic Java implementation of the function. While the code might work,
without the JNI the performance can suffer.

A quick way to check if a JAR contains such shared objects is to simply unzip it and check if
any of the resulting files are shared-objects and if an aarch64 (arm64) shared-object is missing:
```
$ unzip foo.jar
$ find . -name "*.so" -exec file {} \;
```
For each x86-64 ELF file, check there is a corresponding aarch64 ELF file
in the binaries. With some common packages (e.g. commons-crypto) we've seen that
even though a JAR can be built supporting Arm manually, artifact repositories such as
Maven don't have updated versions. To see if a certain artifact version may have Arm support,
consult our [Common JARs with native code Table](CommonNativeJarsTable.md).
Feel free to open an issue in this GitHub repo or contact us at ec2-arm-dev-feedback@amazon.com
for advice on getting Arm support for a required Jar. 

### Building multi-arch JARs
Java is meant to be a write once, and run anywhere language.  When building Java artifacts that
contain native code, it is important to build those libraries for each major architecture to provide
a seamless and optimally performing experience for all consumers.  Code that runs well on both Graviton and x86
based instances increases the package's utility.

There is nominally a multi-step process to build the native shared objects for each supported
architecture before doing the final packaging with Maven, SBT, Gradle etc. Below is an example
of how to create your JAR using Maven that contains shared libraries for multiple distributions
and architectures for running your Java application interchangeably on AWS EC2 instances
based on x86 and Graviton processors:

```
# Create two build instances, one x86 and one Graviton instance.
# Pick one instance to be the primary instance.
# Log into the secondary instance
$ cd java-lib
$ mvn package
$ find target/ -name "*.so" -type f -print

# Note the directory this so file is in, it will be in a directory
# such as: target/classes/org/your/class/hierarchy/native/OS/ARCH/lib.so

# Log into the primary build instance
$ cd java-lib
$ mvn package

# Repeat the below two steps for each OS and ARCH combination you want to release
$ mkdir target/classes/org/your/class/hierarchy/native/OS/ARCH
$ scp slave:~/your-java-lib/target/classes/org/your/class/hierarchy/native/OS/ARCH/lib.so target/classes/org/your/class/hierarchy/native/OS/ARCH/

# Create the jar packaging with maven.  It will include the additional
# native libraries even though they were not built directly by this maven process.
$ mvn package

# When creating a single Jar for all platform native libraries, 
# the release plugin's configuration must be modified to specify 
# the plugin's `preparationGoals` to not include the clean goal.
# See http://maven.apache.org/maven-release/maven-release-plugin/prepare-mojo.html#preparationGoals
# For more details.

# To do a release to Maven Central and/or Sonatype Nexus:
$ mvn release:prepare
$ mvn release:perform
```

This is one way to do the JAR packaging with all the libraries in a single JAR.  To build all the JARs, we recommend to build on native
machines, but it can also be done via Docker using the buildx plug-in, or by cross-compiling inside your build-environment.

Additional options for releasing jars with native code is to: use a manager plugin such as the [nar maven plugin](https://maven-nar.github.io/)
to manage each platform specific Jar.  Release individual architecture specific jars, and then use the primary
instance to download these released jars and package them into a combined Jar with a final `mvn release:perform`.
An example of this methd can be found in the [Leveldbjni-native](https://github.com/fusesource/leveldbjni) `pom.xml` files. 

### Remove Anti-patterns

Anti-patterns can impact the performance on any instance family, but the level of impact can be different between instance families.  Below is a list of anti-patterns
we have observed to impact Graviton positively when removed as much as possible:

1.  **Excessive exceptions**: Throwing exceptions and generating stack-traces has been observed to cost up to 2x more on Graviton platforms compared to x86.  We recommend not to use Java exceptions as control flow, and to remove exceptions when they appear in the hot-code path. Identifying hot exceptions can be done using function profilers like Aperf, Async-profiler, or Linux `perf`. Overhead can be mitigated some by using the `-XX:+OmitStackTraceInFastThrow` JVM flag to allow the Java runtime to optimize the exception flow for some hot paths. The best solution is to avoid the exceptions as much as possible.

### Profiling Java applications

### APerf + Async-profiler

To profile Java we recommend using [Aperf](https://github.com/aws/aperf) to gather profiles and view them
via a static webpage.  On your test system, do the following to install APerf and Async-profiler to profile your
Java code:

```bash
# Get latest APerf release onto the machine you are profiling
wget https://github.com/aws/aperf/releases/download/v0.1.13-alpha/aperf-v0.1.13-alpha-aarch64.tar.gz
tar -zxf aperf-v0.1.13-alpha-aarch64.tar.gz

# Get the latest Async profiler
wget https://github.com/async-profiler/async-profiler/releases/download/v3.0/async-profiler-3.0-linux-arm64.tar.gz
tar -zxf async-profiler-3.0-linux-arm64.tar.gz
cd async-profiler-3.0-linux-arm64
sudo mkdir -p /opt/bin
sudo mkdir -p /opt/lib
sudo cp -a bin/* /opt/bin
sudo cp -a lib/* /opt/lib

export PATH=/opt/bin:$PATH
export LD_LIBRARY_PATH=/opt/lib:$LD_LIBRARY_PATH
sudo sysctl -w kernel.kptr_restrict=0
sudo sysctl -w kernel.perf_event_paranoid=-1
cd aperf-v0.1.13-alpha-aarch64

# While the application is running
./aperf record --profile --profile-java --period 300 -r java_record
./aperf report -r java_record -n java_report

# In java_report folder, open index.html in a browser to view report
```

### Linux `perf` and `libperf-jvmti.so`
If prefering to use the standard Linux `perf` tool, we can capture information about symbols
on Java JIT'ed code leveraging the `libperf-jvmti.so` agent. 
Follow the below steps to use the `libperf-jvmti.so` to dump symbols for
JITed code as the JVM runs.

```bash
# Compile your Java application with -g

# find where libperf-jvmti.so is on your distribution

# Run your java app with -agentpath:/path/to/libperf-jvmti.so added to the command line
# Launch perf record on the system
$ perf record -g -k 1 -a -o perf.data sleep 5

# Inject the generated methods information into the perf.data file
$ perf inject -j -i perf.data -o perf.data.jit

# View the perf report with symbol info
$ perf report -i perf.data.jit

# Process the new file, for instance via Brendan Gregg's Flamegraph tools
$ perf script -i perf.data.jit | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > ./flamegraph.svg
```

### Build libperf-jvmti.so on Amazon Linux 2
Amazon Linux 2 does not package `libperf-jvmti.so` by default with the `perf` yum package for kernel versions <5.10.
Build the `libperf-jvmti.so` shared library using the following steps:

```bash
$ sudo amazon-linux-extras enable corretto8
$ sudo yum install -y java-1.8.0-amazon-corretto-devel

$ cd $HOME
$ sudo yumdownloader --source kernel

$ cat > .rpmmacros << __EOF__
%_topdir    %(echo $HOME)/kernel-source
__EOF__

$ rpm -ivh ./kernel-*.amzn2.src.rpm
$ sudo yum-builddep kernel
$ cd kernel-source/SPECS
$ rpmbuild -bp kernel.spec

$ cd ../BUILD
$ cd kernel-*.amzn2
$ cd linux-*.amzn2.aarch64
$ cd tools/perf
$ make
```
