@layer data
@title What is data?
@core true

# What is data

## Noise vs data

At the most basic level, data differs from noise because it contains intentional value. If you are talking, you are transmitting data via airwaves. A car muffler is using the same medium but does not convey anything of value, we label this as noise. One man's noise can be another man's data but those particular details go beyond the scope of this article.

## Structured vs unstructured

Once we have established that data contains something of value, the question becomes: how can we extract that value to act upon the data? In the case of speech, our ears and brain transform the airwaves into electric signals that can lead to understanding and decision making.

However, from a computer perspective we would label this type of data as "unstructured". It is an analogue waveform with infinite precision. Even if we sample it into a digital signal with discrete precision, we would still consider it unstructured because it lacks syntax. There is no formal, documented way to parse this data into something the computer understands. 

This is typical of data derived from analogue sources like video, images, audio,... or things like language which has evolved in ways that make rule-based analysis nearly impossible because there are more exceptions to rules than actual rules in any given language.

Suppose you want to get all the contract details for a particular contract number, the unstructured way to ask a computer could be: 

```
Hey uh...computer...can you give me the latest info on contract A37..no wait, A389001?
```

Did you read that as "A-3-8-9-0-0-1" or did you arbitrarily group numbers like "A-38-90-01" or even "A-389-0-0-1"?

A structured way could be using one of the many well documented formats like XML:

```xml
<contractNumber>A389001</contractNumber>
```

The fishtags and the trailing slash are all part of the syntax, making it trivial for a computer to get the two key aspects from this message: the word "contractNumber" and the actual value of that field: A389001.

Of course what is easy for a computer to parse and understand it not necessarily easy for a human to create with the precision required by a computer. That is at the core application development: building ways for computers and humans to interact.

### Artificial Intelligence

At this point it is important to highlight the significance of AI, especially in the last few years. Classic AI is, much like our brains, incredibly good at pattern recognition. While in single instances it does not perform as well as our brain, it more than makes up for it in volume and complexity. You can categorize a single image better than a computer, but when it comes to categorizing millions of images, the computer will always win.

In the same vein, when figuring out the pattern in a limited set of numbers, our brains will usually win, but given a big enough data set our brains can no longer cope while the computer actually thrives on bigger datasets.

This pattern recognition power can be used to extract the value from seemingly unstructured data. Given enough language examples, the computer can figure out the language patterns allowing it to deduce which words are significant and how. Given enough examples of cat pictures, it can quickly find cats in millions of images.

Even if you have an incredible amount of structured data, AI can be used to extrapolate new insights you were not aware off by noticing patterns hidden in the dataset.

However, this process is so CPU-intensive that we have switched to dedicated hardware rather than doing this on general purpose computers. Even with all that processing power, it can result in incorrect data, which means you should only go this route if there is no other way to extract the value from the data.

Data is only valuable once we get the value out, structured data simply makes it a lot easier to extract that value and use it in other processes. That means AI analysis of unstructured data is mostly valuable to get to some form of structured data that then feeds into other systems which means it sits at the periphery of structured data systems, allowing them to work with data that would otherwise be worthless to them.

## Two dimensions

When we talk about structured data, there are actually two dimensions to that structure.
The one we highlighted above is the _syntax_, it is the lexical conformity that the data must adhere to so other systems can easily parse and understand the data. It defines the seperators or patterns that a computer can look for to determine where one piece of data stops and another begins. Examples of this are JSON, CSV, XML, EDI,...

The other dimension to data structure is defining what data is actually in there. While both these examples use the same syntax (XML), they talk about entirely different topics:

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

The syntax structure is relevant to _parsing_ the data, the content structure is relevant to _using_ the data. While you can do some neat technical tricks on data where the content structure is unknown, there is very little useful business logic that can be built on a data with an unknown content structure because at some point, you need to know how the data you have relates to the problem you are trying to solve.

## Defined content structure vs dynamic

Let's say we have an employee record, suppose we know the structure beforehand and _know_ that the record will _always_ contain a firstName and a lastName field:

```xml
<employee>
	<firstName>Johh</firstName>
	<lastName>Smith</lastName>
</employee>
```

This means -if we switch to pseudocode- we could state unequivocally:

```
fullName = employee.firstName + " " + employee.lastName
```

At the moment we are writing that pseudocode -so __at design time__- we _know_ that the fields will exist, we can rest assured that that piece of pseudocode will always work. There are no exceptions. Given the guarantee that those fields are always present with that exact name, we know that we will not run into situations where the someone writes "firstname" in all lowercase or "first_name" with underscores or "voornaam" in another language, or just leaves out the first name alltogether.

This means, not only do we have a _defined_ structure, but it is known at _design time_ and gives us guarantees as to what our business logic will do given that data. If faulty data somehow enters the system, it will never even reach our pseudocode because it will fail earlier validation checks to see that the data matches our expectations.
An alternative to this is that we don't specifically know or enforce the exact structure beforehand, but we have some vague ideas about what it _should_ contain. That severely complicates our business logic:

```
if (employee.firstName != null)
	firstName = employee.firstName
else if (employee.first_name != null)
	firstName = employee.first_name
else if (employee.voornaam != null)
	firstName = employee.voornaam

if (employee.lastName != null)
	lastName = employee.lastName
else if (employee.last_name != null)
	lastName = employee.last_name
else if (employee.achternaam != null)
	lastName = employee.achternaam

fullName = firstName + " " + lastName
```

We call this _defensive_ programming. We have to cover all eventualities in our code, rather than mandating a certain structure from the data. If at some point we notice that some french guy sent along employee.prenom to contain the firstName, we have to update our code, retest it, redeploy it and hope that we don't have a german user in the near future.
Our simple single line of business logic has exploded into a multiline monstrosity that still has a likelihood of failure in production. Defensive programming adds complexity, indicates frailty and makes the end result harder to understand.

## Specificity

The more you know about the structure of your data __while designing your business logic__, the simpler your end solution can be _and_ the more stable it will run in production. Knowing for instance that firstName is _always_ present (so a mandatory field) makes it easier than knowing it is _sometimes_ present (so an optional field) which would still require a minor bit of defensive pseudocode to deal with both situations.

Knowing that a Belgian VAT number _always_ has the same format -BE followed by 10 numbers- we can enforce this on the incoming data by configuring a pattern. That means, at the business logic level we don't have to account for someone sending "abc" as a VAT number. 
As a general rule: the more specific you _can_ be, the better. You can always loosen restrictions later on if you need to support other usecases, conversely once you have agreed upon a specific data format, it is very hard to make it _more_ strict after the fact because that would require all parties involved to adhere to the more strict rules. 

As an example: if a field is mandatory at the outset, that means all third parties sending you data will fill it in. If you make it optional later on, the data being sent by the third parties is still valid. This is what we call a ``backwards compatible`` change. However if that field was optional at the outset, some third parties might not send it along in some usecases. Suddenly making it required means that data will no longer be valid, this is a non-backwards compatible change.

In the above example, if you still really need to support a dutch version of the format because a third party insists on it, you can agree with that third party on a different structure and map the data before you send it to your business logic. But in such a scenario, the business logic remains unchanged and rock solid while you simply have a second, predefined, validatable structure agreement with another party.

## Conclusion

The Nabu Development Platform focuses heavily on what we call ``design time guarantees``, this means it has a heavy focus on data structure and will help you build solutions based on that knowledge to avoid bugs and provide predictable behavior in production with as few edge cases as possible.
The platform also has ways to deal with completely unstructured data in the form of byte streams and data that is only structured in the syntactic dimension without knowing the contents. The latter is rarely necessary when building business applications and discouraged because it can be very error prone.