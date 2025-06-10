# Thoughts on Microservices

No doubt, what we express here lacks originality. We simply need to contemplate the subject and write down our thoughts. Requests about microservices regularly arise in our projects, and we have to explain our position - to clients, to management, and, most importantly, to ourselves.

## On the Nature of Our Projects

We help migrate large projects, usually from mainframes to today’s technologies. These often assume cloud environments and involve Java or C# on the backend and, when relevant, Angular or React on the frontend.

The projects we handle are typically essential to a client’s business, meaning they cannot be decommissioned or easily rewritten. That’s where we come in, offering automatic and verifiable approaches.

Original projects usually follow the classic client-server-database model. They often have a UI, services, or batch operations. Their codebases are large and have a complex graph of connections between programs.

## On Client Expectations

Clients need continuity for their projects’ lifecycles and user experience. They want to migrate to modern technologies. Performance improvements are important. Above all, ease of development and support is paramount. Clients frequently talk about migrating to the cloud, and often equate moving to the cloud with adopting microservices.

## Microservices

What are they?  
We asked Copilot to draw a diagram of a cloud application implemented with microservices. This is what it produced:

![image](https://github.com/user-attachments/assets/2af3c4ef-8408-4469-82c8-e14dd9e05076)

Well, Copilot isn’t great at drawing such diagrams these days, but the idea is clear enough!

Can our projects be represented like this?  
Almost, but with a couple of additions:
* There are hundreds to thousands of services.
* There is shared code used by those services.

We asked Copilot to account for this, and it produced the following:

![image](https://github.com/user-attachments/assets/5e782040-7161-454f-a08f-abac7f7de1d1)

Still an ugly picture, but the idea is even clearer!

**Now, the question is:** Can we implement original applications using this idea?  
**Our answer:** It depends—on goals.

## Client Goals

Let’s try to articulate clients’ goals:
1. Applications that are equivalent after migration (except for differences related to the technology shift).
2. Easy support and development of migrated applications.
3. Good performance—preferably better and more scalable than the original applications.

**So, if one asks the purely theoretical question:**  
"Can original monolithic applications be implemented as a set of microservices in the cloud?"  
**The answer is:** Yes.

The problem with such an implementation is that it will probably not be equivalent, may be difficult to maintain, and could be slow.

1. When you split the original application into microservices, you must consider transaction boundaries. Transaction scope needs to be preserved unless you can prove, or the client confirms, that it isn’t important.

2. When you have broken the original application into microservices - likely residing in different source repositories -consider:
   * What will you do with shared code?
   * How will you change it?
   * After changing shared code, will you re-test and re-deploy all dependent microservices?
   * Or will you copy it, making it a dedicated part of a specific microservice?
   * Will you re-test all dependent microservices when a specific service changes?
   * How will you set up your debug environment?
   * How do you plan CI/CD?

3. The performance of communication between parts of an application through microservices is orders of magnitude lower than direct calls. If you introduce a split in the wrong place, you will kill performance.

If you think that the problem of refactoring applications into microservices is specific to legacy applications, we have to disagree. There are classes of applications that are monolithic by nature, or can only be decoupled into coarse-grained services.

## Not microservices but still scallable

So, what is the point of microservices. Let's quote [Wikipedia](https://en.m.wikipedia.org/wiki/Microservices):

> In software engineering, a microservice architecture is an architectural pattern that organizes an application into a collection of loosely coupled, fine-grained services that communicate through lightweight protocols. This pattern is characterized by the ability to develop and deploy services independently, improving modularity, scalability, and adaptability. However, it introduces additional complexity, particularly in managing distributed systems and inter-service communication, making the initial implementation more challenging compared to a monolithic architecture.

The very first paragraph pinpoints both strengths and weaknesses.  

In our case microservices make the solutions virtually unmanageable.

**You may ask:** But what about scalability?  
Indeed, microservises, are said, to start fast and may scale in and out on demand.  

But we may use the same scaling technique on monolith application. All cloud providers and Kebernetes support it, so no scalability is lost! That's true that monolithic application may start slower but this can be amortized with scaling parameters, like delay to scale down.

## Conclusion

There should be very strong reasons to make a decision to re-implement monolith complex application using microservices.

Consider all aspects and don't do it just because it is a popular pattern today.
