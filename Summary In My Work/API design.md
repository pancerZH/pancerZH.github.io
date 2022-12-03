---
sort: 3
---



# How to design high quality APIs

This note begins from this story: one day one of a micro service I participated in developing has undergone a check by our arch-leader, and he was not so satisficed with several of our API designs. And he offered us a book and told us to follow the items and suggestions in it. To be honest, when I was in college, I was never educated how to design a good API, and everyone was done with a meaningful name of the API. All other parts were left for personal preference and implementations. This book, *Web API Design* by **Brian Mulley**, really changed a lot of my stereotypes. So I made some notes and understandings of this book here.

### Nouns are good, verbs are bad

Here are some simple rules about API names:

1. Keep the base URL simple and intuitive 
2. Keep verbs out of the base URLs
3. Use HTTP verbs to operate on collections and elements

Generally speaking, most scenarios can be as simple as **CRUD**: create something, query something, modify something and delete something. In this way, We can name the base URL the name of the resource, and use some verbs like POST, GET, PUT and DELETE to operate the resources:

| Resource  | POST (create)    | GET (read) | PUT (update)     | DELETE (delete) |
| --------- | ---------------- | ---------- | ---------------- | --------------- |
| /dogs     | Create a new dog | List dogs  | Bulk update dogs | Delete all dogs |
| /dogs/123 | Error            | Show a dog | Update a dog     | Delete a dog    |

In its origin sense, noun URL and HTTP verbs corporates very well to represent all the operations to the resource, so that it is unnecessary to use verbs in URLs.

### Plural nouns and concrete names

Here are also some simple rules:

1. Concrete names are better than abstract 

Remember that we would operate concrete resources, and of course the concrete names should be preferred. The key is: Keep the name of URLs as close to the certain resource as possible.

### Simplify association - sweep complexity under the '?'

For two conditions:

1. For simple relationships between resources, we can design it like `/resource/identifier/resource`
2. With many other attributes, we can put them behind the '?', things would look like this: `GET /dogs?color=red&state=running&location=park`

This is how we should deal with associations and parameters.

### Handling errors

We should make use of HTTP status codes to deal with error. There are lots of HTTP status codes, but only some of them are most commonly used:

1. **200** - OK - Everything is OK
2. **400** - Bad Request - The application did something wrong
3. **500** - Internal Server Error - The API did something wrong
4. 201 - Created
5. 304 - Not Modified
6. 404 - Not Found
7. 401 - Unauthorized
8. 403 - Forbidden

Also, we should leave some messages for human users to help to understand what happens.

### Versioning

It is a good habit to leave the version information in the API, which makes it look like this:

> /v1/dogs

In my working experience with some open source projects, a standard API design always include the version information. It helps developers and users know which set of APIs they are using, and it would help them refer to the correct documents once needed. Also, it would be good for API developers themselves to organize different versions of APIs.

### Pagination and partial response

#### Partial response

Partial response allows servers to pass only the resources the client requests, which would save bandwidth and other resources. But I have to say I have never seen such a design in out products and services. Perhaps it is not so critical about the resources saved.

It is not hard to implement it: Just add some optional fields in a comma-delimited list.

> /dogs/?fields=name,color,location

It means we only want the name, color and location fields of the dog we request.

#### Pagination

I bet every developer is familiar with this kind of requirements: pagination. Generally speaking, two parameters are required: page and pageSize (this is slightly different from what the author wrote in the book: it is what I did in work. I guess names of the two parameters does not matter at all). The URL would like this:

> /dogs?page=1&pageSize=25

We had better record some meta data in the metadata field of the response, like how many pages how many items in total. In my work, these information can be easily fetched from database if a pagination query request is generated and sent to the database. It is not necessary to query and compute these information manually.

Last but not least, we should make a default value for both page and pageSize, which could be determined by personal preference, but should be reasonable values. 100000&1000000 are obviously bad ideas!

### Attribute names

Just as described, I used JSON as my APIs' response format. But the name of fields are different: Though in our programs, both frontend and backend, the attribute names follow **camelCase** rule; But in the transporting process, we convert them into **snake_case** format at backend and recover them to camelCase format at frontend. I have to say, it could be a convention for a group or some developers, and document it clearly, and then it should be enough.

### Search

For search, which I am not so familiar with, could be designed according to scenarios:

- For global search, make the URL like this: 

  > /search?q=keyword

- For scoped search:

  > /owner/1234/dogs?q=keyword

Also, I have seen some URLs look like this:

> /scope?q=names:name1,name2&q=plans:plan1,plan2

Or, make the parameter part different format:

> /scope?names=name1,name2&plans:plan1,plans

Fun fact: I just met these two types of querying in two different versions (v2 and v3) of one project.

### Consolidate API requests in one subdomain

It would make things hard by distributing APIs of one scope into different domains. So keep them in one. For my work, our APIs are distributed in different micro services, but in each micro service, the APIs are collected and put under one common domain name, generally the name of the micro service. This action helps developers belonging to different micro services and invoke others' APIs. Also, when we make a release, we also give a list of public APIs in one subdomain, which is the name of our product.

### Authentication

**OAuth 2.0**, nothing more to say, next.