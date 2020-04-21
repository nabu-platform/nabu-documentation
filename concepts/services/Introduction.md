@tags service

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

When everything is expressed as a service, your task as application creator is mainly to orchestrate how data flows from one to the other. This orchestration can be straight forward linking of services or it can contain more complicated logic. You might for example want to take conditional steps that only occur in certain circumstances.

By orchestrating, you create a new service which can be used by other orchestrations and so on and so forth. Combine this with the vast amount of services that Nabu offers in its ready-to-go modules and you can build applications really fast by literally connecting the dots.

## Example

Let's say we want to create a simple service, it has this ``interface``:

[:.resources/basic/orchestration-reorderer-interface.png]

And the ``intent`` of the service is to take the input, consider it a comma separated list and reverse the order of said list. Here is how it might do that through simple orchestration:

[^.resources/basic/orchestration-reorder-implementation.mp4]

Now that we have created a very simple orchestration service, we can also run it using the Nabu IDE:

[^.resources/basic/orchestration-reorder-execute.mp4]

This visual orchestration is accomplished by using the blox service implementation. In a typical application, 80% of your backend will consist of creating such orchestration logic. More complicated logic is possible by creating conditional steps, looping over data sets,...

## Design Time Guarantees

Nabu will help you at every step when creating these orchestrations, it will for example validate that the data you are passing around is of a format that can work at runtime. For example if you have a date and give it to a service that expects a boolean, what would you logically conclude as the expected behavior? In a lot of cases it +is+ clear and Nabu will allow you to draw the line, in case of doubt however, Nabu will not allow you to draw the line, preventing problems later on when the application is running.

Getting this feedback while you are creating the service (as opposed to when you are running it), is called ``design time guarantees``.

