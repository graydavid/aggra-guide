# FAQ

This wiki answers what I expect might be some frequently-asked questions about Aggra and more subtle points about Aggra's design.

## What are the Best Practices?

Here are some of the best practices for using Aggra:
* Use it only if the features justify the cost. As mention in [the main Aggra wiki](https://graydavid.github.io/aggra-guide), Aggra has a lot of features, but it's also a new, non-trivial programming paradigm to learn. Figure out whether the trade-off is worth it for your particular usecase.
* Minimize the number of nodes. In line with the previous point, if you do decide to use Aggra, it would be a good idea to minimize the number of nodes you need for your usecase. Each node's behavior represents a typical, more-familiar program. E.g. in the extreme case, there would be one node written almost completely traditionally. You would break that out into other nodes as optimality required it.
* Follow the advice in [runnable-decorating-executor](https://github.com/graydavid/runnable-decorating-executor) about how/when to wait on futures, create executors that decorate their runnables, and set default Thread handlers. All of these steps are important to avoid and debug programs that stall.
* Follow the advice in [the advanced wiki](https://graydavid.github.io/aggra-guide/advanced/advanced.html) for how to protect against rogue nodes and wait for GraphCalls to finish.
* Always check unhandledExceptions from a GraphCall's State object when you're done with it. These are exceptions you may not have examined before and might be interested in.
* Prefer predefined [common nodes](https://graydavid.github.io/aggra-guide/common/common.html) before defining your own type.
* Don't save memory instances you create: give ownership over to Aggra immediately and forget about them.
* Make all dependency calls from explicitly-modeled dependencies rather than a new GraphCall. This makes sure the graph structure models how the program works and helps avoid cycles (if you didn't follow the previous advice).
* Don't call dependencies completely on a new Thread (e.g. running DependencyCallingDevice#call on a new thread). If the call blocks, Aggra won't know about the existence of and so will be unable to track the dependency call. Instead, use TimeLimitNodes or the executor-accepting variant of DependencyCallingDevice#call (which only run part of the node call logic on another thread but specifically not the part that produces the Reply, making sure that Aggra knows about the Reply's existence before DependencyCallingDevice#call finishes).
* Avoid using the GRAPH DependencyLifetime. As explained in [the advanced wiki](https://graydavid.github.io/aggra-guide/advanced/advanced.html)'s node phases section, there are tracking costs incurred for node's and consumers. 
* Prefer simpler cancellation over more advanced variants. As mentioned in [the cancellation wiki](https://graydavid.github.io/aggra-guide/cancellation/cancellation.html), the more hooks that a node supports, the more costly it is for Aggra to implement them both in the node itself and consumers.
* If you do end up supporting custom cancel action hooks in a node, make sure to follow all the advice in [the cancellation wiki](https://graydavid.github.io/aggra-guide/cancellation/cancellation.html) to avoid stalling.

## Why are Graphs Static?

As mentioned in [the basics wiki](https://graydavid.github.io/aggra-guide/basics/basics.html), graphs are static: nodes should exist for the lifetime of an application, beyond the scope of a single request. The alternative to static graphs would be dynamic graphs: nodes that are created freshly for every request to a graph. So, why are graphs static and not dynamic?

* Efficient creation of objects -- if we knew before calling a graph that all nodes in the graph would be invoked, we could always create all nodes per request and let the graph run. However, we allow the conditional calling of nodes, meaning some nodes aren't going to be called at all. Always creating them beforehand would be wasteful. For small graphs, that's not such a big deal, but with large graphs with large portions of the graph being conditional, there would be a lot of waste. Static graphs avoid the need to create new nodes ever, at all.
* Centralized cooperation for node creation -- building on the last point, it's actually impossible to know beforehand (for anything other than simple graphs) which nodes are needed for a particular call before actually calling the graph. To avoid duplicate calls to the same node, we need some centralized way to manage node creation... which is exactly what static graphs provide.
* Describability -- since static graphs are always available and don't change form between requests, they're more describable and hence understandable. E.g. it would be possible to create a graph visualization tool.
* Ease/cost of referencing results -- one benefit of dynamic graphs is that the results of a node call would be associated with the node instance directly. With static graphs, nodes have to share and coordinate access to a centralized memory. This coordination is more costly... but probably unaviodable given the previous point.

Considering the tradeoffs, Aggra prioritizes static graphs. You technically *can* create dynamic graphs for every request, but Aggra isn't optimized and won't optimize for this usecase.

## Why can Nodes Call Dependencies in Ancestor Memories?

[The basics wiki](https://graydavid.github.io/aggra-guide/basics/basics.html) justifies why the creation of new memories and calls to dependency nodes there is allowed: to support iteration and reuse. It also provides one justification for a node needing to call a dependency in an ancestor memory: sharing results with other parts of the graph. There is, however, at least one other way to provide that ancestor-accessing feature: force users to pre-call and pass the results of these dependency nodes to the new memory in the form of input. Why doesn't Aggra implement this model, instead?

* Memory hierarchies -- not having to call dependencies in ancestor memories could simplify the Memory implementation, First, there would be no need for memory hierarchies: each node in a memory would already have access to every reply it needs. Second, there would be one less reason to force users to create their own Memory subclasses: i.e. there could be a single Memory class parameterized by a composite input type. For the latter point, there would still be one reason to force users to create Memory subclasses: when parameterizing Nodes by Memory type custom Memory subclasses would remove ambiguity (e.g. Memory<String> would be replaceable with any Memory<String> even though they're perhaps intended for different usecases). So, there's definitely a potential for simplification here.
* Modularity -- nodes would no longer be able to call dependencies in their ancestors conditionally. Either those dependencies would have to be pre-called all the time, or the conditions for making those case would have to be moved (or duplicated) into the ancestor itself. The result would be a loss in modularity: the logic that should be isolated to a single node would be spread out or duplicated across several.
* Latency -- by forcing users to pre-call dependencies and pass them as input, that could negatively affect latency. The user would have to wait until the dependencies were finished before creating a memory and invoking a node there. Not all nodes in the new memory might have needed the dependency calls, though, and could have run in parallel.

Considering the tradeoffs, Aggra prioritizes modularity and latency (as two of its main features) at the cost of the inconvenient memory hierarchies and (probably safer) custom memory subclasses.

## What are the Biggest Causes for Concern?

Here are some of my current biggest causes for concern/the things I'm most uncertain about in the current version of Aggra. It's not that there aren't tests for a lot of these, it's just the complexity of each space and the lack of real-world use cases that provides pause.
* Latency inefficiencies -- there's definitely framework-overhead in what Aggra does compared to a typical program (e.g. storing and retrieving results from a central memory). How much is there? Is there any low-hanging fruit that can be optimized?
* Exceptions -- how practical is the current exception propagation mechanism? How much overhead does it add compared to programs with a successful response?
* Temporal orderings -- is Aggra globally implementing all of the temporal orderings it promises in [the advanced wiki](https://graydavid.github.io/aggra-guide/advanced/advanced.html).
* Edge cases causing stalls -- are there any edge cases in the framework that lead to node calls that don't complete, stalling the entire graph?
* Complexity -- how easy is it to use Aggra? Is everything as self explanatory as possible? What improvements can be made in the API?

## What are Future TODO Options?

Here are some thoughts for future directions to explore with Aggra:
* More examples/comprehensive test cases -- there should be more examples and test cases for bigger, realistic applications. These could address the causes for concern as well as provide a testing platform to verify new changes. [The Aggra Examples](https://github.com/graydavid/aggra-examples) project is probably a good place for this new code to live.
* Better description of abandoned node calls -- GraphCalls allow themselves to be abandoned if not all node calls are finished by some timeout. Once abandoned, there's no built-in way to figure out what's the problem/what node calls are still running. This could be useful debugging information. One idea: create an observer to keep track of replies as they're created and completed; replies created but not completed would point the way. Another idea: traverse over all replies in all memories and figure out which replies aren't complete, yet; this would probably also require an observer to track the created/relevant memories to look into.
* Graph visualization -- since graph are static and traversable, it's possible and useful to create a visualization tool. For example, we could use http://tinkerpop.apache.org/ to create a graphml file and then use https://www.yworks.com/products/yed and/or https://gephi.org/ to create a visualization.