# Binary Content

There are two ways to upload binary content to the Nabu Platform using a form. One entails a dedicated REST service that takes only the binary file as input. The other entails an embedded field that can be part of a larger JSON or XML upload. 

Only the second method allows you to have file uploads as part of a larger form submit. However, the second method adds 33% overhead to the file size as it needs to encode the file contents to fit into a serialized format.

# Method 1: Dedicated REST Service

## The REST Service

We set the ``input as stream`` option in the REST service to true, this allows it to receive binary content. Once you enable this option, the REST service will automatically add metadata like the file name and the content type. 

As a reminder: streams are different from other data types as they can only be read once.

[^.resources/binary/method1-rest.mp4]

The advantage of this method is that you receive the original binary data without any overhead. With some additional settings you can receive arbitrarily large files without impacting the memory usage of your HTTP server.

## The Form

In the form component we choose the REST service we just created, add a field that binds to the "body" and choose type file.

[^.resources/binary/method1-configure-form.mp4]

Now can test the end result.

[^.resources/binary/method1-run-form.mp4]

# Method 2: Embedded


## The REST Service

## The Form


