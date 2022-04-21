# Data Format

When you hear terms like "parsing" and "formatting" or "marshalling" and "unmarshalling", you are generally in the realm of data formatting.

What a computer "knows", resides in its memory or file system as a long series of 0's and 1's. However the specific sequence of 0's and 1's can depend on a number of factors:

- the type of hardware it is running on
- the operating system that is running on it
- the chosen file system
- the application that is manipulating the data
...

When it wants to transmit that knowledge to another system, it needs to communicate that data in a way that is independent of all these factors because the other system might not be set up the same way.

## Syntax

This is where our syntax structure comes into play, the computer can marshal the data according to a specific syntax, pass that to another system who can use the same syntax to unmarshal the original data into its own memory for further processing.
There are many syntactical constructs that may or may not be supported by certain systems. Finding a syntactical construct supported by all systems that are involved is the first step. Some examples:

JSON:

```json
{ 
	"company": {
		"vat": "BE123",
		"name": "Example Company"
	}
}
```

XML:

```xml
<company>
	<vat>BE123</vat>
	<name>Example Company</name>
</company>
```

YAML

```
---
company:
  vat: BE123
  name: Example Company
```


CSV:

```
vat,name
BE123,Example Company
```

All of these convey the same data, just in different formats. These are all "mainstream" formats meaning there is a large chance most systems will support these. While XML used to dominate the enterprise space, JSON is currently the most dominant data exchange format. There are other, lesser known formats that might be used in specific settings or for specific purposes. EDI for example reigned supreme for a long time but has been steadily replaced by newer formats.

People can also invent their own formats for specific reasons, for example custom flat files definitions that use fixed length or custom separators are still being used. They might not be as easily human readable as JSON, but they are generally smaller in size, allowing more data to be contained in the same file size.



--------

binary vs text
flat file
streaming mode, windowing
json specifics with key/value pairs and all the other specialities