# 0x04. Files Manager

**Back-end | JavaScript | ES6 | NoSQL | MongoDB | Redis | NodeJS | ExpressJS | Kue**

This project is a summary of this back-end trimester: authentication, NodeJS, MongoDB, Redis, pagination, and background processing.

The objective is to build a simple platform to upload and view files:

- User authentication via a token
- List all files
- Upload a new file
- Change permission of a file
- View a file
- Generate thumbnails for images

You will be guided step by step for building it, but you have some freedoms of implementation, split in more files etc… (utils folder will be your friend)

Of course, this kind of service already exists in the real life - it’s a learning purpose to assemble each piece and build a full product.

Enjoy!

## Resources

Read or watch:

- [Node JS getting started](https://nodejs.org/en/docs/guides/getting-started-guide/)
- [Process API doc](https://nodejs.org/api/process.html)
- [Express getting started](https://expressjs.com/en/starter/installing.html)
- [Mocha documentation](https://mochajs.org/)
- [Nodemon documentation](https://nodemon.io/)
- [MongoDB](https://docs.mongodb.com/)
- [Bull](https://github.com/OptimalBits/bull)
- [Image thumbnail](https://www.npmjs.com/package/image-thumbnail)
- [Mime-Types](https://www.npmjs.com/package/mime-types)
- [Redis](https://redis.io/documentation)

## Learning Objectives

At the end of this project, you are expected to be able to explain to anyone, without the help of Google:

- How to create an API with Express
- How to authenticate a user
- How to store data in MongoDB
- How to store temporary data in Redis
- How to setup and use a background worker

## Requirements

- Allowed editors: vi, vim, emacs, Visual Studio Code
- All your files will be interpreted/compiled on Ubuntu 18.04 LTS using node (version 12.x.x)
- All your files should end with a new line
- A `README.md` file, at the root of the folder of the project, is mandatory
- Your code should use the `.js` extension
- Your code will be verified against lint using ESLint

## Provided files

- `package.json`
- `.eslintrc.js`
- `babel.config.js`

And…

Don’t forget to run `$ npm install` when you have the `package.json`.

## Tasks

### 0. Redis utils
**mandatory**

Inside the folder `utils`, create a file `redis.js` that contains the class `RedisClient`.

`RedisClient` should have:

- The constructor that creates a client to Redis:
  - Any error of the redis client must be displayed in the console (you should use `on('error')` of the redis client)
- A function `isAlive` that returns `true` when the connection to Redis is a success otherwise, `false`
- An asynchronous function `get` that takes a string `key` as argument and returns the Redis value stored for this key
- An asynchronous function `set` that takes a string `key`, a `value` and a `duration` in second as arguments to store it in Redis (with an expiration set by the `duration` argument)
- An asynchronous function `del` that takes a string `key` as argument and remove the value in Redis for this key

After the class definition, create and export an instance of `RedisClient` called `redisClient`.

```javascript
bob@dylan:~$ cat main.js
import redisClient from './utils/redis';

(async () => {
    console.log(redisClient.isAlive());
    console.log(await redisClient.get('myKey'));
    await redisClient.set('myKey', 12, 5);
    console.log(await redisClient.get('myKey'));

    setTimeout(async () => {
        console.log(await redisClient.get('myKey'));
    }, 1000 * 10)
})();
```

```bash
bob@dylan:~$ npm run dev main.js
true
null
12
null
bob@dylan:~$
```

**Repo:**

- GitHub repository: `alx-files_manager`
- File: `utils/redis.js`

### 1. MongoDB utils
**mandatory**

Inside the folder `utils`, create a file `db.js` that contains the class `DBClient`.

`DBClient` should have:

- The constructor that creates a client to MongoDB:
  - `host`: from the environment variable `DB_HOST` or default: `localhost`
  - `port`: from the environment variable `DB_PORT` or default: `27017`
  - `database`: from the environment variable `DB_DATABASE` or default: `files_manager`
- A function `isAlive` that returns `true` when the connection to MongoDB is a success otherwise, `false`
- An asynchronous function `nbUsers` that returns the number of documents in the collection `users`
- An asynchronous function `nbFiles` that returns the number of documents in the collection `files`

After the class definition, create and export an instance of `DBClient` called `dbClient`.

```javascript
bob@dylan:~$ cat main.js
import dbClient from './utils/db';

const waitConnection = () => {
    return new Promise((resolve, reject) => {
        let i = 0;
        const repeatFct = async () => {
            await setTimeout(() => {
                i += 1;
                if (i >= 10) {
                    reject()
                } else if (!dbClient.isAlive()) {
                    repeatFct()
                } else {
                    resolve()
                }
            }, 1000);
        };
        repeatFct();
    })
};

(async () => {
    console.log(dbClient.isAlive());
    await waitConnection();
    console.log(dbClient.isAlive());
    console.log(await dbClient.nbUsers());
    console.log(await dbClient.nbFiles());
})();
```

```bash
bob@dylan:~$ npm run dev main.js
false
true
4
30
bob@dylan:~$
```

**Repo:**

- GitHub repository: `alx-files_manager`
- File: `utils/db.js`

### 2. First API
**mandatory**

Inside `server.js`, create the Express server:

- It should listen on the port set by the environment variable `PORT` or by default `5000`
- It should load all routes from the file `routes/index.js`

Inside the folder `routes`, create a file `index.js` that contains all endpoints of our API:

- `GET /status` => `AppController.getStatus`
- `GET /stats` => `AppController.getStats`

Inside the folder `controllers`, create a file `AppController.js` that contains the definition of the 2 endpoints:

- `GET /status` should return if Redis is alive and if the DB is alive too by using the 2 utils created previously: `{ "redis": true, "db": true }` with a status code 200
- `GET /stats` should return the number of users and files in DB: `{ "users": 12, "files": 1231 }` with a status code 200
  - `users` collection must be used for counting all users
  - `files` collection must be used for counting all files

Terminal 1:

```bash
bob@dylan:~$ npm run start-server
Server running on port 5000
...
```

Terminal 2:

```bash
bob@dylan:~$ curl 0.0.0.0:5000/status ; echo ""
{"redis":true,"db":true}
bob@dylan:~$
bob@dylan:~$ curl 0.0.0.0:5000/stats ; echo ""
{"users":4,"files":30}
bob@dylan:~$
```

**Repo:**

- GitHub repository: `alx-files_manager`
- File: `server.js`, `routes/index.js`, `controllers/AppController.js`

### 3. Create a new user
**mandatory**

Now that we have a simple API, it’s time to add users to our database.

In the file `routes/index.js`, add a new endpoint:

- `POST /users` => `UsersController.postNew`

Inside `controllers`, add a file `UsersController.js` that contains the new endpoint:

`POST /users` should create a new user in DB:

- To create a user, you must specify an email and a password
  - If the email is missing, return an error `Missing email` with a status code 400
  - If the password is missing, return an error `Missing password` with a status code 400
  - If the email already exists in DB, return an error `Already exist` with a status code 400
  - The password must be stored after being hashed in SHA1
  - The endpoint is returning the new user with only the email and the id (auto generated by MongoDB) with a status code 201
  - The new user must be saved in the collection `users`:
    - `email`: same as the value received
    - `password`: SHA1 value of the value received

```bash
bob@dylan:~$ curl 0.0.0.0:5000/users -XPOST -H "Content-Type: application/json" -d '{ "email": "bob@dylan.com", "password

": "toto1234" }' ; echo ""
{"id":"5f1e7d35c7ba06511e683b21","email":"bob@dylan.com"}
bob@dylan:~$
bob@dylan:~$ curl 0.0.0.0:5000/users -XPOST -H "Content-Type: application/json" -d '{ "email": "bob@dylan.com", "password": "toto1234" }' ; echo ""
{"error":"Already exist"}
bob@dylan:~$
```

**Repo:**

- GitHub repository: `alx-files_manager`
- File: `controllers/UsersController.js`, `routes/index.js`

### 4. Authenticate a user
**mandatory**

In the file `routes/index.js`, add 2 new endpoints:

- `GET /connect` => `AuthController.getConnect`
- `GET /disconnect` => `AuthController.getDisconnect`

Inside `controllers`, add a file `AuthController.js` that contains the new endpoints:

`GET /connect` should sign-in the user by generating a new authentication token:

- By using the header `Authorization` with a base64 value of the string `email:password` (like `Authorization: Basic email:password`)
  - If the user is not found, return an error `Unauthorized` with a status code 401
  - Otherwise:
    - Generate a random string (using `uuid` npm package) as token
    - Create a key: `auth_${token}`
    - Use this key for storing in Redis (by using the `redisClient` create previously) the user ID for 24 hours
    - Return this token: `{ "token": "155342df-2399-41da-9e8c-458b6ac52a0c" }`

`GET /disconnect` should sign-out the user based on the token:

- Retrieve the user based on the token:
  - If not found, return an error `Unauthorized` with a status code 401
  - Otherwise, delete the token in Redis and return nothing with a status code 204

```bash
bob@dylan:~$ curl -v 0.0.0.0:5000/connect -u "bob@dylan.com:toto1234"
Note: Unnecessary use of -X or --request, GET is already inferred.
*   Trying 0.0.0.0:5000...
* Connected to 0.0.0.0 (0.0.0.0) port 5000 (#0)
* Server auth using Basic with user 'bob@dylan.com'
> GET /connect HTTP/1.1
> Host: 0.0.0.0:5000
> Authorization: Basic Ym9iQGR5bGFuLmNvbTp0b3RvMTIzNA==
> User-Agent: curl/7.64.1
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< X-Powered-By: Express
< Content-Type: application/json; charset=utf-8
< Content-Length: 53
< ETag: W/"35-pFwLiFqqtjkAvWBO+xhly8CKzXI"
< Date: Fri, 24 Jul 2020 02:26:13 GMT
< Connection: keep-alive
<
* Connection #0 to host 0.0.0.0 left intact
{"token":"031ed663-3ed5-4fda-8820-74fd173bd0a4"}
bob@dylan:~$
bob@dylan:~$ curl -XGET -H "X-Token: 031ed663-3ed5-4fda-8820-74fd173bd0a4" 0.0.0.0:5000/disconnect ; echo ""
bob@dylan:~$
bob@dylan:~$ curl -XGET -H "X-Token: 031ed663-3ed5-4fda-8820-74fd173bd0a4" 0.0.0.0:5000/users/me ; echo ""
{"error":"Unauthorized"}
bob@dylan:~$
```

**Repo:**

- GitHub repository: `alx-files_manager`
- File: `controllers/AuthController.js`, `routes/index.js`

### 5. Get user
**mandatory**

In the file `routes/index.js`, add a new endpoint:

- `GET /users/me` => `UsersController.getMe`

In the file `UsersController.js`, add the new endpoint:

`GET /users/me` should retrieve the user base on the token:

- Retrieve the user based on the token:
  - If not found, return an error `Unauthorized` with a status code 401
  - Otherwise, return the user object (email and id only)

```bash
bob@dylan:~$ curl -v 0.0.0.0:5000/connect -u "bob@dylan.com:toto1234"
Note: Unnecessary use of -X or --request, GET is already inferred.
*   Trying 0.0.0.0:5000...
* Connected to 0.0.0.0 (0.0.0.0) port 5000 (#0)
* Server auth using Basic with user 'bob@dylan.com'
> GET /connect HTTP/1.1
> Host: 0.0.0.0:5000
> Authorization: Basic Ym9iQGR5bGFuLmNvbTp0b3RvMTIzNA==
> User-Agent: curl/7.64.1
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< X-Powered-By: Express
< Content-Type: application/json; charset=utf-8
< Content-Length: 53
< ETag: W/"35-pFwLiFqqtjkAvWBO+xhly8CKzXI"
< Date: Fri, 24 Jul 2020 02:26:13 GMT
< Connection: keep-alive
<
* Connection #0 to host 0.0.0.0 left intact
{"token":"031ed663-3ed5-4fda-8820-74fd173bd0a4"}
bob@dylan:~$
bob@dylan:~$ curl -XGET -H "X-Token: 031ed663-3ed5-4fda-8820-74fd173bd0a4" 0.0.0.0:5000/users/me ; echo ""
{"id":"5f1e7d35c7ba06511e683b21","email":"bob@dylan.com"}
bob@dylan:~$
```

**Repo:**

- GitHub repository: `alx-files_manager`
- File: `controllers/UsersController.js`, `routes/index.js`

### 6. Create new file
**mandatory**

In the file `routes/index.js`, add a new endpoint:

- `POST /files` => `FilesController.postUpload`

Inside `controllers`, add a file `FilesController.js` that contains the new endpoint:

`POST /files` should create a new file in DB and in disk:

- Retrieve the user based on the token:
  - If not found, return an error `Unauthorized` with a status code 401
- To create a file, you must specify:
  - `name`: as filename
  - `type`: either `folder`, `file` or `image`
  - `parentId`: (optional) as ID of the parent (default: 0 -> the root)
  - `isPublic`: (optional) as boolean to define if the file is public or not
  - `data`: (only for `type` file and image) as Base64 content of the file
  - If the `name` is missing, return an error `Missing name` with a status code 400
  - If the `type` is missing or not part of the list, return an error `Missing type` with a status code 400
  - If the `data` is missing and `type` is `file` or `image`, return an error `Missing data` with a status code 400
  - If the `parentId` is set:
    - If no document is present in DB with this ID, return an error `Parent not found` with a status code 400
    - If the file present in DB with this ID is not of type `folder`, return an error `Parent is not a folder` with a status code 400
  - The user ID should be added to the document saved in DB - as `userId`
  - The filename and the file type should be saved in the attribute `name` and `type` - same as the value received
  - If `isPublic` is not present, set as `false` - same as the value received
  - If the type is `folder`, add the new document in the collection `files` and return the new file with a status code 201
  - Otherwise:
    - All file must be stored in a local directory `~/files_manager`:
      - If the directory doesn’t exist, it should be created
      - The file should be named with an ID unique for this project
      - Add the new document in the collection `files`
      - Return the new file with a status code 201

```bash
bob@

dylan:~$ curl -XPOST -H "X-Token: 031ed663-3ed5-4fda-8820-74fd173bd0a4" -H "Content-Type: application/json" -d '{ "name": "myText.txt", "type": "file", "data": "SGVsbG8gd29ybGQh\n" }' 0.0.0.0:5000/files ; echo ""
{"id":"5f1e7cda04a394508232559d","userId":"5f1e7d35c7ba06511e683b21","name":"myText.txt","type":"file","isPublic":false,"parentId":0}
bob@dylan:~$
bob@dylan:~$ head -c 12 /root/files_manager/5f1e7cda04a394508232559d | base64
SGVsbG8gd29ybGQh
bob@dylan:~$
```

**Repo:**

- GitHub repository: `alx-files_manager`
- File: `controllers/FilesController.js`, `routes/index.js`

### 7. List all files
**mandatory**

In the file `routes/index.js`, add a new endpoint:

- `GET /files` => `FilesController.getIndex`

In the file `FilesController.js`, add the new endpoint:

`GET /files` should retrieve all users file documents for a specific `parentId` and with pagination:

- Retrieve the user based on the token:
  - If not found, return an error `Unauthorized` with a status code 401
- Based on the `parentId` and the `page`, return all files:
  - `parentId`:
    - No `parentId` should return the root files
    - `parentId=0` should return the root files
    - No `parentId` and `parentId=0` are the same
    - If the `parentId` is not a folder or not linked to the user, return an empty list
  - `page`:
    - Page is used for pagination
    - If `page` is not present, page is set to 0
    - Pagination must be 20 items per page
    - Pagination can be done directly by the aggregation `skip` and `limit` in MongoDB
    - Return the list of files in a specific folder in the DB and not sub folders

```bash
bob@dylan:~$ curl -XGET -H "X-Token: 031ed663-3ed5-4fda-8820-74fd173bd0a4" 0.0.0.0:5000/files | jq .
[
  {
    "id": "5f1e7cda04a394508232559d",
    "userId": "5f1e7d35c7ba06511e683b21",
    "name": "myText.txt",
    "type": "file",
    "isPublic": false,
    "parentId": 0
  }
]
bob@dylan:~$
bob@dylan:~$ curl -XGET -H "X-Token: 031ed663-3ed5-4fda-8820-74fd173bd0a4" 0.0.0.0:5000/files?page=0 | jq .
[
  {
    "id": "5f1e7cda04a394508232559d",
    "userId": "5f1e7d35c7ba06511e683b21",
    "name": "myText.txt",
    "type": "file",
    "isPublic": false,
    "parentId": 0
  }
]
bob@dylan:~$
bob@dylan:~$ curl -XGET -H "X-Token: 031ed663-3ed5-4fda-8820-74fd173bd0a4" 0.0.0.0:5000/files?page=1 | jq .
[]
bob@dylan:~$
```

**Repo:**

- GitHub repository: `alx-files_manager`
- File: `controllers/FilesController.js`, `routes/index.js`

### 8. Get a file
**mandatory**

In the file `routes/index.js`, add a new endpoint:

- `GET /files/:id` => `FilesController.getShow`

In the file `FilesController.js`, add the new endpoint:

`GET /files/:id` should retrieve the file document based on the ID:

- Retrieve the user based on the token:
  - If not found, return an error `Unauthorized` with a status code 401
- Retrieve the user based on the ID:
  - If not found, return an error `Not found` with a status code 404
  - If the file document is not linked to the user, return an error `Not found` with a status code 404
  - Otherwise, return the file document

```bash
bob@dylan:~$ curl -XGET -H "X-Token: 031ed663-3ed5-4fda-8820-74fd173bd0a4" 0.0.0.0:5000/files/5f1e7cda04a394508232559d | jq .
{
  "id": "5f1e7cda04a394508232559d",
  "userId": "5f1e7d35c7ba06511e683b21",
  "name": "myText.txt",
  "type": "file",
  "isPublic": false,
  "parentId": 0
}
bob@dylan:~$
```

**Repo:**

- GitHub repository: `alx-files_manager`
- File: `controllers/FilesController.js`, `routes/index.js`

### 9. Update file
**mandatory**

In the file `routes/index.js`, add a new endpoint:

- `PUT /files/:id/publish` => `FilesController.putPublish`
- `PUT /files/:id/unpublish` => `FilesController.putUnpublish`

In the file `FilesController.js`, add the new endpoint:

`PUT /files/:id/publish` and `PUT /files/:id/unpublish` should update the `isPublic` attribute to `true` or `false`:

- Retrieve the user based on the token:
  - If not found, return an error `Unauthorized` with a status code 401
- Retrieve the file document based on the ID:
  - If not found, return an error `Not found` with a status code 404
  - If the file document is not linked to the user, return an error `Not found` with a status code 404
  - Otherwise:
    - Update the value `isPublic` to `true` if `/publish`
    - Update the value `isPublic` to `false` if `/unpublish`
    - And return the file document with the new value `isPublic`

```bash
bob@dylan:~$ curl -XPUT -H "X-Token: 031ed663-3ed5-4fda-8820-74fd173bd0a4" 0.0.0.0:5000/files/5f1e7cda04a394508232559d/publish | jq .
{
  "id": "5f1e7cda04a394508232559d",
  "userId": "5f1e7d35c7ba06511e683b21",
  "name": "myText.txt",
  "type": "file",
  "isPublic": true,
  "parentId": 0
}
bob@dylan:~$
bob@dylan:~$ curl -XPUT -H "X-Token: 031ed663-3ed5-4fda-8820-74fd173bd0a4" 0.0.0.0:5000/files/5f1e7cda04a394508232559d/unpublish | jq .
{
  "id": "5f1e7cda04a394508232559d",
  "userId": "5f1e7d35c7ba06511e683b21",
  "name": "myText.txt",
  "type": "file",
  "isPublic": false,
  "parentId": 0
}
bob@dylan:~$
```

**Repo:**

- GitHub repository: `alx-files_manager`
- File: `controllers/FilesController.js`, `routes/index.js`

### 10. Get file data
**mandatory**

In the file `routes/index.js`, add a new endpoint:

- `GET /files/:id/data` => `FilesController.getFile`

In the file `FilesController.js`, add the new endpoint:

`GET /files/:id/data` should return the content of the file document based on the ID:

- Retrieve the user based on the token:
  - If not found, return an error `Unauthorized` with a status code 401
- Retrieve the file document based on the ID:
  - If not found, return an error `Not found` with a status code 404
  - If the file document is not linked to the user and is not public, return an error `Not found` with a status code 404
  - If the type of the file document is `folder`, return an error `A

 folder doesn't have content` with a status code 400
  - If the file is not locally present, return an error `Not found` with a status code 404
  - Otherwise:
    - Retrieve the content of the file in the local storage (using `fs`), and return it
    - If the user agent is a browser, display the file in the browser with the correct type (for example `image/png`)

```bash
bob@dylan:~$ curl -XGET -H "X-Token: 031ed663-3ed5-4fda-8820-74fd173bd0a4" 0.0.0.0:5000/files/5f1e7cda04a394508232559d/data
Hello world!
bob@dylan:~$
```

**Repo:**

- GitHub repository: `alx-files_manager`
- File: `controllers/FilesController.js`, `routes/index.js`

## Author
**Teddy Omondi**
* [GitHub](https://github.com/TeddyO323)