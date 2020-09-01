# Custom Vue Component

You can expose a standard vue component created with ``Vue.component`` but that requires manually registering the necessary route, describing the input parameters etc. It is a lot easier to use the custom ``Vue.view`` which does these things automatically. Apart from the initial name it is entirely compatible with the standard ``Vue.component``.

Suppose you create this component:

```javascript
Vue.view("hello-world", {
	template: "<div>{{message}}</div>",
	props: {
		message: {
			type: String,
			default: "hello world!"
		}
	}
});
```

Note that in ``Vue.view`` the template parameter is _optional_. If not filled in, it will look for a template with the same name as the component. In the above example if we were to remove the template declaration, it would have the same behavior as if you wrote ``template: "#hello-world"``.

If we then open up page builder, it will be automatically picked up and available in the dropdown. The input parameter ``message`` is automatically exposed and can be bound to other state or you can set a fixed value.

[^.resources/custom-vue-component.mp4]
