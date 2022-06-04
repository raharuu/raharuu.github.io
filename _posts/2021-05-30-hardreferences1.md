---
title: What Are Hard References & Reasons to Avoid Them
description: 'Excessive Hard References grind projects to halt with technical debt, poor load times, and decreased productivity.'
excerpt: A hard reference is created when an asset depends upon another asset. The result is that whenever one asset is loaded...
categories:
  - Unreal
tags:
  - Unreal Engine
  - Best Practices
  - Soft References
  - Hard References
  - Asset Management
slug: hard-references-reasons-avoid
lastmod: '2022-06-02T12:07:31.628Z'
---

A hard reference is created when an asset depends upon another asset. The result is that whenever one asset is loaded, all assets it depends upon are also loaded into memory.

A simple example below demonstrates this using a blueprint that depends upon a static mesh. The hard reference is created because our actor's StaticMeshComponent depends upon the Cube static mesh. To get this reference view for one of your assets, right-click on the asset in the content browser and select **Reference Viewer**.

[![ReferenceViewer]({{ site.url }}{{ site.baseurl }}/assets/images/referenceviewer1.png)]({{ site.url }}{{ site.baseurl }}/assets/images/referenceviewer1.png)

Cases that result in a Hard Reference being created:

- A variable that is of an asset type (*Material, Texture, BP_XYZ, StaticMesh*)
- SpawnActor, via the Class input pin
- Casting (Cast to BP_XYZ)
- GetAllActorOfClass via the Class input pin
- GetAllWidgetOfClass via the Class input pin
- A nested blueprint via a ChildActor component (Child Actor Class)
- Inheritance (BP_Earth inherits from BP_Planet, BP_Planet the child depends upon BP_Earth, creating a hard reference)

[![ReferenceViewer]({{ site.url }}{{ site.baseurl }}/assets/images/hardreferencenodes.png)]({{ site.url }}{{ site.baseurl }}/assets/images/hardreferencenodes.png)

## Why Are Hard References Bad?

Hard references create a dependency where if asset A is loaded, anything it depends on is also loaded, and then assets those assets depend on are also loaded, and again, and again until everything required for the original asset is loaded into memory.

Without keeping references in mind throughout the project life cycle, it will quickly get to a point where most assets have a Reference Viewer that looks something like this, with hundreds or even thousands of asset references for a single blueprint, in the case below a PlayerController.

[![ReferenceViewer]({{ site.url }}{{ site.baseurl }}/assets/images/referenceviewer2.png)]({{ site.url }}{{ site.baseurl }}/assets/images/referenceviewer2.png)

When reference trees start to get this unwieldy, the amount of data required to load a single asset will begin severely impacting editor load times, blueprint compile times and packaged load times. This will come at the cost of reducing productivity for everyone on the team. Further to this, it’s usually a good sign that something is wrong with how a system or systems have been architected or implemented.

All assets have a Size Map view showing all hard references, their summed data footprint on disk, and other useful information. To access this, right-click on any asset and select Size Map.

[![ReferenceViewer]({{ site.url }}{{ site.baseurl }}/assets/images/sizemap1.png)]({{ site.url }}{{ site.baseurl }}/assets/images/sizemap1.png)

Letting hard references fester in a project is likely to cause technical debt and challenges further down the line, as systems will have to be refactored to solve the underlying problems and dependencies between assets.

## Size Map / Reference Viewer

Use the Size Map and Reference Viewer tools to help you identify and debug high dependencies, slow load times, and high memory usage.

[![ReferenceViewer]({{ site.url }}{{ site.baseurl }}/assets/images/referenceviewer3.png)]({{ site.url }}{{ site.baseurl }}/assets/images/referenceviewer3.png)

## Avoiding Hard References

How you approach removing hard references depends heavily upon the context of what the hard reference is being caused by, and why it might be necessary in the first place. Here are a few solutions you can deploy when you are dealing with hard references:

### Casting

Casting is one of the main causes of hard references and tangled dependencies in a project. By casting to uassets such as blueprints, you create hard references to them. Avoid this whenever possible. It is however perfectly safe to cast to native classes such as a regular **APawn**, **AActor**, **APlayerController** **UTexture2D** etc, or one of your own natively defined classes **AMyPlayerController**.

### Native C++ definitions

Casting to a native C++ class does not incur a hard reference and is perfectly safe. Thus, define member variables and functions natively in C++ as opposed to the blueprint layer. This removes any risk of creating hard references through casting as other classes can safely cast to the native class. C++ members can be exposed to the blueprint layer where they are potentially implemented, overriden, modified or accessed. Here's an example use case:

You have a **BP_PlayerController** that you’d like to access from **BP_ControllerBuddy** via a *“Cast to BP_PlayerController”* node with the intent of accessing some data stored on the **BP_PlayerController**. this will create an undesirable **hard reference**. To avoid this, you can create a **AMyPlayerController** native C++ class that defines the required data, then inherit from that native class with your **BP_PlayerController**. **BP_ControllerBuddy** can then access the data via *“Cast to AMyPlayerController”* instead, which is perfectly safe and no hard reference is created. Additionally, **BP_ControllerBuddy** still has full control over the values of that data if exposed to the blueprint layer.

### Parent Classes

If you don’t have access to C++ or do not feel comfortable working with it to implement a native C++ solution, you can instead create a **BP_PlayerController_Base**. Defining the class variables and functions you need to access there instead. Although the parent class is a blueprint, thus casting to it will create a hard reference, the idea is that you will never reference any other assets in this blueprint, keeping it as purely a container for variables and functions. Instead, a child class (e.g. **BP_PlayerController**) is intended to modify and implement the variables and functions. To give an example, we might define a *ConfirmationWidgetClass* as part of our **_Base** class, but only initialise that to the actual confirmation widget within **BP_PlayerController**. Thus, any other class can cast to **_Base** to retrieve the relevant *ConfirmationWidgetClass* and the cast will not result in a hard reference.

In time, programmers can move these classes into C++ if necessary - and without much hassle too, thanks to the abstract nature of the parent classes you created. This is a great way for those working within Blueprint to reduce technical debt and to better maintain project hygiene.

### Soft Object / Class References (Async Loading)

There’s another type of asset reference, a Soft Object/Class reference. Whenever you have a soft reference to an asset, instead of that asset being forcefully loaded into memory, it is left unloaded. This means that we have direct control over when soft referenced assets are loaded - a powerful tool that we can utilise when we know that an asset is not required immediately or at all times.

[![ReferenceViewer]({{ site.url }}{{ site.baseurl }}/assets/images/softref1.png)]({{ site.url }}{{ site.baseurl }}/assets/images/softref1.png) | [![RefernceViewer]({{ site.url }}{{ site.baseurl }}/assets/images/softref2.png)]({{ site.url }}{{ site.baseurl }}/assets/images/softref2.png) |

You can load a soft reference using these nodes: **Async Load Asset** (*For object references, such as a static mesh*) and **Async Load Class Asset** (*For class assets, such as a blueprint*)

A use case example... You have a **BP_Bridge** in your game with a static mesh that has materials and textures. You want to give the player the ability to select between a variety of static meshes and materials to add customisability to the bridge. One option might be to create an array of StaticMesh and Texture2D, creating hard references to any referenced assets. This results in all possible variants being loaded into memory permanently whenever **BP_Bridge** is loaded. Soft References are better suited for this case. It enables the bridge to dynamically load in the desired variants as the player updates their selection as seen below.

[![ReferenceViewer]({{ site.url }}{{ site.baseurl }}/assets/images/softref3.png)]({{ site.url }}{{ site.baseurl }}/assets/images/softref3.png)

### Interfaces

Interfaces enable you to avoid hard references, *as long as the interface itself lacks a hard reference to another uasset type as part of any function parameters or return values*. You can think of interfaces in UE4 as assets themselves with a reference tree and size map. You can make interface calls on an object without needing to know its specific type (*_class*).

Example Context: You have a **BP_PlayerPawn** that can interact with objects. A **BP_Door** which is an example of one such interactable, and a **BPI_InteractInterface** which defines an Interact function.

If we remove the interface from the equation, one way you might tell the player to interact with the door, would be to *“Cast to BP_Door → Interact”*. The two big problems with that are, for one, you’ve created a hard reference, and even more importantly, every time you want to add a new interactable type to your game, you need to cast again, until you cover every possible interactable you have.

This is where interfaces can become quite powerful, the caller of an interface does not need to know what type is on the receiving end of the call, unlike casting. Instead, running through the same example using an interface:

**BP_Door** implements an Interact event from **BPI_InteractInterface**. Whenever the player interacts, instead of casting to specific objects, the player just sends out an **Interact** call to the Object (Note, not a specific type) and either something will happen, in this case, **BP_Door** will run **Interact**. Or nothing will happen. With this, we no longer need to create that Cast chain and have a much more extensible system as a result. We avoid creating any hard references to the interactable types themselves, all thanks to our interface. Nice!

## Commandlets

It is challenging for larger teams to manage blueprints as binary assets. Unlike native code, reviewing blueprint changes is not as simple as previewing the changes using a diff tool from perforce/git changelist. A reviewer looking at a blueprint changelist needs to be in the editor, manually cycling through each blueprint and its many graphs. As a result, reviewing blueprints is more time-consuming, leading to a drop in overall quality and risk of technical debt build-up in projects making heavy use of them.

Commandlets can help you regain that control. With UCommandlet, you can create a commandline tool that audits your blueprints externally from the editor.

One such Commandlet might search for all blueprint assets and create a text-based report that warns when blueprints are casting to other blueprint classes. These commandlets can be integrated with CI and build management pipelines to highlight issues daily. They could also be exposed to developers directly, enabling anyone to check and audit their work from the editor before submitting it to source control.

You can learn more about creating commandlets of your own here [How to Write A Commandlet](https://www.oneoddsock.com/blog/2020/07/08/ue4-how-to-write-a-commandlet).
