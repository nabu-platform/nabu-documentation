# Security

## Custom Context

In some cases you don't have the node id in the REST path, but instead have some other identifier. This could for example be when designing an API for third party consumption, you might agree on a particular external identifier rather than our internal CMS identifiers.

In that case you can continue using the CMS security but need to register a custom resolver that tells the CMS which node is relevant for the permission context.

For example suppose you have this path in a REST service:

```
/customer/{vat}/invoices
```

You want to have the security check whether the person asking has access to that customer invoices. The permission context setting of the REST service might look like this:

```
="vat:" + input/path/vat
```

You create a new blox service and implement the interface ``nabu.cms.core.interfaces.contextResolver``.

As input you get the `type` which would be the string literal 'vat' (allowing for one resolver to resolve multiple types) and the `context` which contains the actual runtime value for the vat.
You then select the correct node based on that VAT and return it.

Finally, in the CMS configuration for that web application, you add an entry to the ``context`` list where you configure type `vat` to match the string literal you hardcoded in the permission context and as `contextResolver` the blox service you created.
