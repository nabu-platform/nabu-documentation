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

These data models are described using a computer-readable ``data format`` and follow their own ``data model`` rules.

You don't really need to worry too much about the exact notation of these schema's as you'll rarely need to edit them by hand. There are tons of tools available that will give you a graphical environment to model your data, abstracting away from the notational details.

Before we dig into more complex data modeling concepts, let's start with some basics.

## Two Types

In data modelling we generally distinguish between two different data types:

- :simple type: an atomic piece of information like a number, a date, a piece of text,...
- :complex type: a composition of simple types and/or other complex types

In our company example, the vat and name fields would be simple types while the company wrapper element is a complex type.

Because a string is any random sequence of characters, almost all data we use in our day to day lives is at the very least a string.
However, some of these strings have further restrictions imposed on them because they are conveying something more specific than free text, for example a date or a number.

In human writing we already agree on specific formats to convey data to avoid confusion, although geographic differences in formats can still lead to misunderstandings. For example in the US, most people will write month/day/year while in Europe we write day/month/year. Of course to avoid confusion things like this are standardized in data models.

I've added a few more fields to the company example:

```xml
<company xmlns="example.com">
	<vat>BE1234567890</vat>
	<name>Example Company</name>
	<founded>1984-07-13T10:00:00</founded>
	<amountOfEmployees>37</amountOfEmployees>
</company>
```

If we then look at the XML Schema that describes the content of the company example we see:

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
- we have added a restriction to our VAT, in this case a pattern which is a regex that the string must comply with in order to be valid. Regular expression (regex) can be a powerful tool in shaping your requirements though the syntax can take some getting used to. In this case we state that we are expecting 10 characters between 0 and 9. So a number.
- we have added a minOccurs to our amountOfEmployees element which means it the minimum occurence in the actual data set is 0, in other words: it's optional

**Note**: XML (and the related XML Schema's to describe them) can become very complex (and powerful) once you start working with namespaces. They were added to the example to make it complete.

## Specificity

By adding properties to our elements, we can specify exactly what we want. For example we want a number instead of a string. And maybe that number should never be below 0 or over 100.

As a general rule: the more specific you +can+ be, the better. You can always loosen restrictions later on if you need to support other usecases, conversely once you have agreed upon a specific data model, it is very hard to make it +more+ strict after the fact because that would require all parties involved to adhere to the more strict rules. 

For example: if a field is mandatory at the outset, that means all third parties sending you data will fill it in. If you make it optional later on, the data being sent by the third parties is still valid. This is what we call a ``backwards compatible`` change to a data model. However if that field was optional at the outset, some third parties might not send it along in some usecases. Suddenly making it required means that data will no longer be valid, this is a non-backwards compatible change.

When evolving data models, we strive to make only backwards compatible changes. You can easily validate if it is backwards compatible by asking yourself: is all the data that is generated against the old model valid against the new model?

## Limits of specificity

Most validation is performed on a single simple type. The pattern you set on a string only applies to that field, not its sibling field.
Sometimes you want to create more complex validations, for instance: if the "founded" field is filled in, the "vat" field must also be filled in. These cross field validations are typically harder to define in a data model.

There are some specifications (like schematron) that try to capture rule-based validation that can include multiple fields, however they are rarely used.

More complex validations are often implemented in the "business logic". It's not always clear where validation rules end and business logic begins.

Take a numeric value. It could be easy to simply state that it should never be below zero. But what if the numeric value is a dollar amount for a payment and we actually want to check that the number is never larger than your current bank balance sitting in a database record? What if some customers are allowed to go into the red on their account, but only up to a specific amount? Some payments might need to be flagged for manual inspection, some might require additional verification from an account holder,... 

Two general guidelines that can help you decide if it belongs in validation or business logic:

- When the actual validation rule is dynamic or the consequences to failing a validation are dynamically decided, it falls squarely in the domain of business logic
- Validation rules should say something about the absolute nature of the value rather than how it will be used.

As an example of that second guideline: you could state, with a quite a bit of certainty, that a value containing "degrees Kelvin" should never be below 0. That's an intrinsic limitation on the frame of reference you are using. If however, the field only contains "degrees" and another field indicates a variable unit (e.g. celsius, fahrenheit,...), the reference frame is dynamic and it will become a lot harder to set viable limits on the degrees field.

## Purpose

Let's say we are making an application that deals with employee records:

```xml
<employee>
	<firstName>Johh</firstName>
	<lastName>Smith</lastName>
</employee>
```

Suppose we agree on a data model beforehand and +know+ that the record will +always+ contain a firstName and a lastName field. We can then state unequivocally (in pseudocode):

```
fullName = employee.firstName + " " + employee.lastName
```

While we are writing that pseudocode, ``at design time``, we know that the fields will exist, we can rest assured that that piece of pseudocode will always work. There are no exceptions. Given the guarantee that those fields are always present with that exact name, we know that we will not run into situations where someone writes "firstname" in all lowercase or "first_name" with underscores or "voornaam" in another language, or just leaves out the first name alltogether.

> An accurate data model makes the business logic predictable.

This means, not only do we have a +defined+ model, but it is known at ++design time++ and gives us guarantees as to what our business logic will do given that data. If faulty data somehow enters the system, it will never even reach our pseudocode because it will fail earlier validation checks to see that the data matches our expectations.

> An accurate data model let's us automatically validate that the data complies to our expectations.

An alternative to this is that we don't specifically know or enforce the exact structure beforehand, but we have some vague ideas about what it +should+ contain. That severely complicates our business logic:

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

We call this +defensive+ programming. We have to try and cover all eventualities in our code, rather than mandating a certain structure from the data. If at some point we notice that a French developer sent along employee.prenom to contain the firstName, we have to update our code, retest it, redeploy it and hope that we don't have a German developer in the near future.
Our simple single line of business logic has exploded into a multiline monstrosity that still has a likelihood of failure in production. Defensive programming adds complexity, indicates frailty and makes it harder to read the end result because the business logic is obscured by all the defensive constructs around it.

> Specificity helps reduce edge cases, requiring fewer defensive checks.

A good data model is the backbone of any application. It guides us while we are implementing the solution, it allows us to minimize runtime errors of unexpected edge cases and serves as living documentation on the current state of the system, allowing for easier expansions or alterations to the system in future iterations.

## The art of modeling

Creating a data model is about capturing the concepts that your application is built upon in two dimensions:

- which fields does a concept need to be fully expressed?
- how do these concepts relate to one another?

For example, imagine an application that needs to model companies, contracts and invoices.

In our first dimension we check the fields we need to represent these concepts. A company will likely need a VAT number, a name, maybe an address etc. A contract will like have a contract number, maybe a start date and an end date. An invoice will likely have an amount payable, an invoice number, a invoicing date etc.
In the second dimensions we might conclude that 1 company can have many contracts, and each contract can have many invoices. This indirectly also states that 1 company can have many invoices.

Creating a correct model requires digging deep into the business domain of whatever you are trying to build. It requires understanding both the current requirements as well as the future dreams. You don't have to model everything on day 1 but you do need to leave room to grow.

Once you start building your application, you will start embedding assumptions about the data model into your business logic. This means, the further along in the development process you are, the harder it becomes to change the data model because you will need to revisit all the business code affected by the changes. Changes to the data model become harder still once the application is live and compiling data. You might need to provide transition paths for the current data to fit the new model requirements so our updated business logic can also operate on older data.

This does not mean a data model is irrevocably rigid, but it does mean you need to preemtively estimate the cost of a wrong design decision in the data model. Some changes are trivial but some changes can have a huge impact. 

An example of a typical small change is adding a field. Realizing belatedly that an employee also has a middleName is not going to be a costly refactor. 
Larger changes are generally in how concepts relate to one another. If your whole application is built on the premise that 1 company has exactly 1 employee that performs all the roles, it might be a costly refactor if you suddenly realize a single company can have multiple employees with different roles.

The bigger a design decision, the more validation you need to ensure that the underlying assumptions are indeed correct.