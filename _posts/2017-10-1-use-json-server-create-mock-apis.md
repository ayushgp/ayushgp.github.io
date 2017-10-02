---
layout: post
title: How to Use json-server to Create Mock APIs
future: true
---

So you have an awesome idea for a mobile/web app and want to start working on its front end right away. But you need to create an API for it first because without it, you wouldn’t be able to proceed with the app.

[json-server](https://github.com/typicode/json-server) is an open source mock API tool solves this problem for you. It allows you to create an API with a database within minutes with bare minimum setup. Creating mock CRUD APIs with JSON-server requires zero-coding, you just need to write a configuration file for it.

You should have basic knowledge of [RESTful principles](http://www.restapitutorial.com/) and [how to consume APIs](https://www.codementor.io/restful/tutorial/rest-api-design-best-practices-strategy).

## Installation
You need the following tools:

-  [Node.js](https://nodejs.org/en/): json-server is built on top of Node.js (Read this tutorial on [how to use JSON files in Node.js](https://www.codementor.io/nodejs/tutorial/how-to-use-json-files-in-node-js)).
-  [npm](https://www.npmjs.com): Package manager for Node.js.
-  [cURL](https://curl.haxx.se/): A utility to test the routes of your server. 
-  *Optional*: You can also use [Postman](https://www.getpostman.com/) to test the routes of the server as well. 

### Windows
Setting up cURL on windows is a little tricky, this [Stack Overflow answer](http://stackoverflow.com/a/16216825) will help you set is up.

To install json-server, open the command line and enter the following command: 
```bash
$ npm install -g json-server
```
The `-g` flag installs json-server globally on your system. This allows you to run your server from any directory you like.

## Resources
### What is a resource?
Any information that can be named can be a resource. For example, if you are working on a books review website, books, users, reviews, etc. would be resources.

API endpoints are named after these resources. We use these endpoints to retrieve/update data on our server.
### Creating a Resource
json-server works in a JSON file. It acts as both a config and database file for our mock server. Create a file called `Database.json` and add the following content to it:
```
{
	"books": [
		{
			"id": 101, 
			"title": "Zero to One", 
			"author":"Peter Thiel", 
			"year_published": 2014,
			"rating": 4.03
		},
		{
			"id": 102, 
			"title": "The Origin of Species", 
			"author": "Charles Darwin", 
			"year_published": 1889,
			"rating": 4.20
		}
	]
}
```
Save this file and run your server using:
```bash
$ json-server --watch Database.json
```
Now you have a working books API. You can perform all CRUD operations on the resource "book". 

To test this server, open a new terminal and enter:
```bash
$ curl -X GET "http://localhost:3000/books"
```
This will return a detailed list of all the books in our database. We can also retrieve books individually by specifying its id at the end of the URI, like: http://localhost:3000/books/101.

We used the GET HTTP verb to retrieve the book details. To insert a book to our database, we need to send the data using a POST request. For example, 
```bash
$ curl -X POST -H "Content-Type: application/json" -d '{
	"id": 103,
	"title": "The Discovery of India",
	"author": "Jawaharlal Nehru",
	"year_published": 1946
	"rating": 3.8
}' "http://localhost:3000/books"
```
We need to post this data without specifying an ID in the URI, but add it in the data. To check if this was successfully added, send a GET request to the server:
```bash
$ curl -X GET "http://localhost:3000/books/103"
```
You can use other HTTP verbs like PUT, DELETE, etc. to access and modify data on this server. Note that PUT, POST, and PATCH requests need to have a `Content-Type: application/json` header set.
### Features
#### 1. Search
You can search within a resource for relevant results. For example, in the books API, if you want to search for the word "Discovery", then you need to add an optional parameter `q` to your URI:
```bash
$ curl -X GET "http://localhost:3000/books?q=Discovery"
```
This will return all the books which somewhere in their fields contain the word "Discovery".
#### 2. Filters
You can apply filters to your requests again using the `?` sign. The `q` filter is reserved for search as we've seen above. 

If you want to get the details of books by "Peter Thiel", you can send a GET request to your resource URI appending it with a `?` followed by the property name you want to filter with and its value:  
```bash
$ curl -X GET "http://localhost:3000/books?author=Peter+Thiel"
```
Note that the author name we specified here is URL encoded. 

You can also combine multiple filters by adding an ampersand between different filters. For example, if we also want to filter by the book's name in above example, we could use:
```bash
$ curl -X GET "http://localhost:3000/books?author=Peter+Thiel&title=Zero+to+One"
```
#### 3. Pagination
json-server, by default, provides pagination support with 10 items per page. Pagination comes in handy when prototyping apps that'll have pages or will load data when scrolling.

For example, if you want to access page 3 of your book's API, send a GET request:
```bash
$ curl -X GET "http://localhost:3000/books?_page=3"
```
This will respond with books having IDs 121–130 stored on your server.
#### 4. Sorting
It also allows you to request sorted data from your API. Use the `_sort` and `_order` properties for specifying the property on whose basis you want to sort and the order in which you want to sort it respectively. If you are sorting on a text field, the entries would be sorted alphabetically. 

For example, if you want the list of books to be sorted in descending order of ratings, then you'll send a GET request:
```bash
$ curl -X GET "http://localhost:3000/books?_sort=rating&order=DESC"
```
#### 5. Operators
Operators are essential to further filter out results and only get the required data. json-server also provides you with logical operators. You can use `_gt` and `_lt` as greater than and less than operators respectively. You also have `_ne` for excluding a value from the response.

For example, if you want all books whose ratings are greater than or equal to 4, the you'll send a get request:
```bash
$ curl -X GET "http://localhost:3000/books?rating_gt=4"
```
Note that you can combine multiple operators using the ampersand sign. For example, to get all books which have been published between 1990 and 2016 (both inclusive), you’ll need to send the request:
```bash
$ curl -X GET "http://localhost:3000/books?rating_gte=1990&rating_lte=2016"
```

Note the `gte` and `lte`, these mean greater than or equal to and less than or equal to respectively.

Check out the complete [documentation](https://github.com/typicode/json-server) to learn more about these features.  

## Generating Mock Data for your API
To prototype front ends, you need enough data that you can test for all cases. Typing that data in all by yourself is really boring. You can create some mock data for your mock API using modules like [casual](https://github.com/boo1ean/casual). Install the package using:
```bash
$ npm install casual
```
Now create a file called mockdata.js and enter the following in it:
```js
var casual = require('casual');

// Create an object for config file
var db = {books:[]};

for(var i=101; i<=105; i++){
    var book = {};
	book.id = i;

	// Create a random 1-6 word title
	book.title = casual.words(casual.integer(1,6));
	book.author = casual.first_name + ' ' + casual.last_name;
	
	// Randomly rate the book between 0 and 5
	book.rating = Math.floor(Math.random()*100+1)/20;

	// Assign a publishing year between 1700 and 2016
    book.year_published = casual.integer(1700,2016)
    db.books.push(book);
}
console.log(JSON.stringify(db));
```

To create a `Database.json` file using this script, run the following command in your terminal:
```bash
$ node mockdata.js > Database.json
```
Now you have a database of many books. You can now use this server to prototype your apps.

## Limitations
There are certain obvious limitations to using json-server. Some of these are:

- You can only use it to build prototypes involving textual data.
- You cannot place restrictions on areas of the API based on authentication.
- These mock APIs cannot and should not be used in production.

## Wrapping Up
You will now be able to quickly create your own mock APIs and use them to rapidly prototype your front ends. All this without investing any time (almost) in creating a backend for it.
