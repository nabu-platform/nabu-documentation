# Data model

A data model (or schema) defines the structure of the content that is expected.

If we look at our company XML from before:

```xml
<company>
	<vat>BE1234567890</vat>
	<name>Example Company</name>
</company>
```

We can see that there are a number of things that are important:

- the root tag is called "company"
- it contains two child tags, one called "vat" which contains a VAT number, the other called "name" which appears to be free text

Simply giving those bullet points to a computer fares about as well as our earlier unstructured request for data, we must capture this definition in a way that the computer can understand it.
Over the years many systems have evolved that describe data, each have their own advantages and disadvantages. Some examples are XML Schema, JSON Schema, UML,...

For example describing this data model in XML Schema could look like this:

```xml
<schema xmlns="http://www.w3.org/2001/XMLSchema" xmlns:tns="example.com" targetNamespace="example.com">
    <complexType name="company">
        <sequence>
            <element name="vat" nillable="true" type="string"/>
            <element name="name" nillable="true" type="string"/>
        </sequence>
    </complexType>
    <element name="company" nillable="true" type="tns:company"/>
</schema>
```

In JSON Schema we might get:

```json
{
	"type": "object", 
	"required": [
		"vat", 
		"name"
	], 
	"properties": {
		"vat": {
			"type": "string"
		}, 
		"name": {
			"type": "string"
		}
	}
}
```

There are of course tons of tools available that will give you a graphical environment to model your data rather than expecting you to type these schema's by hand.

As mentioned before, each modeling format has its own advantages and disadvantages but getting into the nitty gritty details of these formats would require several books worth of material and is outside the scope of this article.
Instead we'll be looking at some important concepts and how Nabu handles data modeling.

## Different Types

In data modelling we generally distinguish between two different data types:

- :simple: an atomic piece of information like a number, a date, a piece of text,...
- :complex: a composition of simple types and/or other complex types

In our company example, the vat and name fields would be simple types while the company wrapper element is a complex type.

Because a lot of the most commonly used data formats are string-based (check out the data formatting section for more information), all of the simple types are at the very least strings.
However, some of these strings have further restrictions imposed on them because they are conveying something more specific than free text, for example a date.

In human writing we already agree on specific formats to convey data to avoid confusion. Although geographic differences in formats can still lead to misunderstandings. For example in the US, most people will write month/day/year while in Europe we write day/month/year. Of course to avoid confusion things like this are standardized in data models.

```xml
<company xmlns="example.com">
	<vat>BE1234567890</vat>
	<name>Example Company</name>
	<founded>1984-07-13T10:00:00</founded>
	<amountOfEmployees>37</amountOfEmployees>
</company>
```

If we look at the XML Schema for this example:

```xml
<schema xmlns="http://www.w3.org/2001/XMLSchema" xmlns:tns="example.com" targetNamespace="example.com" elementFormDefault="qualified">
    <complexType name="company">
        <sequence>
            <element name="vat" nillable="true">
                <simpleType>
                    <restriction base="string">
                        <pattern value="BE[0-9]{10}"/>
                    </restriction>
                </simpleType>
            </element>
            <element name="name" nillable="true" type="string"/>
            <element name="founded" nillable="true" type="dateTime"/>
            <element minOccurs="0" name="amountOfEmployees" nillable="true" type="long"/>
        </sequence>
    </complexType>
    <element name="company" nillable="true" type="tns:company"/>
</schema>
```

A few additions jump out from our previous example:

- we have different data types (dateTime and long)
- we have added a restriction to our VAT, in this case a pattern which is a regex that the string must comply with in order to be valid. Regular expression (regex) can be a powerful tool in shaping your requirements but it is a quite extensive topic that is outside the scope of this article.
- we have added a minOccurs to our amountOfEmployees element which means it the minimum occurence in the actual data set is 0, in other words: it's optional

**Note**: XML (and the related XML Schema's to describe them) can become very complex once you start working with namespaces. Because a lot of other data modeling notations and data formats do not support namespaces at all and XML has steadily been losing ground to other formats, this article will not dig into these specific details.

## Specificity

By adding properties to our elements, we can further specify exactly what we want. For example we want a number instead of a string. And maybe that number should never be negative.

As a general rule: the more specific you +can+ be, the better. You can always loosen restrictions later on if you need to support other usecases, conversely once you have agreed upon a specific data model, it is very hard to make it +more+ strict after the fact because that would require all parties involved to adhere to the more strict rules. 

For example: if a field is mandatory at the outset, that means all third parties sending you data will fill it in. If you make it optional later on, the data being sent by the third parties is still valid. This is what we call a ``backwards compatible`` change to a data model. However if that field was optional at the outset, some third parties might not send it along in some usecases. Suddenly making it required means that data will no longer be valid, this is a non-backwards compatible change.

When evolving data models, we strive to make backwards compatible changes only if at all possible. You can easily validate if it is backwards compatible by asking yourself: is all the data that is generated against the old model valid against the new model?







simple types vs complex types

data formats (date, uri,...)

properties (required, pattern,...)

validation


schematron for rule based validation

because the structure of data is also in itself data, we generally use the same formats to store them
for example XML schema is described in XML, UML uses XML as well, json schema is described in json

advanced modeling: extension, restriction (1-* inheritance vs *-*)




## Transformation

In the above example, if you still really need to support a dutch version of the format because a third party insists on it, you can agree with that third party on a different model and +transform+ the data before you send it to your business logic. But in such a scenario, the business logic remains unchanged and rock solid while you simply have a second, predefined, validatable model agreement with another party.

Data transformation can be a straight mapping where the same fields are present but simply with another name or a different, convertible value or can become more complex by enriching, filtering,... the data.
