---
title: Fix LLDB
description: 2022-11-08T14:27:50-06:00
date: 2022-11-08T14:27:50-06:00
draft: false
categories:
  - tech
  - iOS
---
# Fixing “virtual filesystem overlay file all-product-headers.yaml not found” error


In the last few days, I saw an increasing number of developers facing a common problem trying to use lldb during a debugging session.

```Bash
error: virtual filesystem overlay file '/Users/t2ac32/Library/Caches/
org.carthage.CarthageKit/DerivedData/13.4.1_13F100/RxSwift/6.5.0/Build/
Intermediates.noindex/ArchiveIntermediates/RxSwift/IntermediateBuildFilesPath/
Rx.build/Release-iphoneos/RxSwift.build/all-product-headers.yaml' not found
```

This error could be really annoying, because it blocks the developer to use the lldb resources, such as po to verify values or conditions during runtime.

## The root cause

There is a known problem for Swift community, about how the compiler generates some artifacts, used by LLDB to process the symbols during the debug session.

The generated DWARFs contains stored data in absolute paths; that means those data are bounded to fixed paths at the machine whose generated it.

Once a developer tries to import those DWARFs, the absolute path, that usually comes from the $HOME folder of the generator user, doesn’t exist in his machine, and LLDB emits an error saying that it doesn’t found the specified path in DWARF.

As this is a compilation problem, there is no easy way to fix that.

And the situation gets worse, because the symbolication tool searches everywhere in your machine for those DSYMs, using SpotLight, an Apple tool that index all files, to provide a faster and assertive search.

The only solution here is to remove the troublemakers from your machine: the DSYMs.

## How to fix it

First, we need to identify which dsym is causing the problem.

For this, let’s get the images UUIDs that has been loaded in app execution.

Run the app, then place a breakpoint anywhere in your code. When the execution stops, run the following command in lldb:

```bash 
image list
```

The LLDB will return a list containing all the loaded images with their UUIDs, as shown in the next image:

![UUID image](UUID.png)

The image shows the image list command after its execution, followed with the exact item of the image the app causing the problem.

Get the UUID of the image that is causing the error. For example, in the bellow error, we can see the framework that is causing the error as RxSwift:

![error image](/posts/swift/images/error.png)

Looking for the framework in the image list, we will find the following UUID:

```Bash 
A970D16E-305F-38A0-8A1D-4D7E2627C6E3
```
Next, let’s open out terminal and run the command that will search for the broken dsym and delete it, running:
```Bash
mdfind -0 "com_apple_xcode_dsym_uuids == UUID-DA-IMAGEM-AQUI" | xargs -0 rm -rf --
```

In our example, we will run the command as:
```Bash
mdfind -0 "com_apple_xcode_dsym_uuids == A970D16E-305F-38A0-8A1D-4D7E2627C6E3" | xargs -0 rm -rf --
```

Next let's erase the DerivedData Folder. It's important to erase all the data contained in that folder, because we may found some left references for the dsym from previous compilations.

if your Derived Data folder is set as defaul, run the followind command in terminal. Otherwise follow the steps bellow.
```Bash
rm -rf $HOME/Library/Developer/Xcode/DerivedData
```

Steps to remove DereviedData Folder:
1. In xcode pres command + ,
2. In the preferences menu go to Locations tab.
3. You will see a row for DerivedData 
4. Next to the grayed out path press the arrow button to open the folder location.
5. In the opened finder window delete the DerivedData folder.

## Finally
I will be necessary to reset your swift packages if any.

At the end, just build and run your application, and the LLDB’s commands like po will be available and working again.


**✘ Made by Tracer for Pocket Operators  [pocket operators](https://tracer.cc) ✘** 
