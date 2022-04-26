@layer data
@title What is data?
@core true

# What is data

Data is the most important commodity of the 21st century. Every company revolves around the data it has. Whether those are trade secrets to design some kind of proprietary system, data about the customers or their behavior, data about the market it operates it or that can affect sales, the list goes on and on.
Software applications revolve around data. Storing data, transforming data, exchanging data, interacting with data, it is at the core of every digital solution in one way or another.

Before we dig into specifics, let's define some basics.

## Noise vs data

At the most basic level, data differs from noise because it contains something of value. 

When you are talking, you are transmitting data via airwaves. A car muffler is using the same medium but does not convey anything of value, we label this as noise. One man's noise +can+ be another man's data but those particular details go beyond the scope of this article.

## Structured vs unstructured

Once we have established that data contains something of value, the question becomes: how can we extract that value? In the case of speech, our ears and brain transform the airwaves into electric signals that can lead to understanding and decision making.

However, from a computer perspective we would label this type of data as "unstructured". It is an analogue waveform with infinite precision. Even if we sample it into a digital signal with discrete precision, we would still consider it unstructured because it lacks syntax: there is no formal, documented way to parse this data into something the computer understands. 

This is typical of data derived from analogue sources like video, images, audio,... or things like language which have evolved in ways that make rule-based analysis nearly impossible because there are more exceptions to rules than actual rules in any given language.

Suppose you want to get all the contract details for a particular contract number, the unstructured way to ask a computer could be: 

```
Hey uh...computer...can you give me the latest info on contract A37..no wait, A389001?
```

Did you read that as "A-3-8-9-0-0-1" or did you arbitrarily group numbers like "A-38-90-01" or even "A-389-0-0-1"?

A structured way to ask the same thing could be using one of the many well documented formats like XML:

```xml
<contractNumber>A389001</contractNumber>
```

The fishtags and the trailing slash are all part of the syntax, making it trivial for a computer to get the two key aspects from this message: the word "contractNumber" and the actual value of that field: A389001.

Of course what is easy for a computer to parse and understand it not necessarily easy for a human to create with the precision required by a computer. That is at the core of application development: building ways for computers and humans to interact.

## Artificial Intelligence

At this point it is important to highlight the significance of AI, especially in the last few years. Classic AI is, much like our brains, incredibly good at pattern recognition. While in single instances it generally does not perform as well as our brain, it more than makes up for it in volume and complexity. You can categorize a single image better than a computer, but when it comes to categorizing millions of images, the computer will always win.

In the same vein, when figuring out the pattern in a limited set of numbers, our brains will usually win, but given a big enough data set our brains can no longer cope while the computer actually thrives on bigger datasets.

This pattern recognition power can be used to extract the value from seemingly unstructured data. For example given enough language samples, the computer can figure out the language patterns allowing it to deduce which words are significant and how. Given enough examples of cat pictures, it can quickly find cats in millions of images.
Even if you have a large amount amount of already structured data, AI can be used to extrapolate new insights you were not aware off by noticing patterns hidden in the dataset.

However, this process is so CPU-intensive that we have switched to dedicated hardware rather than doing this on general purpose computers. Even with all that processing power, it can result in incorrect data, which means you should only go this route if there is no other way to extract the value from the data.

Data is only valuable once we get the value out, structured data simply makes it a lot easier to extract that value and use it in other processes. That means AI analysis of unstructured data is mostly valuable to get to some form of structured data that then feeds into other systems. This means it often sits at the periphery of structured data systems, allowing them to work with data that would otherwise be worthless to them.

## Two dimensions

When we talk about structured data, there are actually two dimensions to that structure.
The one we highlighted above is the ``data format``, it is the lexical conformity that the data must adhere to so other systems can easily parse and understand the data. It defines the seperators or patterns that a computer can look for to determine where one piece of data stops and another begins. Examples of this are JSON, CSV, XML, EDI,...

The other dimension to structured data is defining what it is actually trying to convey. While both these examples use the same syntax (XML), they talk about entirely different topics:

```xml
<book>
	<isbn>123</isbn>
	<author>John Smith</author>
</book>
```

```xml
<company>
	<vat>BE123</vat>
	<name>Example Company</name>
</company>
```

While the data format is relevant to +parsing+ the data into something the computer understands, the ``data model`` details what it actually means and is relevant to +using+ the data in business logic. While you can do some neat technical tricks on data where the data model is unknown, there is very little useful business logic that can be built on a data with an unknown model because at some point, you need to know how the data you have relates to the problem you are trying to solve.

