# Javascript

Page builder is the visual layer which wraps around a host of underlying custom javascript frameworks. While the goal is of course to build entire applications without ever writing javascript code, new components may from time to time be necessary to add functionality that has not yet been added.

Before we dive into how you can add your custom components, I want to quickly describe the basic building blocks that are used.

## Vue

Vue is the framework that we use to build our SPA's. Page builder ships with a slightly customized version of Vue which integrates further into the Nabu stack, for example couping debug mode in Vue to the development mode in Nabu, adding support for our server renderer,... The customization is documented at the top of the vue file that is included.

Over the past few years a lot of libraries have been created that add a ton of functionality, some of those core libraries are split into two parts:

- vanilla javascript
- vue-specific extension

Some examples:

- **basics**: there are a ton of basic libraries and utilities covering promises, ajax, form components,...
- **router**: the core router allows for complex routing rules, the core router does all the heavy lifting while the vue extension ensures that the creation, rendering and destroying is done according to the Vue lifecycle diagram. The vue extension also adds support for asynchronous component activation. Any Vue component can be registered as a valid route though there are utilities that make this a lot easier.
- **service**: we have services in the frontend which are global singletons meant to maintain state and (when custom coding) include logic around that state. The vanilla service library has support for asynchronous service initialization (for example to build up state via ajax calls), lazy service initialization (to only initialize a service upon first use), inter-service dependencies (to guarantee particular services are fully loaded and available before the new service is initialized),... The vue service extension makes services as Vue.component which makes it easy to have all the stored data be reactive and make use of anything vue has to offer (e.g. watchers, computed,...). You can technically expose a 
- **swagger client**: the swagger client is at the core of all the static awareness page builder has about the data types involved. Swaggers are automatically constructed by the backend and the swagger client can read those generated swagger and allow for easy service calling. By executing swagger operations you bring the frontend more in line with the backend which is service oriented: with the swagger client you call services rather than http endpoints. Additionally all the validations you enable in the backend (e.g. regexes, max length,...) are exposed through swagger and can be automatically fed into the form components, preventing the need to duplicate all the validation rules.

The core libraries are split into two parts:

- vanilla javascript
- vue-based javascript