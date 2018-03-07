---
layout: default
permalink: /docs/getting-started/create-events-and-commands/
title: Events and commands
---

# Create Events and commands

Events are the core of any event sourced system. They are the payload,
the message, they allow our system to communicate in a meaningful way.
Events and commands are very simple objects. They are should be modeled
as "read-only" objects. This means they have to be  instantiated with
all the data they need and _only_ expose the data. They also have but
one technical requirements:

> All events must implement the `EventSauce\EventSourcing\Event` interface.

This interface requires you to create implement **3** public functions:

> 1. `timeOfRecording(): PointInTime`
> 2. `toPayload(): array`
> 3. `fromPayload(array $payload, PointInTime $timeOfRecording): Event`

## Time of recording 

Every event has a `EventSauce\EventSourcing\Time\PointInTime` object. This
is one of the few constraints that EventSauce places upon its
users, and for very good reasons. Almost every event sourcing project
eventually comes to a point where the timing of events (and/or commands)
becomes significant. Having this information from the start is a small
investment that always pays itself back.

## To and from payload

The `toPayload` and (static) `fromPayload` methods are used in the serialization
process. The `toPayload` method is expected to return an array that's serializable as JSON.
The `fromPayload` method is exctected to create an instance from a deserialized JSON array.

To illustrate:

```php
$event1 = new MyEvent($clock->pointInTime());
$event2 = MyEvent::fromPayload(
    $event1->timeOfRecording(),
    $event1->toPayload()
);

assert($event1 == $event2);
```

## Defining events (and commands)

Defining events and commands can be done in 2 ways.

* Defining them in YAML (code generation).
* Creating classes by pressing keys on your keyboard.


## Manually creating classes.

EventSauce provides interfaces for events and commands. You can create implementations of this. Here are minimal 
examples.

### Event

```php
<?php

use EventSauce\EventSourcing\Event;
use EventSauce\EventSourcing\PointInTime;

class SomeEvent implements Event
{
    private $timeOfRecording;

    public function __construct(PointInTime $timeOfRecording)
    {
        $this->timeOfRecording = $timeOfRecording;
    }

    public function timeOfRecording(): PointInTime
    {
        return $this->timeOfRecording;
    }

    public function toPayload(): array
    {
        return [];
    }

    public static function fromPayload(array $payload, PointInTime $timeOfRecording): Event
    {
        return new SomeEvent($timeOfRecording);
    }
}
```

As you can see in the examples above, there are a handful of required methods.  The _from_ and _to_ payload methods are
used in the serialization process. This ensures the events can be properly stored. Values returned in the `toPayload`
method should be `json_encode`-able. Additional required properties for an event should be injected into the constructor
and properly formatted in the payload methods.

## Defining commands and events using YAML.

Commands and events aren't very special, they're often just glorified arrays with accessors. A common name for these kind
of objects is DTO (Data Transfer Object). Because of their simplicity it's possible to use code generation:

```php
<?php

use EventSauce\EventSourcing\CodeGeneration\CodeDumper;
use EventSauce\EventSourcing\CodeGeneration\YamlDefinitionLoader;

$loader = new YamlDefinitionLoader();
$dumper = new CodeDumper();
$phpCode = $dumper->dump($loader->load('path/to/definition.yml'));
file_put_contents($destination, $phpCode);
```

Here's an example YAML file containing some command and event definitions.

```yaml
namespace: Acme\BusinessProcess 
commands:
    SubscribeToMailingList:
        fields:
            username:
                type: string
                example: example-user
            mailingList:
                type: string
                example: list-name
    UnsubscribeFromMailingList:
        fields:
            username:
                type: string
                example: example-user
            mailingList:
                type: string
                example: list-name
            reason:
                type: string
                example: no-longer-interested
events:
    UserSubscribedFromMailingList:
            fields:
                username:
                    type: string
                    example: example-user
                mailingList:
                    type: string
                    example: list-name
        UserUnsubscribedFromMailingList:
            fields:
                username:
                    type: string
                    example: example-user
                mailingList:
                    type: string
                    example: list-name
                reason:
                    type: string
                    example: no-longer-interested
```

Which compiles to the following PHP file:
 
 ```php
 <?php
 
 namespace Acme\BusinessProcess;
 
 use EventSauce\EventSourcing\Event;
 use EventSauce\EventSourcing\PointInTime;
 
 final class UserSubscribedFromMailingList implements Event
 {
     /**
      * @var string
      */
     private $username;
 
     /**
      * @var string
      */
     private $mailingList;
 
     /**
      * @var PointInTime
      */
     private $timeOfRecording;
 
     public function __construct(
         PointInTime $timeOfRecording,
         string $username,
         string $mailingList
     ) {
         $this->timeOfRecording = $timeOfRecording;
         $this->username = $username;
         $this->mailingList = $mailingList;
     }
 
     public function username(): string
     {
         return $this->username;
     }
 
     public function mailingList(): string
     {
         return $this->mailingList;
     }
     
     public function timeOfRecording(): PointInTime
     {
         return $this->timeOfRecording;
     }
 
     public static function fromPayload(
         array $payload,
         PointInTime $timeOfRecording): Event
     {
         return new UserSubscribedFromMailingList(
             $timeOfRecording,
             (string) $payload['username'],
             (string) $payload['mailingList']
         );
     }
 
     public function toPayload(): array
     {
         return [
             'username' => (string) $this->username,
             'mailingList' => (string) $this->mailingList
         ];
     }
 
     public function withUsername(string $username): UserSubscribedFromMailingList
     {
         $this->username = $username;
         
         return $this;
     }
 
     public function withMailingList(string $mailingList): UserSubscribedFromMailingList
     {
         $this->mailingList = $mailingList;
         
         return $this;
     }
 
     public static function with(PointInTime $timeOfRecording): UserSubscribedFromMailingList
     {
         return new UserSubscribedFromMailingList(
             $timeOfRecording,
             (string) 'example-user',
             (string) 'list-name'
         );
     }
 
 }
 
 
 final class SubscribeToMailingList
 {
     /**
      * @var PointInTime
      */
     private $timeOfRequest;
 
     /**
      * @var string
      */
     private $username;
 
     /**
      * @var string
      */
     private $mailingList;
 
     public function __construct(
         PointInTime $timeOfRequest,
         string $username,
         string $mailingList
     ) {
         $this->timeOfRequest = $timeOfRequest;
         $this->username = $username;
         $this->mailingList = $mailingList;
     }
 
     public function timeOfRequest(): PointInTime
     {
         return $this->timeOfRequest;
     }
 
     public function username(): string
     {
         return $this->username;
     }
 
     public function mailingList(): string
     {
         return $this->mailingList;
     }
 
 }
 
 final class UnsubscribeFromMailingList
 {
     /**
      * @var PointInTime
      */
     private $timeOfRequest;
 
     /**
      * @var string
      */
     private $username;
 
     /**
      * @var string
      */
     private $mailingList;
 
     /**
      * @var string
      */
     private $reason;
 
     public function __construct(
         PointInTime $timeOfRequest,
         string $username,
         string $mailingList,
         string $reason
     ) {
         $this->timeOfRequest = $timeOfRequest;
         $this->username = $username;
         $this->mailingList = $mailingList;
         $this->reason = $reason;
     }
 
     public function timeOfRequest(): PointInTime
     {
         return $this->timeOfRequest;
     }
 
     public function username(): string
     {
         return $this->username;
     }
 
     public function mailingList(): string
     {
         return $this->mailingList;
     }
 
     public function reason(): string
     {
         return $this->reason;
     }
 }
 ```