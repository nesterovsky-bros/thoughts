# Agentic Era

It's not the first time we need massive use of AI in our code.

## Summary

This post documents our exploration of agentic AI frameworks as an alternative to our proven OpenAI Batch API approach. While pressure existed to adopt trendy agentic solutions, we remained pragmatic: our existing Batch API solution is scalable, well-understood, and well-maintained.

We evaluated **LangChain** and **LangGraph** for implementing complex AI workflows with state management and resumability. Our conclusion is nuanced:
- **LangChain** excels at simplifying AI integration through declarative service interfaces.
- **LangGraph** introduces unnecessary complexity for our use cases; pure Java with explicit state management is cleaner and easier to debug.

By combining pure Java with **LangChain** for AI calls, we achieved cleaner, more efficient code than our original Batch API implementation, while maintaining the readability and maintainability Arthur and I both value.

## Use Case

Let's formulate a use case in simple form:

- There is a folder of input data that you need to process somehow.
- One or multiple steps in such processing involve AI prompts.
- The result is a folder of output data.

## Journey So Far

For a couple of years, we've used **OpenAI Batch API** to do such processing.
Originally we used Azure OpenAI, but when we became part of IBM, we switched to **watsonx.ai**, and even re-implemented a lacking Batch API drop-in replacement service using IBM Code Engine.

Solution works, but... here the journey only begins.

### The Conversation

> **IBM presses** - you need to build agentic solutions.
>
> **Arthur tells** - what we have can be called agentic use of AI. It just works without buzz words.
>
> **I tell** - yes, that's correct, but it's not easy to implement non-trivial AI flow, including tool calls, while sticking to the Batch API.
>
> **Arthur** - not trivial but possible. Our solution is scalable, we understand it, and we easily support it. So, per the first rule of programming - don't fix something that is working!
>
> **I** - I want to explore what other solutions can give us.
>
> **Arthur, skeptically** - go on, and I'll be looking at your efforts!

### Exploring the Landscape

So, I went on discovering today's landscape of agentic AI libraries.

As you know, there are many of them - from Microsoft, from the community, and for other providers. We have our constraints. Being part of IBM means **Langchain** is the default choice. You would need to have good arguments to use something different.

Okay, we agreed with Arthur that **Langchain** is what I'll continue to explore.

Then I went forward looking at how I could implement our sample task with massive processing of folders of data.

What came naturally is **LangGraph** - the framework to implement different workflows, including cycles, parallel processing, AI calls, and checkpointing of execution.

Checkpointing looked promising - I could run the flow, save intermediate results, stop, restart, and do different things with state. This was encouraging for a long-running flow.

I've found that with some effort, you could even integrate the Batch API into it. I was positive about the right direction. Arthur wisely listened and agreed that this sounds promising.

### Language Selection

Next step was to select the language.

Several years ago I would say it's weird to select a framework and then language, but these days the world has changed.

We found that **LangGraph** has:
- Original implementation in **Python**, which is most advanced
- **TypeScript** port
- **LangGraph4j** - Java version inspired by Python

For us it was not immediately obvious what to use, as we don't start a new project but continue ongoing development. Our preferences go with **LangGraph4j** - there will be fewer moving parts, and it will integrate naturally.

### POC Sample

Now we needed hands-on experience with the technology to study it deeper and implement a simplest flow.

What I learned is that **LangGraph** is a library to define a logic flow in a semi-declarative way as a graph of edges and vertices, where:
- **Edge** - defines a unit of logic
- **Vertices** - define transitions between edges

In general, **LangGraph** has nothing to do with **LangChain**, which is an AI wrapper. You can do whatever you want with **LangGraph**, including calling AI through **LangChain**. **LangGraph** allows you to stop execution, save state somewhere (e.g., in files or in a database), and later resume.

Arthur was satisfied and told me that the flow graph resembled his old favorite BizTalk, which allowed you to define a high-level algorithm visually, and the final block could be implemented by calling real code.

All this looked promising and allowed us to implement some simple tasks:

1. An agent to perform simple math: `Sum(a, b)`
2. An agent to perform a simple cycle of sub-agents
3. An agent to perform a parallel cycle of sub-agents

I started with Task 1 - the Sum agent - and implemented it using a **LangGraph** sample. It included a map-based immutable state, graph definitions, and node handlers.

Arthur looked at it for a long time and told me that he barely understood what he saw and barely understood where the business logic (`a + b`) was.

**Verdict** - this is an obscure, unmanageable pile of code. We need something less clever if we want to understand it in half a year.

I went back and introduced some helpers to simplify agent coding in **LangGraph4j**. This allowed me to produce much smaller and concise code, which I want to quote here:

```java
package com.bphx.coolgen.agents;

import org.bsc.langgraph4j.CompileConfig;
import org.bsc.langgraph4j.CompiledGraph;
import org.bsc.langgraph4j.RunnableConfig;
import org.bsc.langgraph4j.StateGraph;

import static com.bphx.coolgen.agents.Agent.*;

import java.io.Serializable;

/// Sample agent demonstrating basic LangGraph4j usage. Receives two numbers and
/// computes their sum.
///
/// Uses state class [Data].
///
/// ## Usage
///
/// var graph = Sum.build();
/// var data = Sum.of(10.0, 32.0);
/// var result = Agent.invoke(graph, data);
/// System.out.println("Sum: " + result.result); // prints: Sum: 42.0
public final class Sum
{
  /// State class for the Sum agent. Contains input operands and the
  /// computed result.
  public static class Data implements Serializable
  {
    public Data(double a, double b)
    {
      this.a = a;
      this.b = b;
    }

    public final double a;
    public final double b;
    public double result;
  }

  public static Data of(double a, double b)
  {
    return new Data(a, b);
  }

  /// Builds and returns a compiled graph for the sum agent.
  /// 
  /// @return compiled graph ready to invoke
  /// @throws Exception if graph compilation fails
  public static CompiledGraph<State<Data>> build() throws Exception
  {
    return Agent.<Data>graph()
        .addNode("sum", node(Sum::sum))
        .addEdge(StateGraph.START, "sum")
        .addEdge("sum", StateGraph.END)
        .compile(CompileConfig.builder().build());
  }

  private static Data sum(Data data, RunnableConfig config)
  {
    data.result = data.a + data.b;

    return data;
  }
}
```

Arthur confirmed that he was able to understand that `sum()` defines the business logic, the `Data` class defines the graph state, and `build()` defines the graph itself, like this: `start -> sum -> end`.

However, neither Arthur nor I were fully satisfied.

At the next step, I implemented Task 2 - a cycle of Sum calls. This worked, but it looked as if you were building assembly with a graph, as if you need to implement:

```
start:
  index = 0;
  count = 1000;

cycle:
  if (index >= count)
    goto end;

iteration:
  result[index] = a[index] + b[index];
  index = index + 1;
  goto cycle;

end:
```

This works but doesn't look nice. Besides, debugging **LangGraph** code is an adventure.

Finally, the 3rd task was the least trivial part to implement. It turned out there is no built-in parallel cycles, and we needed to go to the basic fork-join building block in **LangGraph** to generate a dynamic graph for parallel processing.

This worked as well, but it became even more complex, involving merging states after parallel branches and uncontrolled serialization and deserialization of all accumulated states.

It did not look like a good trade. We lost clean, readable Java code in exchange for a resumable graph that is hard to understand and debug.

## Next Steps

We implemented the original task of processing a folder in parallel, where we called **LangChain** and were pleased with its AI services. You could define an interface and get its implementation from `ChatModel` (which even supports tool calls), like this:

```java
  /// AI service interface for generating HTML documentation summaries.
  ///
  /// LangChain4j creates an implementation that calls the chat model with
  /// a system prompt.
  interface Summarizer
  {
    @SystemMessage("""
        You are a world class expert in programming... 

        Create documentation for the provided code to ... 
        Use basic HTML markup, like
        <h1>, <p>, <b>, <ol>, <ul>, <li>. Don't use <html> and <body>.

        - Documentation must be terse.
        - No greetings and farewells must be used.
        - Provide a summary of the code's purpose and functionality.
        ...
        """)
    String summarize(@UserMessage String code);
  }

  ...
  var summarizer = AiServices.create(Summarizer.class, chatModel);
  ...
  var summary = summarizer.summarize(code);
```

Notice the elegance: the entire AI integration is reduced to a single interface definition and one line of instantiation. The business logic (`summarize()`) is immediately visible. Compare this to the earlier **LangGraph** Sum agent with its graph definitions, node handlers, and state management—this is why we prefer it.

### Sharing Our Approach

We're planning to share this pattern with our team as the recommended path forward for AI integration. Rather than pursuing the latest agentic frameworks, teams should:

1. **Start with explicit state management** in their language of choice
2. **Use focused AI libraries** like **LangChain4j** for clean abstraction
3. **Avoid graph frameworks unless truly needed** for distributed resumability
4. **Prioritize code clarity** over framework comprehensiveness

### Future Roadmap

The Java + **LangChain** pattern scales well for:
- **Batch processing** of large folders with parallel **LangChain** calls
- **Multi-step AI workflows** (chain multiple AI services together)
- **Tool-augmented AI** (extending ChatModel with custom tools)
- **Integration with existing systems** where Java is already the lingua franca

We're confident this approach will serve our organization better than chasing trending frameworks.

## Key Learnings

1. **Buzzwords ≠ Solutions**: The pressure to "build agentic solutions" made us evaluate tools strictly on merit, not hype. Sometimes "agentic" is just marketing.

2. **Abstraction Layers Matter**: **LangChain**'s declarative approach (`@SystemMessage`, `AiServices`) hides complexity elegantly. **LangGraph**'s explicit node-edge model exposed too many low-level details for our domain.

3. **Code Readability Wins Long-term**: Arthur's visceral reaction to the initial **LangGraph** code—"barely understand where the business logic is"—reminded us that clever code is harder to maintain than straightforward code (even with AI).

4. **Pragmatism Over Features**: Our Batch API solution already solved resumability and scalability. Adopting a framework for features you don't need creates unnecessary technical debt.

5. **Pure Language + Focused Libraries**: Combining plain Java + **LangChain** gave us the best of both worlds: explicit control over algorithms and clean AI abstraction.

### When LangGraph Makes Sense

Before we dismiss **LangGraph** entirely, it's worth noting when it might be the right choice:
- **Truly distributed systems** where processes run on different machines and need to resume
- **Long-running, unattended workflows** that span days or weeks and need checkpointing (though native java implementation gave us flexibility on checkpoints)
- **Complex branching logic** with many conditional paths that benefit from visual representation
- **Systems requiring audit trails** of every state transition

For our use case—batch processing with intermediate results we can manage explicitly—**LangGraph** was overkill.

## Conclusion

After working through these explorations, we reached a **nuanced but clear conclusion**:

- **LangChain** - Excellent. Provides clean abstractions for AI integration without forcing architectural decisions.
- **LangGraph** - Not a fit for our use cases. Its complexity and explicit state serialization overhead outweigh the benefits for batch processing. For truly distributed, long-running workflows, it might be appropriate, but we have simpler solutions.

### The Better Way Forward

We re-implemented several processes that used the OpenAI Batch API in pure Java + **LangChain**, and even extended the logic to multi-step AI calls. The results:
- **Cleaner code** than the original Batch API implementation
- **More powerful** (support for complex multi-turn conversations)
- **Faster execution** (parallel processing without batching delays)
- **Easier to maintain** (straightforward Java instead of graph abstractions)

Thus, we believe our decision to explore and eventually adopt the Java + **LangChain** pattern was the right one. We gained the benefits of modern AI frameworks without sacrificing the simplicity and maintainability that Arthur and I both prioritize.
