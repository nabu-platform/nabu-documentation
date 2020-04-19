# Services

When it comes to building [backends|$/explainers/Backend and Frontend.md], Nabu is a service oriented platform. This means the service concept is used throughout the platform in a wide variety of contexts. Once you grasp what a service is, you understand a significant chunk of what the platform offers for building backend solutions.

## Basic

A service has three basic concepts:

- :input: you give it data. This can be numbers because you are doing calculations or strings, or database tables, excel files,... 
- :logic: it performs some kind of computation on the input you gave it. For example it might add numbers together.
- :output: at the end of the computational logic, the service returns some answer. For example the total sum of the numbers you gave it.

There are many types of services but they all share that same paradigm. The only thing that differs is _how_ the logic works. 
Some examples of logic are: 

- query a database
- retrieve a configuration file
- talk to a remote server
- follow a visual execution plan you set up

## Interface

The combination of ``input`` and ``output`` of a service is called the ``interface``. The interface explains in detail which data the service expects you to supply and which data it will return.

[:.resources/basic/interface.png]

We can see an input on the left which defines two fields:

- firstNumber
- secondNumber

Both are numeric fields which we can deduce from the icon. Both are required, as they lack the green marker that would indiciate that they are optional, for example the same service but with optional inputs:

[:.resources/basic/interface-optional.png]

Both fields only expect a single value, lists are visually marked by a stacking of the icon:

[:.resources/basic/interface-list.png]

Apart from these visual cues for the most important properties of a field, Nabu has a ton of additional restrictions you can add. For example maybe you want only positive numbers to be passed in? Or all numbers must be below 1000?

There is a lot more to data types, you can read up on them [here|$/concepts/data/Introduction.md].

## Contract

The combination of ``interface`` and the ``intent`` of the logic we call the ``contract``. It is an agreement between the creator of the service and those who use it. +How+ the service performs the logic within, is +not+ part of the contract and rather what we call an ``implementation detail``. Only the creator of a service should be concerned with the implementation, not the person who uses the service.

That means if you understand the input, output and intent, you can use the service without the need to fully grasp how the details work.

## Running a Service

A service, much like any software, is only useful when you run it. Services can be triggered by all kinds of circumstances, for example:

- :web application: a user clicks on a button in the browser, that might trigger a service in the backend.
- :scheduler: you may want to run a certain service at regular intervals
- :API: you might expose an endpoint so other systems can trigger logic in your application

You can also manually run a service using the Nabu IDE:

[^.resources/basic/service-run.mp4]

# Orchestration


