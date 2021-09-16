# Cancellation

This wiki talks about cancellation in the Aggra framework: how to get an already-running node to stop running or to prevent new nodes from running at all.

## Basics

Cancellation works a little bit differently in Aggra. You can't cancel node calls directly; rather, you can only indicate that you don't need the reply from a node call anymore (Aggra calls this "ignoring" a reply). Aggra then tries to figure out if every possible consumer of the reply has ignored it. If so, then Aggra will initiate cancellation. (Note on terminology: "consumer node" is a node that consumes another node; while "consumer" indicates anything that consumes a node, including both consumer nodes and top-level, "external consumers" (exclusive to root nodes)).

This algorithm is intentional. Aggra is based around the idea of modular, independent consumers. If one consumer wants to cancel a reply and it were allowed to do that, then the reply would be cancelled for other consumers as well, all without their knowledge or approval. That would be unexpected. Instead, Aggra ensures that all consumers agree that a reply should be cancelled before Aggra allows and initiates it. Aggra organizes this agreement in terms of what it calls "signals", "triggers", and "hooks".

### Signals

Cancel signals indicate that groups of replies should be cancelled. There are three varieties of signals:

1. GraphCall -- this signal means all replies for a given GraphCall instance should be cancelled.
2. MemoryScope -- this signal means all replies for a given MemoryScope instance should be cancelled.
3. Reply -- this signal means that a given Reply instance should be cancelled.

### Triggers 

Triggers are points/conditions under which Aggra will trigger the cancel signals. Here are the supported triggers for each signal:

1. GraphCall -- there are two different triggers for this signal:
    1. When users invoke `GraphCall#triggerCancelSignal` directly.
    2. After users invoke `GraphCall#weaklyClose` and then all root replies complete
2. MemoryScope -- there are two different triggers for this signal:
    1. After all "externally-accessible" replies in a MemoryScope complete. E.g. all replies for root nodes in a Graph are the only externally-accessible replies for any GraphCall's top-level MemoryScope. Another example would be when a consumer node calls `DependencyCallingDevice#createMemoryAndCall`: the created NewMemoryDependency reply is the only externally-accessible reply for the created MemoryScope.
    2. When a MemoryScope's ancestor MemoryScope's MemoryScope cancellation signal is triggered
3. Reply -- when a consumer ignores the reply and Aggra can prove that no other consumers are interested in the reply. 
    * Ignoring happens either by `GraphCall#ignore` for external consumers and `DependencyCallingDevice#ignore` for consumer nodes. 
    * Aggra only supports a single, static proof that no other consumers are interested in an ignored reply. Specifically, there must be only one possible consumer and that consumer relationship is either a SameMemoryDependency or a NewMemoryDependency (note: external consumers of root nodes are treated as if they have a SameMemoryDependency relationship with the dependency.) These conditions are not all-encompassing: there exist other conditions where Aggra *could* create a proof, but Aggra doesn't support them, because they're too costly to implement given their benefit. That said, Aggra reserves the right to implement these proofs in the future if a simple algorithm is found. The conditions are implemented behind the publicly-accessible method `Graph#ignoringWillTriggerReplyCancelSignal`. If you rely on these conditions being true for a given node, then it would be a good idea either to create a unit test or make use of `GraphValidators#ignoringWillTriggerReplyCancelSignal` to verify that the conditions remains true for that node as your graph evolves over time.

Additionally, note that conceptually, a GraphCall signal triggering triggers all MemoryScope and Reply signals; and a MemoryScope trigger triggers all Reply signals associated with that MemoryScope. (Implementation-wise, Aggra may or may not do this, but logically, it's true, and a useful fact to keep in mind.)

### Hooks

Hooks are mechanisms Aggra provides to detect signals and take the appropriate action. Aggra provides both passive and active hooks. With "passive" hooks, the cancel signal is triggered, and a status is set. Readers must poll that status and take action themselves. With "Active" hooks, nodes register the action to take with Aggra, and Aggra executes that action after triggering the cancel signal. (Note: in either case, Aggra is free to ignore signals if they would have no practical effect (e.g. a Reply cancel signal sent to an already-complete Reply.)

Here are the supported hooks that Aggra provides:

1. Before priming phase (Passive) -- this is a passive hook, where Aggra is the reader itself of the GraphCall and MemoryScope cancel signals. At the start of every node call, before the priming phase, Aggra polls those cancel signals. If they're triggered, the reply will be completed immediately with an encountered exception of CancellationException, and both the priming and behavior phases will be skipped.
2. Between the priming and behavior phases (Passive) -- this is similar to the previous hook, except that it takes place *after* the priming phase. Again, Aggra polls the GraphCall and MemoryScope signals; however, this time it also polls the Reply signal as well (since the node call's reply has been published to consumers in the meantime). If any of those signals are triggered (and the node supports those triggers), the reply will be completed immediately with an encountered exception of CancellationException, and the behavior phase will be skipped.
3. Composite cancel signal (Passive) -- this is a passive hook, where Aggra passes a composite cancel signal to a Behavior variant. This composite signal answers whether any of the GraphCall, MemoryScope, or Reply cancellation signals have been triggered. The idea is that the Behavior can poll this signal at its leisure and decide what to do with the signal response: i.e. it's completely up to the Behavior what to do with this information/what cancellation means.
4. Custom cancel action (Active) -- this is an active hook. As with the composite cancel signal hook, this hook also includes the same composite cancel signal in the call to the node's Behavior. This variant of Behavior also returns the standard CompetionStage response that a normal Behavior returns. Building on that, this hook also requires a custom cancel action that can be called when the Behavior's CompletionStage response should be cancelled. Lastly, this hook also allows this action to support cancellation through interrupts or not. Interrupt support is a separate feature, because it requires Aggra to isolate interrupts to targeted behaviors and not leak to dependency calls or after the behavior has finished. 

### Support

Different types of nodes support different hooks:

* Input nodes and nodes built with `Node.CommunalBuilder#build` -- all built-in/common nodes available through Aggra fall into this category. These nodes support the "before priming phase" and "between the priming and behavior phase" hooks. However, these nodes specifically choose not to support the Reply signal in the second hook. This is an intentional, efficiency-related decision given the cost of supporting it vs. the effect it would have. For more info, see the `build` method documentation.
* Nodes built with `Node.CommunalBuilder#buildWithCompositeCancelSignal` -- Aggra provides no built-in nodes of this category. These nodes support the same hooks as the previous category in addition to the "composite cancel signal" hook.
* Nodes built with `Node.CommunalBuilder#buildWithCustomCancelAction` -- Aggra provides no built-in nodes of this category. These nodes support the same hooks as the previous category in addition to the "custom cancel action" hook.

Given a random node, users can query what hooks it supports by inspecting the node's CancelMode.

Why doesn't every node type just support all the hooks? Cost and complexity. Each category from standard to composite signal to custom action adds synchronization costs (and adding interrupt support to the custom action adds even more). In addition, each category adds another level of complexity to detecting and responding to signals. Aggra allows users to choose what features they want and accept the cost. With all standard nodes, the only practical expense is reading/writing the status of the GraphCall signal (Aggra currently supports the GraphCall signal for all GraphCalls; Aggra could implement a non-cancellable GraphCall that would allow users to bypass this cost, but the tradeoffs don't warrant this feature for now).

## Examples

Next, let's walk through examples for each category of node.

### Standard

Standard nodes support the GraphCall and MemoryScope cancel signals, so let's take a look at each.

#### GraphCall Signal

This first example shows how a GraphCall cancel signal can function with standard nodes.

```java
public class GraphCallStandard {
    // Every Graph needs a Memory subclass
    private static class ExampleMemory extends Memory<Void> {
        private ExampleMemory(MemoryScope scope) {
            super(scope, CompletableFuture.completedFuture(null), Set.of(), () -> new ConcurrentHashMapStorage());
        }
    }

    // Create the Graph and nodes (static is used for this example; Spring/Guice are just as valid)
    private static final Node<ExampleMemory, String> NODE_1;
    private static final Node<ExampleMemory, String> NODE_2;
    private static final GraphCall.NoInputFactory<ExampleMemory> GRAPH_CALL_FACTORY;
    static {
        // Create the nodes in the graph
        NODE_1 = FunctionNodes.synchronous(Role.of("BeNode1"), ExampleMemory.class).getValue("node-1");
        NODE_2 = FunctionNodes.synchronous(Role.of("BeNode2"), ExampleMemory.class).getValue("node-2");

        // Create the Graph and a convenient GraphCall factory
        Graph<ExampleMemory> graph = Graph.fromRoots(Role.of("GraphCallStandardGraph"), Set.of(NODE_1, NODE_2));
        GRAPH_CALL_FACTORY = GraphCall.NoInputFactory.from(graph, ExampleMemory::new);
    }

    public static void main(String args[]) {
        Observer observer = Observer.doNothing(); // We don't want to observe any node calls
        GraphCall<ExampleMemory> graphCall = GRAPH_CALL_FACTORY.openCancellableCall(observer);

        Reply<String> reply1 = graphCall.call(NODE_1);
        graphCall.triggerCancelSignal();
        Reply<String> reply2 = graphCall.call(NODE_2);

        CompletableFuture<GraphCall.State> doneOrAbandoned = graphCall.weaklyCloseOrAbandonOnTimeout(5,
                TimeUnit.SECONDS);
        handleState(doneOrAbandoned.join());

        System.out.println("Reply 1: " + reply1);
        System.out.println("Reply 2: " + reply2);
        System.out.println("Reply 2 Exception: " + reply2.getFirstNonContainerExceptionNow());
    }

    private static void handleState(GraphCall.State state) {
        if (state.isAbandoned()) {
            System.out.println(
                    "Error: we had to abandon graph call. Some processes may still be running in the background");
        }

        state.getUnhandledExceptions().stream().forEach(t -> t.printStackTrace());
    }
}
```

In the above example, there are two root nodes. The program creates a GraphCall, calls node 1, triggers the GraphCall cancel signal, and then calls node 2. Both nodes are simple and should easily produce results. However, because node 2 was called after the signal was triggered, because of the "before priming phase" hook, the node 1 reply is completed immediately with a CancellationException. The output from the program is below:

```
Reply 1: io.github.graydavid.aggra.core.Reply$NonresponsiveToCancelReply@3cbbc1e0[backing=java.util.concurrent.CompletableFuture@13c78c0b[Completed normally];result=node-1]
Reply 2: io.github.graydavid.aggra.core.Reply$NonresponsiveToCancelReply@35fb3008[backing=java.util.concurrent.CompletableFuture@153f5a29[Completed exceptionally: java.util.concurrent.CompletionException: Node call failed: [ExampleMemory] [SynchronousFunction] BeNode2]]
Reply 2 Exception: Optional[java.util.concurrent.CancellationException: Cancelling Node 'BeNode2' after detecting cancellation signals: '[ExampleMemory@3cda1055(RootResponsiveToPassiveCancelMemoryScope@7a5d012c)]']
```

#### MemoryScope Signal

This second example shows how a GraphCall MemoryScope signal can function with standard nodes.

![memory-scope-standard](https://github.com/graydavid/aggra-guide/blob/gh-pages/cancellation/memory-scope-standard.png)

```java
public class MemoryScopeStandard {
    // Every Graph needs a Memory subclass
    private static class ParentMemory extends Memory<Void> {
        private ParentMemory(MemoryScope scope) {
            super(scope, CompletableFuture.completedFuture(null), Set.of(), () -> new ConcurrentHashMapStorage());
        }
    }

    private static class ChildMemory extends Memory<Void> {
        private ChildMemory(MemoryScope scope) {
            super(scope, CompletableFuture.completedFuture(null), Set.of(), () -> new ConcurrentHashMapStorage());
        }
    }

    private static class GrandchildMemory extends Memory<Void> {
        private GrandchildMemory(MemoryScope scope) {
            super(scope, CompletableFuture.completedFuture(null), Set.of(), () -> new ConcurrentHashMapStorage());
        }
    }

    // Create the Graph and nodes (static is used for this example; Spring/Guice are just as valid)
    private static final Node<ParentMemory, Void> PARENT_NODE;
    private static final GraphCall.NoInputFactory<ParentMemory> GRAPH_CALL_FACTORY;
    static {
        // Create the nodes in the graph
        Node<GrandchildMemory, Void> doNothing = FunctionNodes.synchronous(Role.of("DoNothing"), GrandchildMemory.class)
                .run(() -> {
                });
        Node.CommunalBuilder<ChildMemory> callDoNothingBuilder = Node.communalBuilder(ChildMemory.class);
        NewMemoryDependency<GrandchildMemory, Void> consumeDoNothing = callDoNothingBuilder
                .newMemoryDependency(doNothing);
        Node<ChildMemory, Void> callDoNothingUntilFailure = callDoNothingBuilder.type(Type.generic("DoNothingCalling"))
                .role(Role.of("CallDoNothingUntilFailure"))
                .build(callUntilFailure(consumeDoNothing));
        Node<ChildMemory, Void> timeLimitedDoNothingUntilFailure = TimeLimitNodes
                .startNode(Role.of("TimeLimitedDoNothingUntilFailure"), ChildMemory.class)
                .callerThreadExecutor()
                .timeout(1, TimeUnit.MILLISECONDS)
                .timeLimitedCall(callDoNothingUntilFailure);
        PARENT_NODE = MemoryTripNodes.startNode(Role.of("CreateChildAndStartDoNothingUntilFailure"), ParentMemory.class)
                .createMemoryNoInputAndCall((scope, parent) -> new ChildMemory(scope),
                        timeLimitedDoNothingUntilFailure);

        // Create the Graph and a convenient GraphCall factory
        Graph<ParentMemory> graph = Graph.fromRoots(Role.of("MemoryScopeStandardGraph"), Set.of(PARENT_NODE));
        GRAPH_CALL_FACTORY = GraphCall.NoInputFactory.from(graph, ParentMemory::new);
    }

    private static Behavior<ChildMemory, Void> callUntilFailure(NewMemoryDependency<GrandchildMemory, Void> node) {
        return device -> CompletableFuture.runAsync(() -> {
            int numCalls = 0;
            Reply<Void> doNothingReply = null;
            do {
                doNothingReply = device.createMemoryNoInputAndCall((scope, parent) -> new GrandchildMemory(scope),
                        node);
                numCalls++;
            } while (!doNothingReply.isCompletedExceptionally());
            System.out.println("Able to make this many calls to doNothing: " + numCalls);
            System.out.println("DoNothing Exception: " + doNothingReply.getFirstNonContainerExceptionNow());
        });
    }

    public static void main(String args[]) {
        Observer observer = Observer.doNothing(); // We don't want to observe any node calls
        GraphCall<ParentMemory> graphCall = GRAPH_CALL_FACTORY.openCancellableCall(observer);

        CompletableFuture<Reply<Void>> parentReply = graphCall.finalCallAndWeaklyCloseOrAbandonOnTimeout(PARENT_NODE, 5,
                TimeUnit.SECONDS, MemoryScopeStandard::handleCallState);

        parentReply.join();
        System.out.println("Program done");
    }

    private static void handleCallState(GraphCall.State state, Throwable throwable, Reply<Void> finalReply) {
        if (state.isAbandoned()) {
            System.out.println(
                    "Error: we had to abandon graph call. Some processes may still be running in the background");
        }

        if (throwable == null) {
            state.getUnhandledExceptions().stream().forEach(t -> t.printStackTrace());
        } else {
            throwable.printStackTrace();
        }
    }
}
```

In the above example, there are three memories. First, there's a GrandchildMemory. It contains a single DoNothing node that does nothing. Second, there's ChildMemory. It contains two nodes: CallDoNothingUntilFailure, which continuously creates instances of GrandchildMemory and calls DoNothing there until DoNothing fails; and TimeLimitedDoNothingUntilFailure which calls CallDoNothingUntilFailure, allowing 1 ms for it to return before moving on. Third, there a ParentMemory. It contains a single node that creates a ChildMemory and calls TimeLimitedDoNothingUntilFailure.

If the time-limit node didn't set a timeout, then it would wait forever for CallDoNothingUntilFailure to finish, since there's no reason otherwise that DoNothing would fail. The GraphCall would eventually be abandoned after waiting for 5 seconds. However, because the time-limit node completes after 1-ms, this triggers the ChildMemory instance's MemoryScope cancel signal, which in turn triggers all GrandchildMemory (a child of ChildMemory) instance's (both existing and new) MemoryScope cancel signal. DoNothing eventually detects that signal and automatically completes with a CancellationException due to either the "Before priming phase" or "Between the priming and behavior phases" hooks (depending on timing).

Here's an example output from a run:

```
Able to make this many calls to doNothing: 356
DoNothing Exception: Optional[java.util.concurrent.CancellationException: Cancelling Node 'DoNothing' after detecting cancellation signals: '[GrandchildMemory@35b236f2(NonresponsiveToCancelMemoryScope@1cbccfd8)]']
Program done
```

### Composite Cancel Signal

Since we haven't seen the Reply cancel signal come into play yet, and since th composite cancel signal is the first one to support it, let's walk through an example using it.

```java
public class ReplyCompositeSignal {
    // Every Graph needs a Memory subclass
    private static class ExampleMemory extends Memory<Void> {
        private ExampleMemory(MemoryScope scope) {
            super(scope, CompletableFuture.completedFuture(null), Set.of(), () -> new ConcurrentHashMapStorage());
        }
    }

    // Create the Graph and nodes (static is used for this example; Spring/Guice are just as valid)
    private static final Node<ExampleMemory, Void> ENTRY_NODE;
    private static final GraphCall.NoInputFactory<ExampleMemory> GRAPH_CALL_FACTORY;
    static {
        // Create the nodes in the graph
        Node<ExampleMemory, Integer> loopFor100 = nodeLoopingForNumLoops(100);
        Node<ExampleMemory, Integer> loopFor1000000 = nodeLoopingForNumLoops(1000000);
        Node.CommunalBuilder<ExampleMemory> entryNodeBuilder = Node.communalBuilder(ExampleMemory.class);
        SameMemoryDependency<ExampleMemory, Integer> consume100 = entryNodeBuilder
                .sameMemoryUnprimedDependency(loopFor100);
        SameMemoryDependency<ExampleMemory, Integer> consume1000000 = entryNodeBuilder
                .sameMemoryUnprimedDependency(loopFor1000000);
        ENTRY_NODE = entryNodeBuilder.type(Type.generic("ChooseAndCancel"))
                .role(Role.of("ChooseBetween100And1000000"))
                .build(device -> {
                    Reply<Integer> reply100 = device.call(consume100);
                    Reply<Integer> reply1000000 = device.call(consume1000000);
                    return Reply.anyOfBacking(reply100, reply1000000).thenApply(ignore -> {
                        device.ignore(reply100);
                        device.ignore(reply1000000);

                        System.out.println("loopFor100 looped for " + reply100.join());
                        System.out.println("loopFor1000000 looped for " + reply1000000.join());
                        return null;
                    });
                });

        // Create the Graph and a convenient GraphCall factory
        Graph<ExampleMemory> graph = Graph.fromRoots(Role.of("ReplyCompositeSignal"), Set.of(ENTRY_NODE));
        GRAPH_CALL_FACTORY = GraphCall.NoInputFactory.from(graph, ExampleMemory::new);
    }

    private static Node<ExampleMemory, Integer> nodeLoopingForNumLoops(int numLoops) {
        return Node.communalBuilder(ExampleMemory.class)
                .type(Type.generic("NumLooping"))
                .role(Role.of("WaitForLoops" + numLoops))
                .graphValidatorFactory(GraphValidators.ignoringWillTriggerReplyCancelSignal())
                .buildWithCompositeCancelSignal((device, signal) -> {
                    return CompletableFuture.supplyAsync(() -> {
                        int i = 0;
                        for (; i < numLoops && !signal.read(); ++i) {
                            // Do nothing busy work
                            int j = 0;
                            j = j + 1;
                        }
                        return i;
                    });
                });
    }

    public static void main(String args[]) {
        Observer observer = Observer.doNothing(); // We don't want to observe any node calls
        GraphCall<ExampleMemory> graphCall = GRAPH_CALL_FACTORY.openCancellableCall(observer);

        CompletableFuture<Reply<Void>> doneOrAbandonedReply = graphCall.finalCallAndWeaklyCloseOrAbandonOnTimeout(
                ENTRY_NODE, 5, TimeUnit.SECONDS, ReplyCompositeSignal::handleCallState);

        doneOrAbandonedReply.join().join();
        System.out.println("Program done");
    }

    private static void handleCallState(GraphCall.State state, Throwable throwable, Reply<Void> finalReply) {
        if (state.isAbandoned()) {
            System.out.println(
                    "Error: we had to abandon graph call. Some processes may still be running in the background");
        }

        if (throwable == null) {
            state.getUnhandledExceptions().stream().forEach(t -> t.printStackTrace());
        } else {
            throwable.printStackTrace();
        }
    }
}
```

In the above example, there are three nodes. WaitForLoops100 and WaitForLoops1000000 loop their respective number of times, reading the passed-in composite cancel signal each loop and finishing early if it's true. Note that since each node depends on receiving the Reply cancel signal, each adds the `GraphValidators#ignoringWillTriggerReplyCancelSignal` for-node graph validator factory. Then, there's the ChooseBetween100And1000000 which calls both of the other nodes, waits for any one to complete, and then ignores both (potentially triggering the Reply cancel signal for whichever one is running).

Here's an example output from a run (which shows that loopFor1000000 was cut short, as expected):

```
loopFor100 looped for 100
loopFor1000000 looped for 8313
Program done
```

### Custom Cancel Action

Custom cancel action nodes can either not-support or support interrupts for cancellation. Before jumping into some examples for each, let's cover some general rules that all custom cancel actions should follow.

* The custom action should always be fast: don't ever wait around. If this method takes a long time, it will delay other Node calls and Aggra logic.
* The custom action should *never* *just* call `CompletableFuture#cancel` on the response returned from the behavior. CompletableFuture's cancel method does nothing but complete the CompletableFuture with a CancellationException. So, if the custom cancel action simply calls cancel, then it would leave the behavior still running in the background, and both Aggra and the user would be unaware. This is another variant of problems leaving unaccounted-for background tasks running as discussed in [the advanced wiki](https://github.com/graydavid/aggra-guide/blob/gh-pages/advanced/advanced.html).
* A generalization of the previous point is that the custom action and its corresponding behavior should cooperate in such a way as to minimize the amount of work still running after the behavior's response is completed.
* The custom action should not cause the cancellation of any logic that would complete the behavior's response. Said another way, this method and the node's behavior should cooperate such that the behavior's response is guaranteed to complete. One easy mistake to make here is to use an ExecutorService to run the  task that completes the behavior's response, save the future from that submission, and implement this method by calling {@link Future#cancel(boolean)}. The problem is that cancelling a future may cause the associated task never to run at all, which means that the behavior's response would never complete, causing GraphCall processing to halt. What's worse, this is timing dependent: it's only if the call to cancel happens before the task is run that the task will never be run; otherwise, if the task has already started (which will be true in probably the majority of cases), the running task will receive the cancellation signal and complete the behavior. This is an antipattern you should watch out for. See the Interrupt-Supporting example below to see an example of how to avoid it.
* The custom action should only trigger interrupts after declaring that it will do so. The custom cancel action declares this in the return value of its `cancelActionMayInterruptIfRunning` method, which users must implement. This is necessary so that Aggra knows to do the paperwork necessary to isolate interrupts to their intended behaviors and not any dependency calls or random code that's executed after the behavior finishes.
* The custom action BehaviorWithCustomCancelAction extends InterruptModifier, which defines operations that Aggra uses for doing the paperwork necessary to isolate interrupts. Users are free to override this behavior to turn off that isolation when it doesn't make sense. E.g. at the end of a program, shutting down an executor service might cause it to trigger all of its threads' interrupts. In this case, we wouldn't want Aggra to isolate *those* interrupts.

With that out of the way, let's take a look at a few examples.

#### Non-Interrupt-Supporting

This first example shows how a non-interrupt-supporting custom cancel action can work.

```java
public class NonInterruptCustomAction {
    // Every Graph needs a Memory subclass
    private static class ExampleMemory extends Memory<Void> {
        private ExampleMemory(MemoryScope scope) {
            super(scope, CompletableFuture.completedFuture(null), Set.of(), () -> new ConcurrentHashMapStorage());
        }
    }

    /**
     * java.net.Socket blocks during reading. Interrupts won't stop that. Closing will. Simulate this with a simple
     * Socket-like class backed by a CountDownLatch.
     */
    private static class Socket {
        private final CountDownLatch stopReading = new CountDownLatch(1);

        public void read() {
            try {
                stopReading.await();
            } catch (InterruptedException e) {
                Thread.interrupted();
                throw new RuntimeException(e);
            }
        }

        public void close() {
            stopReading.countDown();
        }
    }

    // Create the Graph and nodes (static is used for this example; Spring/Guice are just as valid)
    private static final Node<ExampleMemory, Void> ENTRY_NODE;
    private static final GraphCall.NoInputFactory<ExampleMemory> GRAPH_CALL_FACTORY;
    static {
        // Create the nodes in the graph
        Node<ExampleMemory, Void> readFromSocket = Node.communalBuilder(ExampleMemory.class)
                .type(Type.generic("SocketReading"))
                .role(Role.of("ReadFromSocket"))
                .graphValidatorFactory(GraphValidators.ignoringWillTriggerReplyCancelSignal())
                .buildWithCustomCancelAction(new BehaviorWithCustomCancelAction<>() {
                    @Override
                    public CustomCancelActionBehaviorResponse<Void> run(DependencyCallingDevice<ExampleMemory> device,
                            CompositeCancelSignal signal) {
                        Socket socket = new Socket();
                        CompletableFuture<Void> runSocketResponse = CompletableFuture.runAsync(socket::read);
                        CustomCancelAction action = mayInterrupt -> socket.close();
                        return new CustomCancelActionBehaviorResponse<>(runSocketResponse, action);
                    }

                    @Override
                    public boolean cancelActionMayInterruptIfRunning() {
                        // This node doesn't use interrupts for cancellation
                        return false;
                    }
                });
        Node.CommunalBuilder<ExampleMemory> entryNodeBuilder = Node.communalBuilder(ExampleMemory.class);
        SameMemoryDependency<ExampleMemory, Void> consumeReadFromSocket = entryNodeBuilder
                .sameMemoryUnprimedDependency(readFromSocket);
        ENTRY_NODE = entryNodeBuilder.type(Type.generic("ReadingAndCancelling"))
                .role(Role.of("ReadFromSocketAndCancel"))
                .build(device -> {
                    Reply<Void> socketReply = device.call(consumeReadFromSocket);
                    return socketReply.toCompletableFuture().orTimeout(200, TimeUnit.MILLISECONDS).exceptionally(t -> {
                        device.ignore(socketReply);
                        return socketReply.join();
                    });
                });

        // Create the Graph and a convenient GraphCall factory
        Graph<ExampleMemory> graph = Graph.fromRoots(Role.of("NonInterruptCustomAction"), Set.of(ENTRY_NODE));
        GRAPH_CALL_FACTORY = GraphCall.NoInputFactory.from(graph, ExampleMemory::new);
    }

    public static void main(String args[]) {
        Observer observer = Observer.doNothing(); // We don't want to observe any node calls
        GraphCall<ExampleMemory> graphCall = GRAPH_CALL_FACTORY.openCancellableCall(observer);
        long start = System.nanoTime();

        CompletableFuture<Reply<Void>> doneOrAbandonedReply = graphCall.finalCallAndWeaklyCloseOrAbandonOnTimeout(
                ENTRY_NODE, 5, TimeUnit.SECONDS, NonInterruptCustomAction::handleCallState);

        doneOrAbandonedReply.join();
        Duration duration = Duration.ofNanos(System.nanoTime() - start);
        System.out.println("Program done after " + duration.toMillis() + " ms");
    }

    private static void handleCallState(GraphCall.State state, Throwable throwable, Reply<Void> finalReply) {
        if (state.isAbandoned()) {
            System.out.println(
                    "Error: we had to abandon graph call. Some processes may still be running in the background");
        }

        if (throwable == null) {
            state.getUnhandledExceptions().stream().forEach(t -> t.printStackTrace());
        } else {
            throwable.printStackTrace();
        }
    }
}
```

The above example imagines dealing with a java.net.Socket-like class. According to [Java Concurrency in Practice](https://jcip.net/), Sockets block when reading (or at least its InputStream does), but that blocking is not responsive to interrupts. Instead, to cancel a reading/writing Socket, you can close it. This example constructs a similarly-working Socket-like class based around a CountDownLatch: the `read` operation blocks on the latch, and the `close` operation counts down to free it.

There are two nodes in this example. First, ReadFromSocket creates a new Socket, initiates a blocking-read on another thread, and then creates a custom cancel action to close the Socket. Because we're relying on the Reply cancel signal to initiate an early cancellation, as in the last example, we add a `GraphValidators#ignoringWillTriggerReplyCancelSignal` for-node graph validator factory to it. The second node is ReadFromSocketAndCancel, which calls ReadFromSocket, waits for 200 ms, and then if a TimeoutException happens, it ignores the ReadFromSocket call (which triggers ReadFromSocket's Reply cancel signal and thus custom action). So, the entire program should last around 200 ms, which is what we find from an example output:

```
Program done after 252 ms
```

#### Interrupt-Supporting

This second example shows how a non-interrupt-supporting custom cancel action can work.

```java
public class InterruptCustomAction {
    // Every Graph needs a Memory subclass
    private static class ExampleMemory extends Memory<Void> {
        private ExampleMemory(MemoryScope scope) {
            super(scope, CompletableFuture.completedFuture(null), Set.of(), () -> new ConcurrentHashMapStorage());
        }
    }

    // Create the Graph and nodes (static is used for this example; Spring/Guice are just as valid)
    private static final Node<ExampleMemory, Boolean> ENTRY_NODE;
    private static final GraphCall.NoInputFactory<ExampleMemory> GRAPH_CALL_FACTORY;
    static {
        // Create an executor using daemon threads so that it won't block program shutdown
        Executor executor = Executors.newSingleThreadExecutor(new ThreadFactory() {
            public Thread newThread(Runnable r) {
                Thread t = Executors.defaultThreadFactory().newThread(r);
                t.setDaemon(true);
                return t;
            }
        });

        // Create the nodes in the graph
        Node<ExampleMemory, Boolean> awaitNeverCountDownLatch = Node.communalBuilder(ExampleMemory.class)
                .type(Type.generic("NeverCountDownLatchReading"))
                .role(Role.of("AwaitNeverCountDownLatch"))
                .graphValidatorFactory(GraphValidators.ignoringWillTriggerReplyCancelSignal())
                .buildWithCustomCancelAction(new BehaviorWithCustomCancelAction<>() {
                    @Override
                    public CustomCancelActionBehaviorResponse<Boolean> run(
                            DependencyCallingDevice<ExampleMemory> device, CompositeCancelSignal signal) {
                        CompletableFuture<Boolean> response = new CompletableFuture<>();
                        CompletingOnCancelFutureTask<?> task = new CompletingOnCancelFutureTask<>(() -> {
                            awaitNeverCountDownLatchTowardsFuture(response);
                            return null;
                        }, response);
                        executor.execute(task);
                        CustomCancelAction action = mayInterrupt -> task.cancel(mayInterrupt);
                        return new CustomCancelActionBehaviorResponse<>(response, action);
                    }

                    private void awaitNeverCountDownLatchTowardsFuture(CompletableFuture<Boolean> futureToComplete) {
                        Try.callCatchThrowable(() -> {
                            CountDownLatch latch = new CountDownLatch(1);
                            return latch.await(1, TimeUnit.SECONDS);
                        }).consume((bool, t) -> {
                            if (t == null) {
                                futureToComplete.complete(bool);
                            } else {
                                futureToComplete.completeExceptionally(t);
                            }
                        });
                    }

                    @Override
                    public boolean cancelActionMayInterruptIfRunning() {
                        // This node does use interrupts for cancellation
                        return true;
                    }
                });
        Node.CommunalBuilder<ExampleMemory> entryNodeBuilder = Node.communalBuilder(ExampleMemory.class);
        SameMemoryDependency<ExampleMemory, Boolean> consumeAwaitNeverLatch = entryNodeBuilder
                .sameMemoryUnprimedDependency(awaitNeverCountDownLatch);
        ENTRY_NODE = entryNodeBuilder.type(Type.generic("AwaitingAndCancelling"))
                .role(Role.of("AwaitNeverCountDownLatchAndCancel"))
                .build(device -> {
                    Reply<Boolean> awaitReply = device.call(consumeAwaitNeverLatch);
                    return awaitReply.toCompletableFuture().orTimeout(200, TimeUnit.MILLISECONDS).exceptionally(t -> {
                        device.ignore(awaitReply);
                        return awaitReply.join();
                    });
                });

        // Create the Graph and a convenient GraphCall factory
        Graph<ExampleMemory> graph = Graph.fromRoots(Role.of("InterruptCustomAction"), Set.of(ENTRY_NODE));
        GRAPH_CALL_FACTORY = GraphCall.NoInputFactory.from(graph, ExampleMemory::new);
    }

    /**
     * Completes a provided CompletableFuture if this task is cancelled. This helps make sure the CompletableFuture
     * completes both on normal running and cancellation of the task.
     */
    private static class CompletingOnCancelFutureTask<T> extends FutureTask<T> {
        private final CompletableFuture<?> futureToComplete;

        private CompletingOnCancelFutureTask(Callable<T> callable, CompletableFuture<?> futureToComplete) {
            super(callable);
            this.futureToComplete = futureToComplete;
        }

        @Override
        protected void done() {
            if (isCancelled()) {
                futureToComplete.completeExceptionally(new CancellationException());
            }
        }
    }

    public static void main(String args[]) {
        Observer observer = Observer.doNothing(); // We don't want to observe any node calls
        GraphCall<ExampleMemory> graphCall = GRAPH_CALL_FACTORY.openCancellableCall(observer);
        long start = System.nanoTime();

        CompletableFuture<Reply<Boolean>> doneOrAbandonedReply = graphCall.finalCallAndWeaklyCloseOrAbandonOnTimeout(
                ENTRY_NODE, 5, TimeUnit.SECONDS, InterruptCustomAction::handleCallState);

        doneOrAbandonedReply.join();
        Duration duration = Duration.ofNanos(System.nanoTime() - start);
        System.out.println("Program done after " + duration.toMillis() + " ms");
    }

    private static void handleCallState(GraphCall.State state, Throwable throwable, Reply<Boolean> finalReply) {
        if (state.isAbandoned()) {
            System.out.println(
                    "Error: we had to abandon graph call. Some processes may still be running in the background");
        }

        if (throwable == null) {
            if (!state.getUnhandledExceptions().isEmpty()) {
                System.out.println("Unhandled Exceptions:");
            }
            state.getUnhandledExceptions().stream().forEach(t -> t.printStackTrace());
        } else {
            throwable.printStackTrace();
        }
    }
```

The above example has two nodes. First, there's AwaitNeverCountDownLatch. Its behavior waits for a maximum of 1 second on a CountDownLatch that will never count down. (There's no particular reason why CountDownLatch was used other than that it support interruption.) This waiting logic is run on a separate thread via an Executor as part of a CompletingOnCancelFutureTask FutureTask (to make sure the behavior's response still completes if cancelled before running). The custom cancel action is defined such that it calls "cancel" on this CompletingOnCancelFutureTask, which will raise an interrupt on AwaitNeverCountDownLatch. AwaitNeverCountDownLatch will then stop waiting on the CountDownLatch and quickly return. This pattern shows a way for a behavior and custom action to cooperate to make sure the behavior's response always completes and minimize the amount of work still running after the behavior's response is completed (see the notes above at the top of the "Custom Cancel Action" section). (E.g. if the code used just a normal FutureTask (as a lot of ExecutorServices do by default), then the behavior's response may or may not be completed on cancellation; plus, if the custom action weren't responsive to interrupts, perhaps even blocking again after receiving one, that background work would remain running after cancellation.) 

The second node is InterruptCustomAction, which calls AwaitNeverCountDownLatch, waits for 200 ms, and then if a TimeoutException happens, it ignores the AwaitNeverCountDownLatch call (which triggers AwaitNeverCountDownLatch's Reply cancel signal and thus custom action). 

All things considered, we should expect CompletingOnCancelFutureTask to be ignored, which will cause it to be completed exceptionally and show up in the GraphCall.State's list of unhandled exceptions. Additionally, the entire program should last around 200 ms. Both are confirmed in an (abridged) example output:

```
Unhandled Exceptions:
java.util.concurrent.CompletionException: Node call failed: [ExampleMemory] [NeverCountDownLatchReading] AwaitNeverCountDownLatch
	at io.github.graydavid.aggra.core.ExceptionStrategy.newCompletionException(ExceptionStrategy.java:202)
   ...
	at java.base/java.lang.Thread.run(Thread.java:834)
Caused by: io.github.graydavid.aggra.core.CallException: Error calling node: [ExampleMemory] [NeverCountDownLatchReading] AwaitNeverCountDownLatch
****Start node call stack****
[ExampleMemory] [NeverCountDownLatchReading] AwaitNeverCountDownLatch
	[ExampleMemory] [AwaitingAndCancelling] AwaitNeverCountDownLatchAndCancel
		InterruptCustomAction
****Stop node call stack****
	at io.github.graydavid.aggra.core.ExceptionStrategy.transformToContainerException(ExceptionStrategy.java:195)
	... 28 more
Caused by: java.util.concurrent.CancellationException
	at io.github.graydavid.aggraexamples.cancellation.InterruptCustomAction$CompletingOnCancelFutureTask.done(InterruptCustomAction.java:136)
	...
	at io.github.graydavid.aggra.core.Reply$ResponsiveToCancelReply.runCancelAction(Reply.java:346)
	... 16 more
Program done after 244 ms
```