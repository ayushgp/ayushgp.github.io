---
layout: post
title: HTML Web Component using Vanilla JS - Part 2
comments: true
---

I've earlier written a [post on how to create vanilla JS Web Components](https://ayushgp.github.io/html-web-components-using-vanilla-js/) using the new API spec introduced by W3C for [Custom Elements](https://www.w3.org/TR/custom-elements/), [Shadow DOM](https://dom.spec.whatwg.org/#shadow-trees), [HTML Imports](https://www.html5rocks.com/en/tutorials/webcomponents/imports/) and [`<template>` tag](https://www.html5rocks.com/en/tutorials/webcomponents/template/#toc-pillars). 

The previous post showed how to create a very simple and not that useful web component. In this post I'll teach you how to create multiple components and make them interact with each other and organizing your code. These are what I used when I learnt when I built an app using web components. 

## What we'll build
We'll be building 3 components. 
The first component would be a List of people. 
The second component would display the information of person we select from first component. 
The parent component would orchestrate these components and allow us to independently develop the child components and plug them together. 

## Code Organization
We'll be creating a `components` directory to contain all of our components. Each component will have its own directory that'll contain the component's HTML template, JS and stylesheets. Components that are just used to create other components and are not reused will be placed in that components directory. So in our case the directory structure will look like:

```
src/
	index.html
	components/
    PeopleController/
      PeopleController.js
      PeopleController.html
      PeopleController.css
      PeopleList/
        PeopleList.js
        PeopleList.html
        PeopleList.css
      PersonDetail/
        PersonDetail.js
        PersonDetail.html
        PersonDetail.css
```

We'll be using the following API from `https://jsonplaceholder.typicode.com/` to get some placeholder user data. Here's an example of how the data will look:
```js
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

## Child components

### People List component

Let's start by building the `PeopleList` component. Create the `PeopleList.html` file with the following contents:

```html
<template id="people-list-template">
  <style>
  .people-list__container {
    border: 1px solid black;
  }
  .people-list__list {
    list-style: none
  }

  .people-list__list > li {
    font-size: 20px;
    font-family: Helvetica;
    color: #000000;
    text-decoration: none;
  }
  </style>
  <div class="people-list__container">
    <ul class="people-list__list"></ul>
  </div>
</template>
<script src="/components/PeopleController/PeopleList/PeopleList.js"></script>
```

The `ul.people-list__list` will contain the list of the names we get. Now create the class `PeopleList` with the `constructor`, `connectedCallback` and `render` functions inside an IIFE.

```js
(function () {
  const currentDocument = document.currentScript.ownerDocument;

  // Private Methods will go here:
  // ...

  class PeopleList extends HTMLElement {
    constructor() {
      // If you define a constructor, always call super() first as it is required by the CE spec.
      super();

      // A private property that we'll use to keep track of list
      let _list = [];

      // Use defineProperty to define a prop on this object, ie, the component.
      // Whenever list is set, call render. This way when the parent component sets some data 
      // on the child object, we can automatically update the child.
      Object.defineProperty(this, 'list', {
        get: () => _list,
        set: (list) => {
          _list = list;
          this.render();
        }
      });
    }

    connectedCallback() {
      // Create a Shadow DOM using our template
      const shadowRoot = this.attachShadow({ mode: 'open' });
      const template = currentDocument.querySelector('#people-list-template');
      const instance = template.content.cloneNode(true);
      shadowRoot.appendChild(instance);
    }

    render() {
      // ...
    }
  }

  customElements.define('people-list', PeopleList);
})();
```

In the `render` method, we need to create a list of people names using `<li>`. We will also create a `CustomEvent` for each of the elements. Whenever that element is clicked, its id will propagated with the event upwards in the DOM tree.

We're doing this because it makes our child elements independent of the parent or other sibling elements. We'll watch for this event in our parent component and update the sibling component accordingly from the parent. More on that later. Add the following code to your `render` function:

```js
render() {
  let ulElement = this.shadowRoot.querySelector('.people-list__list');
  ulElement.innerHTML = '';

  this.list.forEach(person => {
    let li = _createPersonListElement(this, person);
    ulElement.appendChild(li);
  });
}
```

Also create a function inside the IIFE but outside your class definition called `_createPersonListElement(person)`. This will be used to create `li` elements with the person information. <i>Note: I've done it this way as it is a great way of using private function in your JS code. </i>

```js
function _createPersonListElement(self, person) {
  let li = currentDocument.createElement('LI');
  li.innerHTML = person.name;
  li.className = 'people-list__name'
  li.onclick = () => {
    let event = new CustomEvent("PersonClicked", {
      detail: {
        personId: person.id
      },
      bubbles: true
    });
    self.dispatchEvent(event);
  }
  return li;
}
```

### PersonDetail component

We've created the `PeopleList` component that'll list the people by names. We also want to create a component that'll show the people details when the person name is clicked in that component. So lets reuse the component we used in the [previous tutorial](https://ayushgp.github.io/html-web-components-using-vanilla-js/), `UserCard`. I won't go into the details of how I'm building this component but just put the code here. You can read more about it in the older post.

#### Template
Open the `PersonDetail.html` file and put the following code in it:

```html
<template id="person-detail-template">
  <link rel="stylesheet" href="/components/PeopleController/PersonDetail/PersonDetail.css">
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
<script src="/components/PeopleController/PersonDetail/PersonDetail.js"></script>
```

#### Styling
We have now created a template for our card. Now, let’s style it using CSS. Create a new file called PersonDetail.css in UsedCard folder with the following content:

```css
.card__user-card-container {
  text-align: center;
  border-radius: 5px;
  border: 1px solid grey;
  font-family: Helvetica;
  margin: 3px;
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

#### Component Scripting

Create the /components/PeopleController/PersonDetail/PersonDetail.js file and provide the PeopleDetail component its functionality using the following code:

```js
(function () {
  const currentDocument = document.currentScript.ownerDocument;

  class PersonDetail extends HTMLElement {
    constructor() {
      // If you define a constructor, always call super() first as it is required by the CE spec.
      super();

      // Setup a click listener on <user-card>
      this.addEventListener('click', e => {
        this.toggleCard();
      });
    }

    // Called when element is inserted in DOM
    connectedCallback() {
      const shadowRoot = this.attachShadow({ mode: 'open' });
      const template = currentDocument.querySelector('#person-detail-template');
      const instance = template.content.cloneNode(true);
      shadowRoot.appendChild(instance);
    }

    // Creating an API function so that other components can use this to populate this component
    updatePersonDetails(userData) {
      this.render(userData);
    }

    // Function to populate the card(Can be made private)
    render(userData) {
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
  }

  customElements.define('person-detail', PersonDetail);
})()
``` 

We have created a function `updatePersonDetails(userData)` so that we can update this component using this function when a Person is clicked in out `PeopleList` component. We could have also done this using attributes.

### Parent Component

Now that we have both our `PeopleList` component and `PersonDetail` component in place, lets create the parent component, `PeopleController`. Open the PeopleController.html file and create its template. Also import both the components in it using HTML imports.

<i>Note: HTML imports have been depracated from the standard and are expected to be replaced by module imports. For the purpose of this tutorial we'll use HTML imports only. You can read more on that at [MDN Blog](https://hacks.mozilla.org/2015/06/the-state-of-web-components/) and use them accordingly.</i>

```html
<template id="people-controller-template">
  <link rel="stylesheet" href="/components/PeopleController/PeopleController.css">
  <people-list id="people-list"></people-list>
  <person-detail id="person-detail"></person-detail>
</template>
<link rel="import" href="/components/PeopleController/PeopleList/PeopleList.html">
<link rel="import" href="/components/PeopleController/PersonDetail/PersonDetail.html">
<script src="/components/PeopleController/PeopleController.js"></script>
```

Open the PeopleController.css file and add the following code to it:

```css
#people-list {
  width: 45%;
  display: inline-block;
}
#person-detail {
  width: 45%;
  display: inline-block;
}
```

Open the `PeopleController.js` file and create the `PeopleController` class. We will call the API to get the data of users. This will take 2 components we defined earlier, populate the PeopleList component as well as provide the first user of this data as the initial data to the PeopleDetail component. 

```js
(function () {
  const currentDocument = document.currentScript.ownerDocument;

  class PeopleController extends HTMLElement {
    constructor() {
      super();
      this.peopleList = [];
    }

    connectedCallback() {
      const shadowRoot = this.attachShadow({ mode: 'open' });
      const template = currentDocument.querySelector('#people-controller-template');
      const instance = template.content.cloneNode(true);
      shadowRoot.appendChild(instance);

      _fetchAndPopulateData(this);
    }
  }

  function _fetchAndPopulateData(self) {
    let peopleList = self.shadowRoot.querySelector('#people-list');
    fetch(`https://jsonplaceholder.typicode.com/users`)
      .then((response) => response.text())
      .then((responseText) => {
        const list = JSON.parse(responseText);
        self.peopleList = list;
        peopleList.list = list;

        _attachEventListener(self);
      })
      .catch((error) => {
        console.error(error);
      });
  }

  function _attachEventListener(self) {
    let personDetail = self.shadowRoot.querySelector('#person-detail');

    //Initialize with person with id 1:
    personDetail.updatePersonDetails(self.peopleList[0]);

    // Watch for the event on the shadow DOM
    self.shadowRoot.addEventListener('PersonClicked', (e) => {
      // e contains the id of person that was clicked.
      // We'll find him using this id in the self.people list:
      self.peopleList.forEach(person => {
        if (person.id == e.detail.personId) {
          // Update the personDetail component to reflect the click
          personDetail.updatePersonDetails(person);
        }
      })
    })
  }

  customElements.define('people-controller', PeopleController);
})()
```

## Using the component
Now that we have our components in place, we can use it in our projects. So for the sake of this tutorial, create a new HTML file called index.html and write the following code in it:

```html
<html>

<head>
  <title>Web Component Part 2</title>
</head>

<body>
  <people-controller></people-controller>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/webcomponentsjs/1.0.14/webcomponents-hi.js"></script>
  <link rel="import" href="./components/PeopleController/PeopleController.html">
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

You can check out the [github repository](https://github.com/ayushgp/web-components-tutorial) I've created to go with these tutorials. Let me know what you think about this approach to use web components and any improvements I can make to either this post or the method I have described here.
