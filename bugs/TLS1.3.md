In java 8.272, TLS1.3 support was backported from java 11 where it has been active since 2018.

Release notes: https://mail.openjdk.java.net/pipermail/jdk8u-dev/2020-October/012817.html
The original bug tracker entry: https://bugs.openjdk.java.net/browse/JDK-8248721

By default, the server will use this newer version, which ends up causing weird issues:

- Chrome throws an exception while loading javascript: ERR_INVALID_CHUNKED_ENCODING
- the css was fully loaded without the exception but the content was wrong, it had jumbled it up.

This behavior was reproduced in java 11 & 14 and is not present in 8.265 (the release before 272).
You can check the chosen TLS version in chrome -> developer tools -> security tab.
Note that the server has to be running in PRD mode so "chunked" mode is activated (though presumably it also occurs in non-chunking mode?)
Note that the problem only occurs if SSL is used.

A workaround is to disable TLS 1.3 in file jdk-14.0.2/conf/security/java.security:


```
# add the TLSv1.3 entry in the already existing line:
jdk.tls.disabledAlgorithms=SSLv3, TLSv1.3, RC4, DES, MD5withRSA, DH keySize < 1024, \
    EC keySize < 224, 3DES_EDE_CBC, anon, NULL, \
    include jdk.disabled.namedCurves
```

A particular post suggested enabling ``-Djdk.tls.acknowledgeCloseNotify=true``
But this does not work, it changes the output a little bit (you get a little bit more javascript before it fails), but not only does it still fail, the additional javascript is (like the css) out of order.

You can debug the SSL a bit using the system property: javax.net.debug

Potentially interesting links:

- https://bensmyth.com/files/Smyth19-TLS-tutorial.pdf
- https://stackoverflow.com/questions/54687831/changes-in-sslengine-usage-when-going-up-to-tlsv1-3: speaks of a potentially other issue because we don't explicitly close the inbound, but if we add closeInbound, we get an error in the javax.net.debug indicating that it should come from the client? because the issue seems to be resolved, we leave it turned off for now. 


If requests start to hang in TLS 1.3, check out the last stack overflow link.
