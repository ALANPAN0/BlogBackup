---
title: Building Faster in Xcode
date: 2018-06-17 20:09:54
tags : [WWDC]
---

Session 408 
<!--more-->

![Building Faster in Xcode_](/images/Building%20Faster%20in%20Xcode_.png)

### Parallelizing Your Build
###### Xcode’s Targets and Dependencies

* Target specifies a product to build
	* iOS App 
	* Framework
	* Unit Tests
* Target dependency requires another target
	* Explicit via Target Dependencies 
	* Implicit via Link Binary with Libraries

###### Game Dependency Graph
![QQ20180617-164649@2x][image-1]

* List of all targets to build 
* Dependency between targets
* Build order can be derived

![QQ20180617-165008@2x][image-2]

![QQ20180617-165142@2x][image-3]

* Amount of work did not change 
* Time to build decreased 
* Increased hardware utilization

###### How do we get there?
* edit scheme
* build
* build option
	1. Parallelize build
	2. Find implicit Dependencies


###### Parallelized Target Build Process
* Compile sources can start earlier
* Waits only for what it needs 
* Must wait for Run Script phases

### Run Script Phases (Reducing the work on rebuilds)
> Allows you to customize your build process for your exact needs.

![QQ20180617-174124@2x][image-4]

###### File Lists  
![QQ20180617-173229@2x][image-5]

* Newline separated
* Support for build setting variables 
* Cannot be generated during the build

### Measuring Build Time
###### Build With Timing Summary
![][image-6]

![][image-7]


### Source-Level Improvements
* Dealing with complex expressions
* Understanding dependencies in Swift
* Limiting your Objective-C/Swift interface

###### Turn off Whole Module Mode For Debug Builds

 ![][image-8]

###### Use Explicit Types for Complex Properties
![][image-9]

###### Provide Types in Complex Closures

![][image-10]

###### Break Apart Complex Expressions

![][image-11]

###### Use AnyObject Methods and Properties Sparingly

![][image-12]

调用 `AnyObject` 的方法时, 会去所有的文件中去查找该方法, 编译效率低

解决方案:
![][image-13]
通过定义 `protocol`, 然后查找方法就会去此 `protocol` 的声明里面直接去找


### Understanding Dependencies in Swift
###### Incremental Builds Are File-Based
![][image-14]

![][image-15]

增加方法此时会重新编译

![][image-16]

改变方法的 bodies 不会影响到其他的文件

![][image-17]

不关联的改变在方法外部也会导致重新编译

###### Dependencies Within a Target Are Per-File (目标文件的依赖关系)

![][image-18]

###### Cross-Target Dependencies Are Coarse-Grained

![][image-19]


###### Swift Dependency Rules
* Compiler must be conservative
* Changes in function bodies do not affect the file’s interface  
* Dependencies within a module are per-file
* Dependencies across targets are for the whole target

###### Mixed-Source App Targets

![][image-20]

###### Keep Your Generated Header Minimal
* Use private when possible

	```
	`@objc private func keyboardWillShow(_: Notification) { // Important keyboard setup code here.
	}.
	// ...
	NotificationCenter.default.addObserver(self, selector: #selector(keyboardWillShow(_:)), ...)
	```
* Use block-based APIs

	```
	`self.observer = NotificationCenter.default.addObserver( forName: UIKeyboardWillShow, object: nil, queue: nil) {
	// Important keyboard setup code here.
	}.
	```
* Turn off “Swift 3 @objc Inference”
	![][image-21]
* Keep Your Bridging Header Minimal
	* Use categories to break up your interface
---- 
![][image-22]

### More Information
> https://developer.apple.com/wwdc18/408




[image-1]:	/images/QQ20180617-164649@2x.png
[image-2]:	/images/QQ20180617-165008@2x.png
[image-3]:	/images/QQ20180617-165142@2x.png
[image-4]:	/images/QQ20180617-174124@2x.png
[image-5]:	/images/QQ20180617-173229@2x.png
[image-6]:	/images/15292293472562.jpg
[image-7]:	/images/15292295109399.jpg
[image-8]:	/images/15292297716350.jpg
[image-9]:	/images/15292304002286.jpg
[image-10]:	/images/15292304507458.jpg
[image-11]:	/images/15292305166448.jpg
[image-12]:	/images/15292305743843.jpg
[image-13]:	/images/15292308090026.jpg
[image-14]:	/images/15292312274443.jpg
[image-15]:	/images/15292312680309.jpg
[image-16]:	/images/15292313035255.jpg
[image-17]:	/images/15292313838157.jpg
[image-18]:	/images/15292346436023.jpg
[image-19]:	/images/15292346898606.jpg
[image-20]:	/images/15292351447821.jpg
[image-21]:	/images/15292355397528.jpg
[image-22]:	/images/15292356504822.jpg

