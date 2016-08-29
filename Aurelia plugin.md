## Aurelia CLI Sketch plugin

This plugin can read an Aurelia CLI project and create Sketch artboards with layers and appropriate meta-data using layer tagging.

We can add and manage these extra tags (metadata) in a similar fashion as demonstrated by [Sketch Notebook](http://marcosvid.al/sketch-notebook/), see [repo](https://github.com/marcosvidal/Sketch-Notebook).

It uses a separate panel with comments for each layer, we could likewise add/manage layer metadata.

## Read CLI project
The Aurelia project reader needs to handle the following:

- CSS styling
- Globals
- Routers & Templates
- Custom elements & attributes
- Routing & composition
- File locations

### CSS styling
Add a plugin command option to open/parse an Aurelia project via path or Cocoa file open dialog. Project is read via `aurelia_project/aurelia.json` file, and `/src` folder.

From the `aurelia.json` we gather all the `.css` sources found in `vendor-libs.js` bundle. We also walk any directory of `app.js` bundle to find css files. we then build up a global path->css file map for use later.

We use the [CSS sketch plugin](https://medium.com/sketch-app-sources/programmers-design-differently-why-i-built-a-css-plugin-for-sketch-3-52a1246305a4#.bzdzhpg2k) with [repo](https://github.com/JohnCoates/CSSketch) to link the CSS to the Sketch layers.

### Globals
Custom elements can be imported either via `<require>` or by making global elements, by convention in `resources/elements`
We assume any element here is global and create an elements map:

### Parsing routers & templates
After making a global CSS map we start parsing the `/src` folder for routers and templates (components).

We assume the routes are defined in a `routes.json` file or multiple `/routes/*.json` files using the [Aurelia route loader](https://www.npmjs.com/package/aurelia-router-loader) plugin.

```json
[
      {
          "route": "users",
          "name": "users",
          "moduleId": "users",
          "nav": true,
          "title": "Github Users"
      },
      {
          "route": "child-router",
          "name": "child-router",
          "moduleId": "child-router",
          "nav": true,
          "title": "Child Router"
      }
]
```

We fallback to parsing the view model, such as `app.js` for the root component if no `.json` files exists. We use a reg exp to extract all inside `config.map\((.*)\)`

```js
config.map([
 { route: ['','home'],  name: 'home',  
    moduleId: './components/home/home',  nav: true, title:'Home' },
 { route: 'about',  name: 'about',
    moduleId: './components/about/about',    nav: true, title:'About' }
  ]);
```

We also parse `config.mapUnknownRoutes('not-found');` to create an arboard for any unknown routes.

### Root view
The root template, by convention called `app.html`. It will create an artboard for the app, where the top layer (group) will be tagged `view: 'app'`.

### Routes
See [Router demo](https://scottwhittaker.net/aurelia/2016/06/12/aurelia-router-demo.html) and [Router docs](http://aurelia.io/hub.html#/doc/article/aurelia/router/latest/router-configuration/1)

The `<router-view>` element will create a dummy layer tagged:
`router-view: default` and the top level layer will be tagged `router:true`.

Any element with `href.bind`

### Custom elements
We make a map of loaded custom elements `<require from="components/navigation.html"></require>`

For `<navigation router.bind="router" class="primary-navigation"></navigation>`By convention `router.bind` is the router. When we then parse the custom element, using `nav: true` config indicator to filter routes that should have a navigation link displayed.

```html
  <li repeat.for="row of router.navigation" class="${row.isActive ? 'active' : ''}">
<a href.bind="row.href">${row.title}</a>
  </li>
```

We build the metadata using the `routes.json` data. We assume the first row is active for `class`. We also substitute `row.title` via the router map.

```html
  <li class="au-target" au-target-id="1">
<a href.bind="row.href" class="au-target" au-target-id="2" href="#/profile">Profile</a>
  </li>
```

Sometimes `route-href` is used instead for more complex routes:

```html
<li repeat.for="contact of contacts" class="list-group-item ${contact.id === $parent.selectedId ? 'active' : ''}">
<a route-href="route: contacts; params.bind: {id:contact.id}"
```

For the `repeat.for` we have to disregard it in most (complex) cases, but we can always try to make one `<li>`. We also disregard any string interpolation.

The router anchors are tagged layers, where the href `#/profile` points to a layer of the same name.

For each route with we create a new artboard, reusing the `router:true` view, substituting the `router-view` with a route target template.

### Multiple router views and slots

For multiple router views we have to create more artboard combinations!

*Router view*

Layer tagged with: `router-view: left`

```html
<template>
  <div class="page-host">
    <router-view name="left"></router-view>
  </div>
  <div class="page-host">
    <router-view name="right"></router-view>
  </div>
</template>
```

*Layout*

Special layer tagged with: `slot-target: aside-content`

```html
<template>
  <div class="left-content">
    <slot name="aside-content"></slot>
  </div>
  <div class="right-content">
    <slot name="main-content"></slot>
  </div>
</template>
```

*Slots*

Layer tagged with: `slot: main-content`

```html
<template>
  <div slot="main-content">
    <p>I'm content that will show up on the right.</p>
  </div>
  <div slot="aside-content">
    <p>I'm content that will show up on the left.</p>
  </div>
</template>
```

For more on slots, see [content projection](http://aurelia.io/hub.html#/doc/article/aurelia/templating/latest/templating-content-projection/1)

*Router*

```js
config.map([
  { route: 'home', name: 'home', moduleId: 'home/index' },
  { route: 'login', name: 'login', moduleId: 'login/index', layoutView = 'views/layout-login.html' },
  { route: 'users', name: 'users', moduleId: 'users/index', layoutViewModel = 'views/model', layoutModel: model }
]);
```

### Layouts

`<router-view layout="views/layout-default.html"></router-view>`

### Custom elements attributes
For each custom attribute, we tag the layer with it, so we can re-export without losing it. For tagging, see [Tagged layers example](https://github.com/kristianmandrup/ExampleSketchPlugins/blob/master/Tagged%20Layers/Tagged%20Layers.sketchplugin/Contents/Sketch/script.cocoascript)

`addAttribute(layer, "value.bind", "colors")`

```js
  var command = context.command; // The current command: MSPluginCommand)

function addAttribute(layer, name, value) {
  [command setValue:value forKey:name onLayer:layer];
}

function getAttribute(layer, name) {
  return [command valueForKey: name onLayer:layer]
}
```

Custom attributes are those with `.` or `-`. For the compose element, we can insert in place if `view.bind` points to a custom element (file) otherwise we ignore it.

```
  <compose view-model="hello"
       view.bind="hello.html"
       model.bind="{ target : 'World' }" ></compose>
```

We can use `as-element` as a kind of special compose.

```
<template>
  <require from="./hello-row.html"></require>
  <table>
    <tr as-element="hello-row">
  </table>
</template>
```

### File Locations

For each component that we turn into a layer/artboard, we need to store its original file location as a tag as well: `file-path: contacts/contact-detail.html`

## Create CLI project

It should be possible to take an existing Sketch project with artboards and tag with meta data to create sufficient info to generate a basic Aurelia CLI `/src` folder with *View Models* and *Views* to reflect it.


The plugin should have the following commands:

- `app view` - tag a layer as an app view
- `router view` - tag a layer as a `<router-view>` element
- `route navigation` - tag a layer with route navigation info:
- `element`: - specify an element output tag to use, not the default `<div>`. Useful to export buttons etc.

*route navigation*
- `name` the name of the route (required)
- `view` - the layer (artboard) to route to (use default convention)
- `route` - the routing pattern (use default convention)

The route navigation tags are also used to generate a `routes.json` file
The default convention is to use the (layer) `name` for each, unless specified directly.

## Write CLI project
Each artboard/layer tagged with `app view` is a separate application.
Each nested layer turns into an appropriate tag, usually a `<div>` or `<p>`, see [Blade](https://github.com/kristianmandrup/blade)

Layers can be tagged with element name (using plugin command with dialog) or use special naming convention such as `input#hello.container` to give the element name `input` and id `hello` and class `container`.

The write should write into the `src` folder of an existing CLI app. We could have an export option to first run the CLI generator using `au new`.


