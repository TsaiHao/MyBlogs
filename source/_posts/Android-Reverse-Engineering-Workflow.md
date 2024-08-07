---
title: Android Reverse Engineering Workflow
date: 2024-08-07 20:55:15
tags:
- Android
- Reverse Engineering
---
## Introduction
Reverse engineering is the process of analyzing a subject system to identify its components and their interrelationships, and to create representations of the system in another form or at a higher level of abstraction. In the context of Android, reverse engineering is often used to understand the behavior of an application, to identify security vulnerabilities, or to modify the application to add new features or fix bugs.

## Prerequisite

1. [ApkTool](https://apktool.org/): A decompiler tool;
2. [Android Development SDK](https://developer.android.com/studio): Recompile smali code and sign the api
3. [Java Development SDK](https://www.oracle.com/java/technologies/downloads/): Run apktool, create keystore for signing

## General Workflow

### Obtain APK

Download apk file from the Internet or CIC website if have access. 

*To be added: Instructions on how to obtain the APK from a device where the package is installed.*

### Decompile APK to smali code

Using `apktool` to decompile .dex binaries into smali code, which is similar to assembly code used in native libraries.

```bash
apktool d release.apk --no-debug-info --force --no-res
```

where:

- `--no-debug-info`: don't write out debug info, including `.param`, `.line` directives.
- `--force`: overwrite the output directory if it already exists.
- `--no-res`: don't extract resource files. This flag could speed up decompiling and recompiling, and useful if resources couldn't be recompiled. However, this flag prevents making a release application debuggable because it does not extract the manifest file.

### Identify Modification Points

Every java/kotlin class, including nested class, has a corresponding `.smali` file, but find the smali file for a given class isn't straitforward as their names are obfuscated in a released apk. There are still some methods to infer their locations.

#### 1. find string literals

String literals, such as trace names and log messages, remain the same in smali code.. For example, I want to find smali class corresponding to `DefaultAudioSink` of `ExoPlayer`, and I find `DefaultAudioSink.java` contains a log:

```java
      try {
        audioTrack.setPlaybackParams(playbackParams);
      } catch (IllegalArgumentException e) {
        Log.w(TAG, "Failed to set playback params", e);
      }
```

I could find the smali with command:

```bash
$ rg '\"Failed to set playback params\"' -w -g "*.smali" ./reverse/release
./reverse/release/smali/J1/S.smali
1882:    const-string v2, "Failed to set playback params"
```

So, `J1.S` is obfuscated name for `androidx.media3.exoplayer.audio.DefaultAudioSink`.

#### 2. compare input parameters and return values

The most efficient way to identify a smali function is to compare its input parameters and return value with candidate java functions.

For example, if we have a function:

```
.method private x0(Z)Ljava/util/List;
```

we could know the java function must like:

```java
private List<XXX> FunctionName(boolean)
```

After we identify which class does x0 belong to, the function signature could help narrow down the range to search function.

#### 3. find AOSP's interface functions

AOSP's interface functions like `MediaCodec`'s won't be mangled in smali code. Therefore, I can directly search for a AOSP function like:

```bash
$ rg 'Landroid/media/MediaCodec;->dequeueOutputBuffer' -w -g "*.smali" ./reverse/release
./reverse/release/smali/O1/K.smali
268:    invoke-virtual {v0, p1, v1, v2}, Landroid/media/MediaCodec;->dequeueOutputBuffer(Landroid/media/MediaCodec$BufferInfo;J)I
```

By cross validation with ExoPlayer's code, it could be known that `O1.K` is mangled `androidx.media3.exoplayer.mediacodec.SynchronousMediaCodecAdapter`

#### 4. translate smali code to Java

Reading complex smali functions could be very hard. Converting them back to Java code could help this.

We need first install [jadx](https://github.com/skylot/jadx), then unzip the apk and decompile a target .dex file.

```bash
unzip release.apk -d apk
cd apk
jadx classes.dex -d class0 --no-imports 
```

Now, we can check decompiled source java code in the class0 directory. `jadx` also supports to decompile a whole .apk file, but it's unnecessary for most use cases. 

### Modify code

Once target code is located, we can modify it to:

1. change behavior;
2. add log;
3. add trace events;
4. print backtrace;
5. etc.

### Repack the apk

1. **Recompile the code and resources**

   ```bash
   apktool b release -o recompile.apk
   ```

   This command simply recompile the smali code and pack them into a new apk. Add other flags as you need: You may use `apktool -advance` to check out what flags are supported.

2. **Create a keystore if not have one already**

   Make sure that JAVA environment has been set up properly, Run command:

   ```bash
   jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore resign.keystore recompile.apk recompile
   ```

   Enter fields interactively when the command asks. Name and organazation don't have to be real. Please remember the password you set to it.

3. **Sign the apk**

   Make sure path of the Android SDK has been exported to $PATH. For example:

   ```bash
   export PATH=/Users/zaijun/Library/Android/sdk/build-tools/30.0.3:$PATH
   ```

   Then, sign the apk with:

   ```bash
   apksigner sign -verbose --v4-signing-enabled=false -ks ./resign.keystore --out signed.apk ./recompile.apk
   ```

   This command will ask for the password you set in the step 2. Note that `--v4-signing-enabled=false` flag is highly recommended, as v4 schema will cause FOS8 system crashing.

4. **Install the application**

   The signed apk can now be installed on a device. It's fine to use overwrite flag "-r":

   ```bash
   adb install -r signed.apk
   ```

   

## Practical Smali Code Snippets

### Debug Log

```
# print out an Uri object
.method private printUri(Landroid/net/Uri;)V
    .locals 2

    const-string v0, "mydebug"
    # tostring
    invoke-virtual {p1}, Ljava/lang/Object;->toString()Ljava/lang/String;
    move-result-object v1

    invoke-static {v0, v1}, Landroid/util/Log;->e(Ljava/lang/String;Ljava/lang/String;)I
.end method
```

Usage example

```
# p1 contains an Uri object
invoke-direct {p0, p1}, Lcom/dss/sdk/media/adapters/exoplayer/AdSourceEventListener;->printUri(Landroid/net/Uri;)V
```

### Add Traces

Code

```
const-string v0, "MyTraceName"
invoke-static {v0}, Landroid/os/Trace;->beginSection(Ljava/lang/String;)V
# wrapped code
invoke-static {v0}, Landroid/os/Trace;->endSection()V
```

### Print Backtrace

Code

```
.method private printBacktrace()V
    .locals 3
    new-instance v0, Ljava/lang/Exception;
    invoke-direct {v0}, Ljava/lang/Exception;-><init>()V
    
    const-string v1, "mydebug"
    const-string v2, "printBacktrace"
    invoke-static {v1, v2, v0}, Landroid/util/Log;->e(Ljava/lang/String;Ljava/lang/String;Ljava/lang/Throwable;)I

    return-void
.end method
```

Usage example

```
invoke-direct {p0}, Lcom/dss/sdk/media/adapters/exoplayer/AdSourceEventListener;->printBacktrace()V
```

### Constructor Log

Code

```
const-string v0, "mydebug-AdPlaybackEndEvent"
invoke-virtual {p0}, Ljava/lang/Object;->toString()Ljava/lang/String;
move-result-object v1
invoke-static {v0, v1}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)I
```

