# Masstransit SAGA implementation

## Problem

In Mageproject there may be changes on Herd Group by HerdManagement Service and this change must be Applied to Other services too. here we fire event of change/Update to GlobalAnalytics service from HerdManagement service. 

## Packages

> PM> NuGet\Install-Package MassTransit

## Implementation

The Saga State Machine Generally devides into 3 parts of :<br/>

1. State Machine Instance
2. State
3. Event
4. Behaviour

### Satate Implementation
State is the defination of state machine data model that holds require information to create and operate a state machin instance. It includes CorrelatioId to distinguish the transactions and the current State which indicates the satete at the moment. the events and behaviourmay change this state at a moment. the other properties carry information about the job to be done.<br/>

```
using MassTransit;

namespace HerdManagement.BusComponents.SAGAStateMachine.State
{
    public class EditGroupState : SagaStateMachineInstance, ISagaVersion
    {
        public Guid CorrelationId { get; set; }
        public int Version { get; set; }
        public string GroupId { get; set; }
        public string GroupTitle { get; set; }
        public string GroupDesc { get; set; }
        public string CurrentState { get; set; }

    }
}

```

### State Machine Instance initiation Implementation
State Machine Instance is the operation center of certained transaction which orchestrate the events and update the defined states based on behaviour. here is the begining setup as follow.

```
using HerdManagement.BusComponents.SAGAStateMachine.State;
using MassTransit;
using MassTransit.Configuration;


namespace HerdManagement.BusComponents.SAGAStateMachine
{
    public class EditGroupStateMachine : MassTransitStateMachine<EditGroupState>
    {

        public EditGroupStateMachine()
        {
            // current default state
            InstanceState(x => x.CurrentState);


        }

        //defined states
        public MassTransit.State Completed { get; set; }
        public MassTransit.State Failed { get; set; }

    }
}
```

### Events and Behaviour Implementation
An event is something that happened which may result in a state change. After setting up State and State machine instance we continue implementing transaction scenario by defining Events and after Events update changes based on the state different behaviour may take place which is nothing but the Evenets fired as consiquence of the previous state changes if there any.<br/>
Speaking of Update Herd Groups we have scenario as <br/>

- (Event) Update Group in the Herd Management service => (State) the state is changed into **HerdManagementUpdateCompleted**.
- (behaviour) after state is changed to  **HerdManagementUpdateCompleted** then (Event) Update Group in GlobalAnalytics Service => (State) The State is Changed into **GlobalAnalyticsUpdateCompleted**.

![Activitydiagram2](https://user-images.githubusercontent.com/105317212/209435946-4c5da27a-bae0-4cc2-98ea-5582cb1eb219.png)

we have to consider that exceptions or fault may be happen during the Transaction that brings other scenario and behaviours to our conserns such as recover and rollback operations to handdle the faults.<br/>

#### Slip Routing
Masstransit Slip Routing is responsible for Chained event of a transaction that capable us manage and orchestrate the transaction or part of chained events as well as handle fault happening during transaction as it could fire compensate event to do that.

<br/>

### Implementation Steps

#### 1. Define Event Message Contract<br/>
Event contracts hold the properties required to handle the operation. In this case these properties are passed to the consumer to do the job.<br/>

1.1. HerdManagement Service Group Update Event Contract.

```
namespace EventBus.Contracts.HerdManagement.EventMessage
{
    public interface IUpdateHerdGroup
    {
        public Guid CorrelationId { get; set; }
        public string GroupId { get; set; }
        public string GroupTitle { get; set; }
        public string GroupDescription { get; set; }
    }
}

```

2. Define and Implemet Slip Routing

As mentioned Masstransit Slip Routing is responsible for orchestrate the chained events that here are defined in terms of Activities.
2.1 Define Herd Management Group Update Activities


