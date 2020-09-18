---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "REST API Using Python Flask"
subtitle: ""
summary: ""
authors: [manish-gupta]
tags: [python, flask, REST]
categories: [Flask]
date: 2020-09-17T18:42:33-05:00
lastmod: 2020-09-17T18:42:33-05:00
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---
When I started to work on the Keyper project, I decided to use Java/Struts to develop its backend REST API. Years ago, I did some programming in Java with Struts 1. However, soon I realized that Struts and many associated libraries have changed and would pretty much require re-learning. I still chugged along and developed the first REST API. My goal was to bundle everything in a docker image taking less than 100MB. During testing, I realized that with java/struts/tomcat it was not possible. So, I started to look for an alternative.

## What is an API
But first, what is REST API? Per Wikipedia, "Representational state transfer (REST) is a software architectural style that defines a set of constraints to be used for creating Web services. Web services that conform to the REST architectural style, called RESTful Web services, provide interoperability between computer systems on the internet. RESTful Web services allow the requesting systems to access and manipulate textual representations of Web resources by using a uniform and predefined set of stateless operations. Other kinds of Web services, such as SOAP Web services, expose their own arbitrary sets of operations."

HTTP is one of the most common protocols used for REST API. Four main methods that are used when making requests with REST API are: GET, POST, PUT, and DELETE. Any REST API request consists of the following:

1. The Url and endpoint
2. The method (GET, POST, PUT, DELETE)
3. The headers
4. The Data

The endpoint is the URL you send a request. It typically consists of a domain, a directory hierarchy, and the resource. e.g.:

```python
https://sprouts.dbsentry.com/api/users
```

The methods are GET, POST, PUT, and DELETE. These are self-explanatory. Using these methods data can be retrieved (GET), added (POST), updated (PUT), and deleted (DELETE) from the backend.  

The headers typically contain some header variable for session or token (e.g. JWT tokens).

Finally, the data is what gets queried. JSON is one of the most popular data formats API endpoints returns data. e.g.

```json
GET https://sprout.dbsentry.com/api/users

[
    {
        "accountLocked": false,
        "cn": "alice",
        "displayName": "Alice Parker",
        "dn": "cn=alice,ou=people,dc=dbsentry,dc=com",
        "givenName": "Alice",
        "mail": "alice@dbsentry.com",
        "memberOfs": [
            "cn=Admins,ou=groups,dc=dbsentry,dc=com"
        ],
        "sn": "Parker",
        "uid": "Alice"
    },
    {
        "accountLocked": false,
        "cn": "bob",
        "displayName": "Bob Parker",
        "dn": "cn=bob,ou=people,dc=dbsentry,dc=com",
        "givenName": "Bob",
        "mail": "bob@dbsentry.com",
        "memberOfs": [
            "cn=getafix,ou=groups,dc=dbsentry,dc=com"
        ],
        "sn": "Parker",
        "sshPublicKeys": [],
        "uid": "bob"
    },
]
```
## API with Flask
In any case, while looking for an alternative to develop REST API, I stumbled upon Python Flask. Having written a ton of shell/Perl scripts in the past, I found myself at home with Python Flask. I found it very easy to understand and yet powerful. I was able to create my first API in less than a day. 

A typical flask application has following components:
1. Libraries
2. Application Configuration
3. Functions
4. Routing

Keyper API source has following structure:

```console
app
├── __init__.py
├── admin
│   ├── __init__.py
│   ├── auth.py
│   ├── groups.py
│   ├── hosts.py
│   └── users.py
├── public
│   ├── __init__.py
│   └── authkeys.py
├── resources
│   ├── __init__.py
│   └── errors.py
└── utils
    ├── __init__.py
    ├── extensions.py
    ├── flask_logs.py
    └── operations.py
```

**__init__.py** under **app** is where we configure the application. Something like this:

```python
''' Keyper API app '''
from flask import Flask, jsonify
from config import config

def create_app():
    ''' Create app '''
    app = Flask(__name__)

    flask_config = "config." + config.get(environ.get('FLASK_CONFIG'), 'ProductionConfig')

    app.config.from_object(flask_config)

    ...
    ...

    return app
```

Routes and functions are defined for each resource is defined something like this:

```python
''' REST API for users '''
import ldap
import ldap.modlist as modlist
from time import strftime, gmtime
from flask import request, jsonify
from flask import current_app as app
from . import admin
from ..resources.errors import KeyperError, errors
from ..utils import operations

@route('/users', methods=['GET'])
def get_users():
    ''' List All Users '''
    app.logger.debug("Enter")
    con = operations.open_ldap_connection()
    result = search_users(con, '(objectClass=*)')
    operations.close_ldap_connection(con)

    app.logger.debug("Exit")

    return jsonify(result)

@route('/users/<username>', methods=['GET'])
def get_user(username):
    ''' List a User '''
    app.logger.debug("Enter")
    con = operations.open_ldap_connection()
    result = search_users(con, '(&(objectClass=*)(cn=' + username + '))')
    operations.close_ldap_connection(con)
    app.logger.debug("Exit")

    return jsonify(result)

@route('/users', methods=['POST'])
def create_user():
    ''' Create a User '''
    app.logger.debug("Enter")
    req = request.get_json()

    try:
        con = operations.open_ldap_connection()

        # Add req to LDAP and return data in list
        ...
        ...
        operations.close_ldap_connection(con)
    except ldap.ALREADY_EXISTS:
        app.logger.error("LDAP Entry already exists:" + dn)
        raise KeyperError(errors["ObjectExistsError"].get("msg"), errors["ObjectExistsError"].get("status"))
    except ldap.LDAPError:
        exctype, value = sys.exc_info()[:2]
        app.logger.error("LDAP Exception " + str(exctype) + " " + str(value))
        raise KeyperError("LDAP Exception " + str(exctype) + " " + str(value),401)
  
    app.logger.debug("Exit")
    return jsonify(list),201

@route('/users/<username>', methods=['PUT'])
def update_user(username):
    ''' Update a user '''
    app.logger.debug("Enter")
    dn = "cn=" + username + "," + app.config["LDAP_BASEUSER"]

    req = request.get_json()

    try:
        con = operations.open_ldap_connection()

        # Update req and return list
        ...
        ...
        
        operations.close_ldap_connection(con)
    except ldap.NO_SUCH_OBJECT:
        app.logger.error("Unable to delete. LDAP Entry not found:" + dn)
        raise KeyperError(errors["ObjectDeleteError"].get("msg"), errors["ObjectDeleteError"].get("status"))
    except ldap.LDAPError:
        exctype, value = sys.exc_info()[:2]
        app.logger.error("LDAP Exception " + str(exctype) + " " + str(value))
        raise KeyperError("LDAP Exception " + str(exctype) + " " + str(value),401)


    app.logger.debug("Exit")
    return jsonify(list), 201

@route('/users/<username>', methods=['DELETE'])
def delete_user(username):
    ''' Delete a User '''
    app.logger.debug("Enter")
    dn = "cn=" + username + "," + app.config["LDAP_BASEUSER"]
    
    try:
        con = operations.open_ldap_connection()
        con.delete_s(dn)
        operations.close_ldap_connection(con)
    except ldap.NO_SUCH_OBJECT:
        app.logger.error("Unable to delete. LDAP Entry not found:" + dn)
        raise KeyperError(errors["ObjectDeleteError"].get("msg"), errors["ObjectDeleteError"].get("status"))
    except ldap.LDAPError:
        exctype, value = sys.exc_info()[:2]
        app.logger.error("LDAP Exception " + str(exctype) + " " + str(value))
        raise KeyperError("LDAP Exception " + str(exctype) + " " + str(value),401)

    app.logger.debug("Exit")
    return jsonify("Deleted User: " + username)
```

**@route** before function definition defines route for the function. **method=** within **@route** definition defines what method would trigger the function.

Finally, a file run.py is created at the **app* folder level:

```python
from app import app

if __name__ == "__main__":
    app.run()
```

One cool feature I found with python is the concept of the local venv environment. Where you install all the project-specific libraries locally. So, it does not mess the python libraries at the system level avoiding conflict with other projects. Venv is created like this:

```console
$ mkdir env
$ python3 -m venv env
$ . env/bin/activate
```

Having set your venv, install modules required to run flask application

```console
(env) $ pip install flask
```

Run your flask REST API

```console
(env) $ flask run
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
[17/Sep/2020:21:30:11.055] DEBUG app:create_app: flask_config: config.DevelopmentConfig
[17/Sep/2020:21:30:11.122] INFO werkzeug:_log:  * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
```

## Verify Endpoints
To verify endpoints we can use either curl or Postman. Curl is CLI and Postman is a GUI application for testing APIs. It works by sending request to webservice and getting the response back. 

## Conclusion
We created a Flask REST webservice for Users. As you can see that wrting REST API seems pretty straighforward with Flask. 