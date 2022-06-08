---
title: 'Blueprint Pure Functions: Yes? No? It''s Complicated'
description: 'Blueprint Pure functions are widely misused, causing performance loss, unexpected output, and bugs. Learn how to use blueprint pure efficiently.'
excerpt: 'To avoid misusing Pure functions, you can follow a simple rule: Never connect an...'
categories:
  - Unreal
tags:
  - Unreal Engine
  - Blueprints
slug: blueprint-pure-functions-complicated
---

You can create a pure function in Unreal Engine by ticking the **“Pure”** tick box in the details panel of a blueprint function. Alternatively, in C++ - you can set a function as `const` or `BlueprintPure` (both will give the same Pure style node in blueprint).

    UPROPERTY(BlueprintCallable)
    float GetHealth() const { return Health; }

    UPROPERTY(BlueprintPure)
    float GetMaxHealth() { return Health; }

[![Pure&Impure]({{ site.url }}{{ site.baseurl }}/assets/images/BlueprintPureFunctions/BlueprintPureFunction1.png)]({{ site.url }}{{ site.baseurl }}/assets/images/BlueprintPureFunctions/BlueprintPureFunction1.png)

A pure blueprint node removes the need to explicitly execute a function via the white execution pins on either side of impure nodes. Instead, Pure nodes are executed on demand whenever their output is required by an impure node for it to execute. One of the important differences between pure and impure nodes is that pure nodes ***do not cache their result***, meaning that the function is called repeatedly every time the output is required by an impure node.

## Misuses of Pure Functions

Pure nodes despite being quite convenient most of the time, are easy to misuse because they do not cache their result. To avoid misusing Pure functions, you can follow a simple rule: ***Never connect an expensive pure function to more than 1 impure node.*** In the 2 images below, I have a simple blueprint pure node that prints “Hello” to the screen, because blueprint pure nodes do not cache their output, the ‘Pure’ event below prints “Hello” to the screen 3 times, despite the node seemingly being reused.

[![SimplePureFunction]({{ site.url }}{{ site.baseurl }}/assets/images/BlueprintPureFunctions/BlueprintPureFunction2.png)]({{ site.url }}{{ site.baseurl }}/assets/images/BlueprintPureFunctions/BlueprintPureFunction2.png)

[![ExampleOfMisuse]({{ site.url }}{{ site.baseurl }}/assets/images/BlueprintPureFunctions/BlueprintPureFunction3.png)]({{ site.url }}{{ site.baseurl }}/assets/images/BlueprintPureFunctions/BlueprintPureFunction3.png)

If you have a pure node that is expensive and you need the result to output to various impure nodes, you have 2 options:

1. Turn the pure node into an impure node so that the result is cached (thus PrintHello only prints “Hello” to the screen once in this case)
[![CachedResultsViaImpureNode]({{ site.url }}{{ site.baseurl }}/assets/images/BlueprintPureFunctions/BlueprintPureFunction4.png)]({{ site.url }}{{ site.baseurl }}/assets/images/BlueprintPureFunctions/BlueprintPureFunction4.png)

2. Cache the output of the pure node and use that as the input to the various impure nodes that need it.
[![CachePureNodeOutput]({{ site.url }}{{ site.baseurl }}/assets/images/BlueprintPureFunctions/BlueprintPureFunction5.png)]({{ site.url }}{{ site.baseurl }}/assets/images/BlueprintPureFunctions/BlueprintPureFunction5.png)

The node **GetAllActorsOfClass** is a classic example of Epic Games intentionally choosing to keep a node impure to prevent bad misuse by inexperienced unreal developers.

## Macro Complications

Remember that simple rule from earlier? "***Never connect a pure function to more than 1 impure node.***" There is an exception to this rule... Macros such as this **ForEachLoop** below may look like an innocent impure node. You may expect the pure **GetComponentsByClass** node to be called once whilst the **ForEachLoop** iterates over its result - but that is not the case. **GetComponentsByClass** will be called again **twice** on each iteration of the loop due to the way this macro (and potentially others) are set up.

If you peek inside of the **ForEachLoop** macro, you'll see that 2 other nodes (*Length* and *Get*) are querying it **each** iteration. An extreme example of how this can be problematic not just for performance, but for unexpected output and even crashes... the blueprint below will result in an infinite loop because **GetComponentsByClass** is called on each iteration of the loop, yet is increasing in size in every iteration as well because we are constructing a fresh component on each iteration. In this case, cache the value first as both a safety precaution and because the **GetComponentsByClass** is an expensive operation you shouldn’t be unnecessarily repeating anyway.

[![ForEachGotchas]({{ site.url }}{{ site.baseurl }}/assets/images/BlueprintPureFunctions/BlueprintPureFunction6.png)]({{ site.url }}{{ site.baseurl }}/assets/images/BlueprintPureFunctions/BlueprintPureFunction6.png)

Another common misuse I see with **ForEachLoop** is when someone wants to **DestroyComponent** on every component of an actor. They'll do as above and plug the **GetComponentsByClass** directly into the **Array** input pin, so for every iteration, a component is destroyed and the list of components is checked again. This results in only half of the components being destroyed. This happens because inside of the ForEachLoop, it is keeping track of the index of the current iteration. If you start with 10 components, destroy 1 component each iteration, and increment your index each iteration, you get this:

    1. GetComponentsByClass returns 10 components (twice)
    2. A component is deleted
    3. Index is incremented (0 -> 1)
    4. GetComponentByClass returns 9 components (twice)
    5. A component is deleted
    6. Index is incremented (1 -> 2)
    7. GetComponentByClass returns 8 components (twice)
    8. A component is deleted
    9. Index is incremented (2 -> 3)
    10. GetComponentByClass returns 7 components (twice)
    11. A component is deleted
    12. Index is incremented (3 -> 4)
    13. GetComponentByClass returns 6 components (twice)
    14. A component is deleted
    15. Index is incremented (4 -> 5)
    16. GetComponentByClass returns 5 components (once)
    17. The current index (5) is no longer less than (<) the 5 components returned, so the loop finishes

## Conclusion

To summarise, be cautious of how you connect your pure nodes to impure nodes (especially when a macro is concerned). Remember the simple rule: "***Never connect a pure function to more than 1 impure node.***", and if that rule cannot be applied, either switch to an impure node or cache your pure node output.

Thanks for reading!

## Further Reading

[Unreal Engine Official Function Documentation](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Blueprints/UserGuide/Functions/#purevs.impure)
