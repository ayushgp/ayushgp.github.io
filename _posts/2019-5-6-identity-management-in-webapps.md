---
layout: post
title: Identity Management in web applications
comments: true
---

## Authentication vs Authorization
**tl,dr;** 

> *Authentication is verifying who you are, authorization is verifying if you have permission to do the action you're trying to do.*

In more verbose terms,

Authentication is used by the system to identify who exactly is trying to perform an action.

Authorization is used by the system to check if the identified person/entity is authorized(has permission) to perform a given action. 

For example, let's see we have a portal for students and teachers of a course which allows teachers and students to login to the portal and access course content. In addition, it allows teachers to update the course content as well. 

If a person without credentials tries to access the protected area of the website, he'll be denied access as he is not *authorized* to view that content. Now let's say a student logs in to the system(*authenticates* herself). She first tries to read the course content then tries to change it. The first action will be allowed, since all students are allowed to read the course contents. When she tries to update the page, the system will check if she is a teacher or not because only teachers are authorized to make changes.

## Permissions
In the above example, we saw that the student was authorized to read data from the course website. The teacher was authorized to read and update data on the course website. In other words, they had permissions to do this. 

In identity management systems, permissions are generally denoted in the following fashion: resource:permission

For example, let's take a case of a closed facebook group. Only admins of the group are allowed to create, update or delete posts. All the members of the group can see the posts, comment on and react to the posts. We can sum up the permissions of the Admins as follows:

```
post:create
post:update
post:delete
```

We can list the permissions of the group members as follows:
```
post:react
post:read
post:comment
```

Here, `post` is a resource over which different groups people have different permissions. 

But wait a minute, don't admins also have read, react and comment permissions on the post? Let's talk about that next.

## Roles
Who are student and teacher in the first example? Who is a member? Who is a admin?

They are all roles a person can take in an identity management system. 

> A role is the function assumed or part played by a person or thing in a particular situation.

Sooooo, can people have multiple roles? Absolutely.

In our second example, the admins of the facebook groups can take on 2 roles, being a member or being an admin of the group. They will have a *union* of all the permissions of both the roles. 

These 3 ideas are enough to understand most identity management systems. You can read more about these on [Wikipedia](https://en.wikipedia.org/wiki/Role-based_access_control).

Note that roles and groups are different things, do not confuse roles with groups that are found on most operating systems identity management systems. You can read more about those differences on [StackOverflow](https://stackoverflow.com/questions/7770728/group-vs-role-any-real-difference)

## Using Identity Management As A Service
There are a lot of services out there that take care of managing your users for you. Examples of such services are auth0, AWS Cognito, Azure active directory, etc. 

All these service providers use the above concepts to model users in their systems and allow you to skip reinventing the wheel. They are a little costly IMO, but if you have a small user base they can do the auth related heavy lifting for you while you focus on building your product. 

These service providers also allow integrations with social profile logins to make registrations easy for your customers.
