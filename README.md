<h1 align="center">Pablo Mickael Quevedo</h1>

<p align="center">
  <strong>.NET Architect · Distributed Systems · Payments &amp; Industrial IoT</strong>
</p>

<p align="center">
  Architecture and engineering for the systems behind a 14-company group and <strong>R$ 3B</strong> in revenue.<br />
  Former Tech Lead of the cross-cutting systems team at Grupo Herval — today responsible for the<br />
  highest-complexity projects across a portfolio of roughly <strong>150 repositories</strong>.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/.NET-10-512BD4?style=flat-square&logo=dotnet&logoColor=white" alt=".NET 10" />
  <img src="https://img.shields.io/badge/C%23-14-239120?style=flat-square&logo=csharp&logoColor=white" alt="C# 14" />
  <img src="https://img.shields.io/badge/Clean%20Architecture-DDD%20%C2%B7%20CQRS-007ec6?style=flat-square" alt="Clean Architecture, DDD, CQRS" />
  <img src="https://img.shields.io/badge/Kubernetes-Docker-326CE5?style=flat-square&logo=kubernetes&logoColor=white" alt="Kubernetes and Docker" />
  <img src="https://img.shields.io/badge/PostgreSQL%20%C2%B7%20Oracle%20%C2%B7%20MongoDB-4169E1?style=flat-square&logo=postgresql&logoColor=white" alt="PostgreSQL, Oracle, MongoDB" />
</p>

---

## Open Source

A set of production libraries extracted from systems that run in production, not from tutorials.
Each one solves a problem I kept re-solving by hand across repositories.

| Package | What it solves | Version | Downloads |
|---|---|---|---|
| [**PMQ.Domain**](https://github.com/PabloMickaelQuevedo/PMQ.Domain) | DDD building blocks — entities with identity-based equality, aggregate roots, value objects and domain events | [![NuGet](https://img.shields.io/nuget/v/PMQ.Domain?style=flat-square&color=004880&logo=nuget&label=)](https://www.nuget.org/packages/PMQ.Domain) | ![Downloads](https://img.shields.io/nuget/dt/PMQ.Domain?style=flat-square&color=555&label=) |
| [**PMQ.Mediator**](https://github.com/PabloMickaelQuevedo/PMQ.Mediator) | CQRS mediator with pipeline behaviors, streaming and FluentValidation integration | [![NuGet](https://img.shields.io/nuget/v/PMQ.Mediator?style=flat-square&color=004880&logo=nuget&label=)](https://www.nuget.org/packages/PMQ.Mediator) | ![Downloads](https://img.shields.io/nuget/dt/PMQ.Mediator?style=flat-square&color=555&label=) |
| [**PMQ.Notifications**](https://github.com/PabloMickaelQuevedo/PMQ.Notifications) | Notification pattern — business rules as data instead of exceptions | [![NuGet](https://img.shields.io/nuget/v/PMQ.Notifications?style=flat-square&color=004880&logo=nuget&label=)](https://www.nuget.org/packages/PMQ.Notifications) | ![Downloads](https://img.shields.io/nuget/dt/PMQ.Notifications?style=flat-square&color=555&label=) |
| [**PMQ.ErrorHandling**](https://github.com/PabloMickaelQuevedo/PMQ.ErrorHandling) | Centralized ASP.NET Core error handling with RFC 9457 responses | [![NuGet](https://img.shields.io/nuget/v/PMQ.ErrorHandling?style=flat-square&color=004880&logo=nuget&label=)](https://www.nuget.org/packages/PMQ.ErrorHandling) | ![Downloads](https://img.shields.io/nuget/dt/PMQ.ErrorHandling?style=flat-square&color=555&label=) |
| [**PMQ.Identity**](https://github.com/PabloMickaelQuevedo/PMQ.Identity) | Provider-agnostic authentication — external OIDC or self-issued JWT | [![NuGet](https://img.shields.io/nuget/v/PMQ.Identity?style=flat-square&color=004880&logo=nuget&label=)](https://www.nuget.org/packages/PMQ.Identity) | ![Downloads](https://img.shields.io/nuget/dt/PMQ.Identity?style=flat-square&color=555&label=) |

### How they compose

They are not five unrelated utilities — they snap together into one request lifecycle:

```mermaid
flowchart TB
    REQ([HTTP request]) --> AUTH

    subgraph API ["&nbsp;API&nbsp;"]
        AUTH["PMQ.Identity<br/><i>OIDC · JWT</i>"] --> CTRL[Controller]
        ERR["PMQ.ErrorHandling<br/><i>RFC 9457 · 400 · 422</i>"]
    end

    CTRL --> MED

    subgraph APP ["&nbsp;Application&nbsp;"]
        MED["PMQ.Mediator<br/><i>command · query · pipeline</i>"] --> VAL["Validator<br/><i>transport contract</i>"]
    end

    VAL --> AGG

    subgraph DOMAIN ["&nbsp;Domain&nbsp;"]
        AGG["PMQ.Domain<br/><i>Entity · ValueObject · events</i>"] --> NOTIF["PMQ.Notifications<br/><i>broken rules as data</i>"]
    end

    AGG --> UOW

    subgraph INFRA ["&nbsp;Infrastructure&nbsp;"]
        UOW["Unit of Work<br/><i>commit, then publish</i>"] --> EVT[Domain event handlers]
    end

    NOTIF -.-> ERR
    ERR -.-> RES([HTTP response])
    UOW --> RES

    classDef pkg fill:#512BD4,stroke:#512BD4,color:#fff
    classDef plain fill:transparent,stroke:#888,color:#888
    class AUTH,ERR,MED,AGG,NOTIF pkg
    class CTRL,VAL,UOW,EVT plain
```

Domain events are published **after** the commit — a handler must never observe a fact the
transaction ended up rolling back.

### The design decision behind them

A violated business rule is an **expected outcome**, not exceptional control flow. So entities
accumulate their failures instead of throwing — which costs less, and reports *every* problem at
once instead of only the first one:

```csharp
var order = Order.Create(request.Items);          // never throws

if (order.IsInvalid)
{
    notificationContext.AddFrom(order, NotificationType.BusinessRule);   // → HTTP 422
    return Guid.Empty;
}
```

Exceptions stay where they belong: programming errors.

<details>
<summary><strong>What the client gets back</strong></summary>

<br />

One request, every broken rule — instead of discovering them one deploy at a time:

```json
{
  "title": "A business rule validation failed.",
  "status": 422,
  "errors": [
    { "field": "Items", "message": "Item must be at most 200 characters." },
    { "field": "Items", "message": "Provide at least one item." }
  ],
  "traceId": "00-079d247677d55f7d8427fd421daec5f0-a773dd27f85d9bdb-01"
}
```

</details>

---

## What I Build

### Payments

The group's payment hub. Microservices around a central workflow API orchestrating acquiring,
antifraud, boleto, Pix, a decision rules engine and a historical transaction base for risk
analysis. Tokenization, authorization and capture, including recurring billing.

Revenue from e-commerce, the mobile app and **200+ stores** flows through it, plus the group's
own programs: Cartão Hoje, HojePay, iPlace Club and subscriptions.

### Industrial Monitoring

OEE and shop-floor monitoring platform. Sensors over **MQTT**, time series in **TimescaleDB**,
**Redis**, real time via **SignalR**, and a **PWA** front end with **IndexedDB** that keeps
operating with the network down.

The rules engine is configurable by the business itself: how to count production, what counts as
downtime, when to raise an alert.

### Integrations

**SAP** as ERP, with internal SDKs standardizing authentication, OData and sales orders.
**Apple GSX** and Trade In supporting technical assistance and buyback. **Zendesk** and
**Emarsys** for support and CRM. And the orders API, where e-commerce, app and stores converge
before the ERP and logistics.

---

## Agent-Oriented Engineering

I work in a model where implementation is carried out by agents and each task is driven
end to end — from technical definition to release — usually several in parallel.

> Implementation stopped being the slow part. Whoever defines the problem better comes out ahead.

---

## Stack

<p>
  <img src="https://skillicons.dev/icons?i=dotnet,cs,docker,kubernetes,postgres,mongodb,mysql,redis,react,angular,git,gitlab&theme=dark" alt="Stack icons" />
</p>

**Platform** · .NET 10 / C# · ASP.NET Core · REST APIs and microservices
**Architecture** · Clean Architecture · DDD · CQRS · Event-driven
**Data** · Oracle · PostgreSQL · MongoDB · MySQL · EF Core · Dapper · Dremio · TimescaleDB
**Infrastructure** · Docker · Kubernetes · GitLab CI/CD · Redis
**Real time** · MQTT · SignalR
**Front end** · React · Angular · PWA
**AI** · LLM integration in .NET

---

<p align="center">
  <img height="160" src="https://github-readme-stats.vercel.app/api?username=PabloMickaelQuevedo&show_icons=true&hide_border=true&theme=transparent&title_color=512BD4&icon_color=512BD4&count_private=true" alt="GitHub stats" />
  <img height="160" src="https://github-readme-stats.vercel.app/api/top-langs/?username=PabloMickaelQuevedo&layout=compact&hide_border=true&theme=transparent&title_color=512BD4&langs_count=6" alt="Top languages" />
</p>

---

## Connect

<p>
  <a href="https://www.linkedin.com/in/pablomickaelquevedo/" target="_blank">
    <img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white" alt="LinkedIn" />
  </a>
  <a href="mailto:pablomickaelquevedo@gmail.com">
    <img src="https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white" alt="Gmail" />
  </a>
</p>
