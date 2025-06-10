# Thoughts on microservices

No doubt what we express here lacks originality. We just need to contemplate on the subject and write our thoughts down, as 
requests around microservices regularly pop up in our projects, and we have to explain our position
to clients, to the management and - more importantly - to ourselves.

## On the nature of our projects

We help with migration of large projects, usually from mainframes to today’s technologies that often assume 
cloud environments and involve Java or C# on the backend and, if relevant, Angular or React on the frontend.

The projects we deal with are usually essential to a client's business, so they cannot be decommissioned or easily rewritten. 
That is where we come in with automatic and verifiable approaches.

Original projects usually follow the "classic" client-server-database scheme. They often have UI, services or batch operations. 
They have large codebases, with a complex graph of connections between programs.

## On client expectations

Clients need continuity of their projects' life and experience. They want to migrate to today's technologies. 
They want performance improvements. Ease of development and support is paramount for them. 
They often talk about migrating to the cloud. 
They know that cloud is almost equal to microservices.

## Microservices

What are they?  
We asked Copilot to draw a diagram of cloud application implemented with microservices. This is what it came with:

![image](https://github.com/user-attachments/assets/2af3c4ef-8408-4469-82c8-e14dd9e05076)

Well, Copilot is not great these days at drawing such diagrams, yet the idea is clear enough!

Can our projects be represented like this?
Almost, but with a couple of additions:
* there are from hundreds to thousands of services;
* there are shared code used by services.

We asked Copilot to account this, and it produced the following:
![image](https://github.com/user-attachments/assets/5e782040-7161-454f-a08f-abac7f7de1d1)

Still ugly picture, but idea is even more clear!
