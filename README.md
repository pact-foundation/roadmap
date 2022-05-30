# Developer Relations @ Pact

> Evangelism / Enablement / Advocacy / Community

Good communities are places where people :heart: to be.

We are committed to making our Pact community, the go to place for all things contract-testing but also for people to get to know each, connect, meet, share experiences and become life-long friends.

As a Developer Advocate / Community Shepherd, I know that pact foundation has huge amount of GitHub repositories and maintainers hold a huge amount of weight on our shoulders, to ensure that people can use Pact day-in day-out to protect their deployments. It's a hard job to keep track of.

Let me make the Pact landscape a little easier to navigate

- [Developer Relations @ Pact - Evangelism / Enablement / Advocacy / Community](#developer-relations--pact---evangelism--enablement--advocacy--community)
  - [Links](#links)
  - [Get involved!](#get-involved)
    - [📙 Docs](#-docs)
    - [🚀 Code](#-code)
    - [Roadmap / Feature Requests](#roadmap--feature-requests)
    - [Recipes](#recipes)
    - [Workshops](#workshops)
    - [Blogs, Videos & Articles](#blogs-videos--articles)
    - [Events](#events)
    - [Helping those in the community](#helping-those-in-the-community)
    - [Pact champions](#pact-champions)
  - [Technical Info](#technical-info)
    - [Dependency Graph of Doom](#dependency-graph-of-doom)
  - [Measurement](#measurement)
  - [External pact repositories](#external-pact-repositories)

## Links

| Project              | Resources                                                                                                                                                                                                       |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Pact Broker          | [Github](https://github.com/pact-foundation/pact_broker) / [Github Actions](https://github.com/pact-foundation/pact_broker/actions) / [Docker](https://hub.docker.com/repository/docker/pactfoundation/pact-broker) |
| Pact CLI             | [Github](https://github.com/pact-foundation/pact-ruby-cli) / [Docker](https://hub.docker.com/repository/docker/pactfoundation/pact-cli)                                                                           |
| Pact Ruby Standalone | [Releases](https://github.com/pact-foundation/pact-ruby-standalone/releases) / [E2E example](https://github.com/pact-foundation/pact-ruby-standalone-e2e-example)                                                 |
| Pact JS              | [Github](https://github.com/pact-foundation/pact-js) / [Message docs](https://github.com/pact-foundation/pact-js#asynchronous-api-testing)                                                                        |
| Roadmap              | [Canny](https://pact.canny.io)                                                                                                                                                                                  |
| Docs                 | [Homepage](https://docs.pact.io)                                                                                                                                                                                |

## Get involved!

The Pact ecosystem is vast, I will be sharing some posts over the upcoming months, showing the size of the estate, and looking to gain insight from you, the community, as to how we can reduce the signal-to-noise and help reduce the cognitive load required to navigate the path the Pact Nirvana in your own organisation.

There are a multitude of ways, and you don't need to be a code wizard to start:

### 📙 Docs

Our documentation is the primary way to communicate to our users, you can help out with small changes like a typo, help rewrite larger pieces, or add new content. Think of it as a open source contract testing wiki, and you are all the curators.

### 🚀 Code

We have implementations across multiple languages, and not all of them are at feature parity. Sometimes you might need that feature, or you've found a bug. Every pact-foundation repository is open-source, and contains a contributing guide to help you get started. Maybe you are building your own Pact tooling, let us know, we would love to shout about it.

### Roadmap / Feature Requests

The Pact roadmap is available on Canny, where you can see some of the teams current and upcoming priorities in the OSS space. You can request new features, or browse the list and vote/comment on ones you would love to see. See one that particularly resonates? You could help work on it, reach out via Slack and we can help guide you through your contribution.

### Recipes

The community use our tools in a variety of different ways, and solve various challenges that others could benefit. Got something to share? Why not add a new recipe to the site?

### Workshops

We created a number of workshops, across several languages. Is there a language implementation not covered in the workshop? Maybe you've created or seen some amazing workshops out there in the wild? Add it to the list, or if you are the author, you can discuss bringing your workshop under the Pact-foundation, if you feel all Pact users could benefit

### Blogs, Videos & Articles

Articles about contract testing are appearing left, right and centre, I can't keep up. Make sure our reading list doesn't get dry, by adding your favourite content to the list

### Events

Meetups, in person, it feels like a distant memory, but as the doors start opening again, and dinner is provided, people are beginning to flock outdoors. Have you got a meetup or event planned? Already had one and recorded it? You can add them to the list, and let us and the community know about it on Slack.

### Helping those in the community

We know many of you in the community love sharing your contract testing knowledge with others, you can see the various places our users land for help, sometimes in GitHub issues, Stack overflow, or Slack. You are welcome to help them out whether you are new to Pact, or a seasoned pro, all questions, opinions and thoughts are welcome.

### Pact champions

Are you like our co-founder Beth Skurrie, who decided that Pact idea was the best thing since sliced bread, and she hasn't stopped yacking on about it since. Want to share your knowledge, and build your social profile in the world of tech with a global platform? Please get in touch, we want to support the amazing work you do!

## Technical Info

This section contains technical information about the pact source code and builds. Go to [pact.io][pactio] for user documentation

### Dependency Graph of Doom

> To minimise the amount of code maintenance, many pact implementations depend on some shared libraries. When the shared libraries are updated, it is important to update the packages that use them.

This graph shows the dependency relationships to assist in updating the libraries.

See the [Dependency Graph of Doom](./dependency_graph.md)

## Measurement

We want to track our GitHub repo usage, health, and cross-correlate with Slack, so we can reach the right people quicker, and ultimately save leaving you hanging.

I'd also love to build some tooling myself, but why reinvent the wheel, there is already plenty of stuff to do in Pact land!

So why Orbit? I'll let them explain in their own [words](https://orbit.love/blog/why-orbit-is-better-than-funnel-for-developer-relations)

![image](https://user-images.githubusercontent.com/19932401/170391529-39cbfa31-8964-475c-b5f6-31c8c806cf78.png)

## External pact repositories

The fact that Pact works across so many different languages these day is due to the contribution of many different people and companies. To acknowledge the support that many companies have made to Pact's open source code, some repositories still live under the github organisation of their original author. They are listed below.

- [Pact Broker Docker](https://github.com/DiUS/pact_broker-docker)
- [Pact Consumer Swift](https://github.com/DiUS/pact-consumer-swift)
- [Pact provider verifier docker](https://github.com/DiUS/pact-provider-verifier-docker)
- [Pact Swagger mock validator](https://bitbucket.org/atlassian/swagger-mock-validator)
- [Scala Pact](https://github.com/ITV/scala-pact)

[pactio]: http://pact.io
