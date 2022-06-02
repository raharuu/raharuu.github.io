---
title: "What Are Hard References & Reasons to Avoid Them"
description: "Excessive Hard References grind projects to halt with technical debt, poor load times, and decreased productivity."
excerpt: "A hard reference is created when an asset is dependent upon another asset. The result is that whenever one asset is loaded..."
categories:
  - Unreal
tags:
  - Unreal Engine
  - Best Practices
  - Soft References
  - Hard References
  - Asset Management
---

A hard reference is created when an asset is dependent upon another asset. The result is that whenever one asset is loaded, all assets it is dependent upon are also loaded into memory.

A simple example below demonstrates this using a blueprint that depends upon a static mesh. The dependency comes from the static mesh being referenced by the StaticMeshComponent in the blueprint’s hierarchy. If you’re interested in looking at this view in one of your projects, you can right-click on any asset in the content browser and hit **Reference Viewer** to view its hard references.

[![ReferenceViewer]({{ site.url }}{{ site.baseurl }}/assets/images/referenceviewer1.png)]({{ site.url }}{{ site.baseurl }}/assets/images/referenceviewer1.png)


Many other cases create a hard reference to another asset:
- A variable that is of an asset type (Material, Texture, BP_XYZ, StaticMesh)
- SpawnActor, via the Class input pin
- Casting (Cast to BP_XYZ)
- GetAllActorOfClass via the Class input pin
- GetAllWidgetOfClass via the Class input pin
- A nested blueprint via a ChildActor component (Child Actor Class)
- Inheritance (BP_Earth inherits from BP_Planet, BP_Planet the child depends upon BP_Earth, creating a hard reference)

[![RefernceViewer]({{ site.url }}{{ site.baseurl }}/assets/images/hardreferencenodes.png)]({{ site.url }}{{ site.baseurl }}/assets/images/hardreferencenodes.png)

# Why Are Hard References Bad?

Hard references create a dependency where if asset A is loaded, anything it depends on is also loaded, and then assets those assets depend on are also loaded, and again, and again until everything required for the original asset is loaded into memory. 

Without keeping references in mind throughout the project life cycle, it will quickly get to a point where the majority of your assets have a Reference Viewer that looks something like this, with hundreds or even thousands of asset references for a single blueprint, in the case below a PlayerController.

[![RefernceViewer]({{ site.url }}{{ site.baseurl }}/assets/images/referenceviewer2.png)]({{ site.url }}{{ site.baseurl }}/assets/images/referenceviewer2.png)

When reference trees start to get this unwieldy, the amount of data that’s required to load a single asset will begin to severely impact editor load times, blueprint compile times and packaged load times. This will come at the cost of reducing productivity for everyone on the team. Further to this, it’s usually a good sign that something is wrong with how a system or systems have been architecture or implemented.

The object's Size Map is accessed by right-clicking on an asset and hitting Size Map. This shows all of the object's Hard References, their summed data size footprint and other useful information.

[![RefernceViewer]({{ site.url }}{{ site.baseurl }}/assets/images/sizemap1.png)]({{ site.url }}{{ site.baseurl }}/assets/images/sizemap1.png)

Letting hard references fester in a project is likely to cause technical debt and challenges further down the line, as systems will have to be refactored to solve the underlying problems and dependencies between assets.

## Size Map / Reference Viewer

Use the Size Map and Reference Viewer tools to help you identify and debug high dependency counts

[![RefernceViewer]({{ site.url }}{{ site.baseurl }}/assets/images/referenceviewer3.png)]({{ site.url }}{{ site.baseurl }}/assets/images/referenceviewer3.png)

# Avoiding Hard References

How you approach removing hard references depends heavily upon the context of what the hard reference is being caused by, and why it might be necessary in the first place. Here are a few solutions you can deploy when you are dealing with hard references:

## Casting

Casting is one of the main causes of hard references and tangled dependencies in a project. By casting to uassets such as blueprints, you create hard references to them. Avoid this whenever possible. It is however perfectly safe to cast to native classes such as a regular **APawn**, **AActor**, **APlayerController** **UTexture2D** etc, or one of your own natively defined classes **AMyPlayerController**.

## Native C++ definitions

Casting to a native C++ type does not incur a hard hard reference, so it is possible to safely cast to C++ defined classes. You can create a Native C++ class to define the data and functions for your class, and then expose them to the blueprint layer to be accessed or implemented. Here's an example use case: 

You have a **BP_PlayerController** that you’d like to access from **BP_ControllerBuddy** via a *“Cast to BP_PlayerController”* node, with the intent of accessing some data stored on the **BP_PlayerController**. this will create a **hard reference** which we don’t want. A way to avoid this is to create an **AMyPlayerController** C++ class that defines the data you need, then inherit from that native class with your **BP_PlayerController**. **BP_ControllerBuddy** can then access the data via *“Cast to AMyPlayerController”* instead, which is perfectly safe and no hard reference is created.

## Parent Classes

If you don’t have access to C++ or don’t feel comfortable working with it to implement a native C++ solution, you can instead create a **BP_PlayerController_Base**, defining the data you need to access here instead. And although this is a blueprint, and so casting to it will create a hard reference, the idea is that you will never reference any other assets in this blueprint, keeping it as purely a container for data and functions which are intended to be modified and/or implemented by a child class such as **BP_PlayerController** in this case.

Eventually, these classes can be moved into C++ by programmers if deemed necessary, and without much hassle thanks to the abstract nature of these parent classes. This is a great way for those working within Blueprint to reduce technical debt and improve overall project health.

## Soft Object/ Class References (Async Loading)

There’s another type of asset reference, a Soft Object/Class reference. Whenever you have a soft reference to an asset, instead of that asset being forcefully loaded into memory, it is left unloaded. This means that we have direct control over when soft referenced assets are loaded - a powerful tool that we can utilise when we know that an asset is not required immediately or at all times.

[![RefernceViewer]({{ site.url }}{{ site.baseurl }}/assets/images/softref1.png)]({{ site.url }}{{ site.baseurl }}/assets/images/softref1.png) | [![RefernceViewer]({{ site.url }}{{ site.baseurl }}/assets/images/softref2.png)]({{ site.url }}{{ site.baseurl }}/assets/images/softref2.png) |

You can load a soft reference using these nodes: **Async Load Asset** (For object references, such as a static mesh) and **Async Load Class Asset** (For class assets, such as a blueprint)

A use case example... You have a **BP_Bridge** in your game with a static mesh that has materials and textures. You want to give the player the ability to select between numerous static meshes and materials to add customisability to the object. If you decided to use Hard References to achieve this feature by creating an array of StaticMesh and Texture2D, it would result in all possible variants being loaded into memory permanently whenever **BP_Bridge** is loaded, and with a not so happy reference viewer. A better approach is to use soft references instead, to load in the desired variants/assets as the player updates their selection as seen below.

[![RefernceViewer]({{ site.url }}{{ site.baseurl }}/assets/images/softref3.png)]({{ site.url }}{{ site.baseurl }}/assets/images/softref3.png)

## Interfaces

Interfaces allow you to avoid hard references, *as long as the interface itself does not have a hard reference to another uasset type as part of any function parameters or return values*. You can think of interfaces in UE4 as assets themselves with a reference tree and size map. You can make interface calls on an object without needing to know its specific type (*_class*). 

Example Context: You have a **BP_PlayerPawn** that can interact with objects. A **BP_Door** which is an example of one such interactable, and a **BPI_InteractInterface** which defines an Interact function.

If we remove the interface from the equation, one way you might tell the player to interact with the door, would be to *“Cast to BP_Door → Interact”*. The two big problems with that are, for one, you’ve created a hard reference, and even more importantly, every time you want to add a new interactable type to your game, you need to cast again, until you cover every possible interactable you have.

This is where interfaces can become quite powerful, the caller of an interface does not need to know what type is on the receiving end of the call, unlike casting. So instead, running through the example using an interface:

**BP_Door** implements an Interact event from **BPI_InteractInterface**. Whenever the player interacts, instead of casting to specific objects, the player just sends out an **Interact** call to the Object (Note, not a specific type) and either something will happen, in this case, **BP_Door** will run **Interact**. Or nothing will happen. With this, we no longer need to create that Cast chain and have a much more extensible system as a result. We avoid creating any hard references to the interactable types themselves, all thanks to our interface. Nice!

## Commandlets 

Blueprints can be difficult for larger teams to manage due to them being binary assets. Unlike native code, reviewing blueprint changes is not as easy as running a diff tool directly from a changelist. To diff blueprints, the reviewer needs to be in the editor itself, manually cycling through each blueprint and its many graphs. For this reason, blueprints within a larger team can quickly become out of control.

Commandlets can help you regain that control. With UCommandlet, you can create a commandline tool that will audit your blueprints externally from the editor.

One such Commandlet might search for all blueprint assets and create a text-based report that warns when blueprints are casting to other blueprint classes. These commandlets can be integrated as part of your CI and build management pipeline to highlight issues daily. They could also be exposed to developers directly, allowing anyone to check and audit their work from the editor before submitting it to source control.

You can learn more about creating commandlets of your own here [How to Write A Commandlet](https://www.oneoddsock.com/blog/2020/07/08/ue4-how-to-write-a-commandlet).