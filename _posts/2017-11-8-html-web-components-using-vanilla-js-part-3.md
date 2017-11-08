---
layout: post
title: HTML Web Component using Vanilla JS - Part 3
comments: true
---

This is the third part of the web components tutorial series I'm writing. In this part we're gonna have a look at attributes, how and when to use them and the parts of the custom elements specs that deals with them. Check out [part 1](https://ayushgp.github.io/html-web-components-using-vanilla-js/) and [part 2](https://ayushgp.github.io/html-web-components-using-vanilla-js-part-2) of this series first.

Quoting from MDN:

> Elements in HTML have **attributes**; these are additional values that configure the elements or adjust their behavior in various ways to meet the criteria the users want.

## What we'll build?
We'll create a component UserCard with the attributes: username, address and is-admin(Boolean that tells us whether user is admin or not). We'll also watch these attributes for changes and update the component accordingly. 

## Getting and Setting attributes
We can easily define attributes in our HTML code like:

```html
<user-card username="Ayush" address="Indore, India" is-admin></user-card>
```

We can use the DOM API in JavaScript to get and set attributes using the `getAttribute(attrName)` and `setAttribute(attrName, newVal)` methods. For example,

```js
let myUserCard = document.querySelector('user-card')
myUserCard.getAttribute('username') // Ayush
myUserCard.setAttribute('username', 'Ayush Gupta') 
myUserCard.getAttribute('username') // Ayush Gupta
```

## Watching for attribute changes
The Custom Elements specs v1 defines an easy way to observe attribute changes and taking action on these changes. When creating our component we need to define 2 things: 

1. *Observed Attributes*: To be notified when attributes change, a list of observed attributes must be defined when initializing the element, by placing an static `observedAttributes` getter on the element's class that returns an array of attribute names.

2. `attributeChangedCallback(attributeName, oldValue, newValue, namespace)`: A lifecycle method which is called when an attribute is changed, appended, removed, or replaced on the element. It is only called for observed attributes.

## Creating UserCard Component
Let's build our `UserCard` component that'll be initialized using attributes and any changes made to its attributes will be observed by our component. We'll organize this project in the same way as [part 1 of this series](https://ayushgp.github.io/html-web-components-using-vanilla-js/). 

Create an `index.html` file in the project directory. Also create a UserCard directory with files: `UserCard.html`, `UserCard.css` and `UserCard.js`.

Open the `UserCard.js` file and enter the following code:
```js
(async () => {
  const res = await fetch('/UserCard/UserCard.html');
  const textTemplate = await res.text();
  const HTMLTemplate = new DOMParser().parseFromString(textTemplate, 'text/html')
                           .querySelector('template');

  class UserCard extends HTMLElement {
    constructor() { ... }
    connectedCallback() { ... }
    
    // Getter to let component know what attributes
    // to watch for mutation
    static get observedAttributes() {
      return ['username', 'address', 'is-admin']; 
    }

    attributeChangedCallback(attr, oldValue, newValue) {
      console.log(`${attr} was changed from ${oldValue} to ${newValue}!`)
    }
  }

  customElements.define('user-card', UserCard);
})();
```

Now that we have a basic framework ready, lets build this component.

### Initializing using attributes
When we'll create the component in our markup(aka HTML), we'll provide it some starting values which it'll use to initialize the component. For example,

```html
<user-card username="Ayush" address="Indore, India" is-admin="true"></user-card>
```

Now in the `connectedCallback`, we'll take these attributes and define a variable corresponding to each of them.

```js
connectedCallback() {
  const shadowRoot = this.attachShadow({ mode: 'open' });
  const instance = HTMLTemplate.content.cloneNode(true);
  shadowRoot.appendChild(instance);

  // You can also put checks to see if attr is present or not
  // and throw errors to make some attributes mandatory
  // Also default values for these variables can be defined here
  this.username = this.getAttribute('username');
  this.address = this.getAttribute('address');
  this.isAdmin = this.getAttribute('is-admin');
}

// Define setters to update the DOM whenever these values are set
set username(value) {
  this._username = value;
  if (this.shadowRoot)
    this.shadowRoot.querySelector('#card__username').innerHTML = value;
}

get username() {
  return this._username;
}

set address(value) {
  this._address = value;
  if (this.shadowRoot)
    this.shadowRoot.querySelector('#card__address').innerHTML = value;
}

get address() {
  return this._address;
}

set isAdmin(value) {
  this._isAdmin = value;
  if (this.shadowRoot)
    this.shadowRoot.querySelector('#card__admin-flag').style.display = value == true ? "block" : "none";
}

get isAdmin() {
  return this._isAdmin;
}
```

The `attributeChangedCallback` is called when an observed attribute is changed. So we'll need to define what happens when any of these attributes change. Rewrite the function to contain the following:

```js

attributeChangedCallback(attr, oldVal, newVal) {
  const attribute = attr.toLowerCase()
  console.log(newVal)
  if (attribute === 'username') {
    this.username = newVal != '' ? newVal : "Not Provided!"
  } else if (attribute === 'address') {
    this.address = newVal !== '' ? newVal : "Not Provided!"
  } else if (attribute === 'is-admin') {
    this.isAdmin = newVal == 'true';
  }
}
```
Now create the template that our component will be using to complete setting up the component.

UserCard.html
```html
<template id="user-card-template">
  <h3 id="card__username"></h3>
  <p id="card__address"></p>
  <p id="card__admin-flag">I'm an admin</p>
</template>
```

## Using our component
Create index.html file with 2 input elements and a checkbox and define onchange methods for all of these to update our component's attributes. As soon as attributes are updated, the change will also be reflected in the DOM.
```html
<html>

<head>
  <title>Web Component</title>
</head>

<body>
  <input type="text" onchange="updateName(this)" placeholder="Name">
  <input type="text" onchange="updateAddress(this)" placeholder="Address">
  <input type="checkbox" onchange="toggleAdminStatus(this)" placeholder="Name">
  <user-card username="Ayush" address="Indore, India" is-admin></user-card>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/webcomponentsjs/1.0.14/webcomponents-hi.js"></script>
  <script src="/UserCard/UserCard.js"></script>
  <script>
    function updateAddress(elem) {
      document.querySelector('user-card').setAttribute('address', elem.value);
    }

    function updateName(elem) {
      document.querySelector('user-card').setAttribute('username', elem.value);
    }

    function toggleAdminStatus(elem) {
      document.querySelector('user-card').setAttribute('is-admin', elem.checked);
    }
  </script>
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

Open your browser and go to [localhost:9080](http://localhost:9080), and you should see the component we just created. 

![web-components-part-3](https://user-images.githubusercontent.com/7992943/32566632-8b030bde-c4de-11e7-98ff-9be1534c2c2b.gif)


## When to use attributes
In the previous post we had created an API for both our child components so that the parent component could initialize and interact with them using this API. In that case, if we already have some config we wish to provide directly without using a parent/other function call, we couldn't do it. 

With attributes we can provide that initial config very easily. This config can then be extracted in either `constructor` or `connectedCallback` to initilize the component. Changing attributes to interact with components can get a bit tedious though. Suppose you want to pass large amounts of json data to the component. Doing that would require the json to be represented as a string attribute and to be parsed when being used by the component.

We have 3 ways in which we can create interactive web components:
1. Using only attributes: This is the approach we saw in this post. We used attributes for initializing the components as well as interacting with them from outside world.

2. Using only created functions: This is the approach we saw in part 2 of the series where we initialized and interacted with the components using only the functions we created for them. 

3. Using a mixed approach: IMO this should be used. In this approach we initialize the components using attributes and for all later interactions just use calls to its API. 

Let me know which approach would you prefer and why in the comments below! Read the [part 1](https://ayushgp.github.io/html-web-components-using-vanilla-js/) and [part 2](https://ayushgp.github.io/html-web-components-using-vanilla-js-part-2) of this series if you haven't already.
