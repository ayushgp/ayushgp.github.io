---
layout: post
title: HTML Web Component using Vanilla JavaScript
---

Web Components have been around for a while now. Google has really been trying to push for their more widespread adoption, but most major browsers still have very little support for it, except for Opera and Chrome.

However, by using polyfills, available from [https://www.webcomponents.org/polyfills](https://www.webcomponents.org/polyfills), you can build your own Web Components now. 

In this article, I'm going to teach you how to create your own HTML tags with styles, functionality, and markup neatly packaged in their own files. 

## Introduction
Web Components are a set of web platform APIs that allow you to create new custom, reusable, encapsulated HTML tags to use in web pages and web apps. 

Custom components and widgets build on the Web Component standards, will work across modern browsers, and can be used with any JavaScript library or framework that works with HTML. 

Features to support Web Components are currently being added to the HTML and DOM specs, letting web developers easily extend HTML with new elements with encapsulated styling and custom behavior. 

It allows you to create reusable components using nothing more than vanilla JS/HTML/CSS. If HTML doesn't provide the solution to a problem, we can create a Web Component that does. 

For example, you have user data associated with an ID and want a component which fetches and populates that data given a user ID as an input. The HTML would be as follows:

```html
<user-card user-id="1"></user-card>
```

This is a pretty basic use case for Web Components. This tutorial will focus on building the user card component. 

## Four Pillars of Web Components
The HTML and DOM standards define four new standards/APIs that are helpful for defining Web Components. These standards are:

1. [Custom Elements](https://www.w3.org/TR/custom-elements/): With Custom Elements, web developers can create new HTML tags, beef-up existing HTML tags, or extend the components other developers have authored. This API is the foundation of Web Components. 
2. [HTML Templates](https://www.html5rocks.com/en/tutorials/webcomponents/template/#toc-pillars): It defines a new `<template>` element, which describes a standard DOM-based approach for client-side templating. Templates allow you to declare fragments of markup which are parsed as HTML, go unused at page load, but can be instantiated later on at runtime. 
3. [Shadow DOM](https://dom.spec.whatwg.org/#shadow-trees): Shadow DOM is designed as a tool for building component-based apps. It brings solutions for common problems in web development. It allows you to isolate DOM for the component and scope, and simplify CSS, etc.
4. [HTML Imports](https://www.html5rocks.com/en/tutorials/webcomponents/imports/): While HTML templates allow you to create new templates, HTML imports allows you to import these templates from different HTML files. Imports help keep code more organized by neatly arranging your components as separate HTML files.

## Defining a Custom Element
For creating a Custom element, we first have to declare a class for the custom element that defines how the element will behave. This class needs to extend the `HTMLElement` class. Let's take a detour and first discuss some of the lifecycle methods of custom elements. You can use the following lifecycle callbacks with custom elements:

- `connectedCallback` — Called every time the element is inserted into the DOM. 
- `disconnectedCallback` — Called every time the element is removed from the DOM.
- `attributeChangedCallback` — The behavior occurs when an attribute of the element is added, removed, updated, or replaced.

Create a new file called `UserCard.js` in a folder called `UserCard`.

```js
class UserCard extends HTMLElement {
  constructor() {
    // If you define a constructor, always call super() first as it is required by the CE spec.
    super();

    // Setup a click listener on <user-card>
    this.addEventListener('click', e => {
      this.toggleCard();
    });
  }

  toggleCard() {
    console.log("Element was clicked!");
  }
}

customElements.define('user-card', UserCard);
```

In this example, we have set up a Class that defines some of the behavior of our Custom Element, `user-card`. The `customElements.define('user-card', UserCard);` call tells the DOM that we have created a new custom element called `user-card`, whose behaviour is defined by `UserCard`. Now we can use the user-card element in our HTML. 

We'll be using the following API from `https://jsonplaceholder.typicode.com/` to create our User cards. Here's an example of how the data will look:
```json
{
  id: 1,
  name: "Leanne Graham",
  username: "Bret",
  email: "Sincere@april.biz",
  address: {
    street: "Kulas Light",
    suite: "Apt. 556",
    city: "Gwenborough",
    zipcode: "92998-3874",
    geo: {
      lat: "-37.3159",
      lng: "81.1496"
    }
  },
  phone: "1-770-736-8031 x56442",
  website: "hildegard.org"
}
```
### Creating a template
Now, let's create a template that'll be rendering this data on screen. Create a new file called UserCard.html with the following code: 

```html
<template id="user-card-template">
  <div class="card__user-card-container">
    <h2 class="card__name">
      <span class="card__full-name"></span> (
      <span class="card__user-name"></span>)
    </h2>
    <p>Website: <a class="card__website"></a></p>
    <div class="card__hidden-content">
      <p class="card__address"></p>
    </div>
    <button class="card__details-btn">More Details</button>
  </div>
</template>
<script src="/UserCard/UserCard.js"></script>
```

*Note: See that I've used class to have a prefix of `card__`. This is because in older browsers, we cannot isolate the DOM using shadow DOM. When styling the DOM, we won't be accidentally styling, say, a class called name.*

### Styling
We have now created a template for our card. Now, let's style it using CSS. Create a new file called UserCard.css in UsedCard folder with the following content: 

```css
.card__user-card-container {
  text-align: center;
  display: inline-block;
  border-radius: 5px;
  border: 1px solid grey;
  font-family: Helvetica;
  margin: 3px;
  width: 30%;
}

.card__user-card-container:hover {
  box-shadow: 3px 3px 3px;
}

.card__hidden-content {
  display: none;
}

.card__details-btn {
  background-color: #dedede;
  padding: 6px;
  margin-bottom: 8px;
}
```

Now, include this CSS file in your template using the following tag at the beginning of the `<template>` tag in the `UserCard.html` file: 
```html
<link rel="stylesheet" href="/UserCard/UserCard.css">
```

With our styles and templates in place, we can now move on to making our component functional.

### connectedCallback
Now we need to define what happens when we create an element and attach it to the DOM. Note that there is a difference between the `constructor` and the `connectedCallback` method.

`constructor` is called when an instance of the element is created, while `connectedCallback` is called every time the element is inserted into the DOM. It is useful for running setup code, such as fetching resources or rendering. 

*Note: At the top of your `UserCard.js` file, define a constant called `currentDocument`. It is needed in imported HTML's scripts to allow them access to the DOM of the imported HTML. Define it as follows:*
```js
const currentDocument = document.currentScript.ownerDocument;
```


Let us define our `connectedCallback`:

```js
  // Called when element is inserted in DOM
  connectedCallback() {
    const shadowRoot = this.attachShadow({mode: 'open'});

    // Select the template and clone it. Finally attach the cloned node to the shadowDOM's root.
    // Current document needs to be defined to get DOM access to imported HTML
    const template = currentDocument.querySelector('#user-card-template');
    const instance = template.content.cloneNode(true);
    shadowRoot.appendChild(instance);

    // Extract the attribute user-id from our element. 
    // Note that we are going to specify our cards like: 
    // <user-card user-id="1"></user-card>
    const userId = this.getAttribute('user-id');

    // Fetch the data for that user Id from the API and call the render method with this data
    fetch(`https://jsonplaceholder.typicode.com/users/${userId}`)
        .then((response) => response.text())
        .then((responseText) => {
            this.render(JSON.parse(responseText));
        })
        .catch((error) => {
            console.error(error);
        });
  }
```

### Rendering the user data
We have our `connectedCallback` in place now. We created a shadow root and attached our template's clone to it. Now we need to populate that clone. For that, we called the `render` method from our `fetch` call. Let's create the `render` method and `toggleCard` method. 

```js
  render(userData) {
    // Fill the respective areas of the card using DOM manipulation APIs
    // All of our components elements reside under shadow dom. So we created a this.shadowRoot property
    // We use this property to call selectors so that the DOM is searched only under this subtree
    this.shadowRoot.querySelector('.card__full-name').innerHTML = userData.name;
    this.shadowRoot.querySelector('.card__user-name').innerHTML = userData.username;
    this.shadowRoot.querySelector('.card__website').innerHTML = userData.website;
    this.shadowRoot.querySelector('.card__address').innerHTML = `<h4>Address</h4>
      ${userData.address.suite}, <br />
      ${userData.address.street},<br />
      ${userData.address.city},<br />
      Zipcode: ${userData.address.zipcode}`
  }

  toggleCard() {
    let elem = this.shadowRoot.querySelector('.card__hidden-content');
    let btn = this.shadowRoot.querySelector('.card__details-btn');
    btn.innerHTML = elem.style.display == 'none' ? 'Less Details' : 'More Details';
    elem.style.display = elem.style.display == 'none' ? 'block' : 'none';
  }
```

Now that we have our component in place, we can use it in our projects. Any of them. So for the sake of this tutorial, create a new HTML file called `index.html` and write the following code in it: 

```html
<html>

<head>
  <title>Web Component</title>
</head>

<body>
  <user-card user-id="1"></user-card>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/webcomponentsjs/1.0.14/webcomponents-hi.js"></script>
  <link rel="import" href="./UserCard/UserCard.html">
</body>

</html>
```

We need to add the webcomponents.js file, as not all browsers support Web Components. Note that we're using the HTML import statement to import our component from the directory. 

To run this code, you'll need to create a static file server. If you don't know how to do that, you can use a simple static server like [`static-server`](https://www.npmjs.com/package/static-server) or [`json-server`](https://github.com/typicode/json-server). For this tutorial, install `static-server` using:

```bash
$ npm install -g static-server
```

Now, navigate to your folder containing the index.html file using `cd` and run the server using:
```bash
$ static-server
```

Open your browser and go to [localhost:3000](http://localhost:3000), and you should see the component we just created. 

## Tips and Tricks
There are a lot of things we did not cover in this relatively short article about Web Components. I'd like to succinctly state some tips and tricks that were useful when developing Web Components.  

### Naming Components
- The name of a custom element must contain a dash. So `<my-tabs>` and `<my-amazing-website>` are valid names, while `<foo>` and `<foo_bar>` are not. This requirement is so the HTML parser can distinguish custom elements from regular elements. 
- It also ensures forward compatibility when new tags are added to HTML. You can't register the same tag more than once. 
- Custom elements cannot be self-closing because HTML only allows a few elements to be self-closing. Always write a closing tag (`<app-drawer></app-drawer>`).

### Extending Components
You can use inheritence while creating components. For example, if you want to create a `UserCard` for two different types of users, you can first create a generic UserCard and extend it in the two specialized user cards. For more info on inheritence in components, refer to this [Google web developers' article.](https://developers.google.com/web/fundamentals/web-components/customelements#extend)

### Lifecycle Callbacks
We created `connectedCallback`, which is automatically called when our element gets attached to the DOM. We also have `disconnectedCallback` that gets called when our element gets removed from the DOM. `attributesChangedCallback(attribute, oldval, newval)` is called when we change an attribute of a custom element. 

### Elements are instances of classes
Since elements are instances of classes, you can define public methods on these classes that can be used to allow other custom elements/scripts to interact with these elements rather than changing their attributes. 

### Defining private methods
You can define private methods many different ways. I prefer using IIFEs, as they are easy to write and understand. For example, if you are creating a component that has very complex internal workings, you could do something like: 

```js
(function() {

  // Define private functions here with first argument as self
  // When calling these functions, pass this from the class 
  // This is a way you can use private functions in JS
  function _privateFunc(self, otherArgs) { ... }

  // Now this is available only in this scope and can be used by your class here:
  class MyComponent extends HTMLElement {
    ...

    // Define functions like this that are accessible to interact with this element.
    doSomething() {
      ...
      _privateFunc(this, args)
    }
    ...
  }

  customElements.define('my-component', MyComponent);
})()
```

### Freeze Class definitions
Freeze your class definitions to prevent new properties from being added to it. Preventing existing properties from being removed and preventing existing properties, or their enumerability, configurability, or writability from being changed, also prevents the prototype from being changed. You can do this using:

```js
class MyComponent extends HTMLElement { ... }
const FrozenMyComponent = Object.freeze(MyComponent);
customElements.define('my-component', FrozenMyComponent);
```

## Conclusion
The tutorials out there on Web Components are very limited. This can be blamed partly on React, which has mostly shadowed Web Components. I hope this article gives you enough information to go and build your own custom components without any dependencies. You can check out the [Custom components API spec](https://www.w3.org/TR/custom-elements/) for more info on Web Components.

We've barely scratched the surface of Web Components in this article. If you want me to write more tutorials on Web Components, feel free to contact me. 
