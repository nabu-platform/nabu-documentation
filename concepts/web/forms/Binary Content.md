# Binary Content

There are two ways to upload binary content to the Nabu Platform using a form. One entails a dedicated REST service that takes only the binary file as input. The other entails an embedded field that can be part of a larger JSON or XML upload. 

Only the second method allows you to have file uploads as part of a larger form submit. However, the second method adds 33% overhead to the file size as it needs to encode the file contents to fit into a serialized format.

# Method 1: Dedicated REST Service

In this approach, we have a REST service that only expects binary input. This allows us to upload exactly one file at a time and limits the amount of additional data we can send along. It is not ideal for complex forms but it is perfect when you want to upload arbitrarily large files without bumping into memory issues.

## The REST Service

We set the ``input as stream`` option in the REST service to true, this allows it to receive binary content. Once you enable this option, the REST service will automatically add metadata like the file name and the content type based on the header information it gets from the browser. 

As a reminder: streams are different from other data types as they can only be read once.

[^.resources/binary/method1-rest.mp4]

## The Form

In the form component we choose the REST service we just created, add a field that binds to the "body" and choose type file. The page builder will make sure the correct headers are set so interesting metadata like file name and content type make it to the backend.

[^.resources/binary/method1-configure-form.mp4]

Now can test the end result.

[^.resources/binary/method1-run-form.mp4]

# Method 2: Embedded Field

In this approach, we want to upload one or more files and possibly in the context of a larger form. Imagine for example a registration form where you want to capture an avatar image during the signup.
The data is [base64 encoded](https://en.wikipedia.org/wiki/Base64) which means during transit it is 33% bigger than the original file.

While this approach is perfect for most form uses, uploading arbitrarily large file can bump into memory limits. There are advanced ways around this, but if that is your usecase, I would consider the first method.

## The REST Service

The REST service for this approach is more like a regular one where we create a defined data type to serve as the input. The field where we want to upload our binary data needs to have the simple type ``base64Binary``.

[^.resources/binary/method2-create-structure.mp4]

We then implement the REST service much like we did in method 1. We currently ignore the firstName and lastName fields but one can imagine that we create a user profile from them.

[^.resources/binary/method2-rest.mp4]

## The Form

In the form component we choose our new REST service and bind the content field that has the correct ``base64Binary`` type as the file content.
Because it is often interesting to capture some metadata about the file as well, you can optionally bind additional fields that will capture the original content type and file name. This is entirely optional however.

[^.resources/binary/method2-configure-form.mp4]

Now can test the end result of our second method.

[^.resources/binary/method2-run-form.mp4]
