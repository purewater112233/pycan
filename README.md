#PyCan

PyCan is a framework-agnostic open source authorization library that can be easily integrated with any Python application.

PyCan's primary goal is to make sure users are kept on the line and don't do anything they were not supposed to.

PyCan's two main functions are [`authorize`](https://github.com/jusbrasil/pycan/blob/master/pycan/__init__.py#L78) and [`can`](https://github.com/jusbrasil/pycan/blob/master/pycan/__init__.py#L11). [`can`](https://github.com/jusbrasil/pycan/blob/master/pycan/__init__.py#L11) is used for granting permission for a `user` to perform an `action` on a given `target`, whereas the `authorize` method performs the authorization per se.

PyCan uses a white-list approach, which means that authorization will be denied for any (action, target) combinations that were not explicitly granted permissions to with a call to [`can`](https://github.com/jusbrasil/pycan/blob/master/pycan/__init__.py#L11).

##Installation

`pip install pycan`

##Getting started

### 1. Declare authorization rules

```python
from pycan import can

can(['delete', 'edit'], # Action that is being performed on a given target
  'profile',  # The target where the action will be performed
  lambda user, context, profile: user.id in profile.owners,
  load_before=lambda user, context: load_profile(context.profile_id)) 
```

As the example above shows, a call to  [`can`](https://github.com/jusbrasil/pycan/blob/master/pycan/__init__.py#L11) always expects an `action` (the first parameter), a `target` (the second parameter) and an `authorization rule` (the 3rd parameter).

The `load_before` parameter is optional and it will be explained further below.

#### 1.1. Actions and Targets

An `action`/`target` pair is basically what a given `user` is trying to do with the application. In the example above, a user is trying to edit or delete (actions) a profile (target).

Both the `action` and the `target` parameters accept lists. For such cases, the authorization rule will be applied to any combination of actions and targets.

#### 1.2. Authorization rules

Authorization rules are functions that determine whether or not the user is allowed to perform the `action` on the `target`. Those functions must always accept three parameters: user, context and authorization resource.

##### User

This is obvious :)

#### Context

The context is everything that is going on at the moment the authorization took place. In a web application the context usually represents the request parameters.

### 1.3. The `load_before` parameter

The `load_before` parameter is responsible for loading data that is required to perform the authorization and has not yet been loaded.

In the example above, we are trying to make sure that a user can edit or delete a profile. In order to determine that, the profile must be loaded prior to running the authorization rule. That's where the `load_before` parameter comes to picture. This parameter accepts a function that will be responsbile for loading the profile (the authorization resource). Its return value is passed as the 3rd parameter of the authorization rule (the `profile` parameter).

### 1.4. The `load_after` parameter

This parameter is optional and it is called only if authorization succeeds. Its main goal is to provide the application with the resource it needs to implement the user action.

### 2. Combining authorization rules

In section 1 we have taught you how to create an authorization rule that requires an authorization resource. Now, say you would like to allow admin users to delete/edit anybody's profiles. One way to implement this is to change the authorization rule to also check if the user is an admin:

```python
from pycan import can

can(['delete', 'edit'], 
  'profile',  
  lambda user, context, profile: user.admin or (user.id in profile.owners), 
  load_before=lambda user, context: load_profile(context.profile_id))
```

The problem with the implementation above is that it does not favor code reuse. The `user.admin` check would need to be duplicated on every place where admins have special permissions.

Fortunately, PyCan allows you to effortlessly combine authorization functions through the `or_`, `and_` and `not_` functions:

```python
from pycan import can, or_

def user_is_admin(user, context, resource):
  return user.admin

can(['delete', 'edit'], 
  'profile',  
  or_(
    user_is_admin,
    lambda user, context, profile:  user.id in profile.owners), 
  load_before=lambda user, context: load_profile(context.profile_id)) 
```

The `and_` and `or_` functions also do perform short-circuiting. Therefore, it will not perform unncessary evaluations of authorization rules. E.g.: In the example above, it would not execute the second authorization rule if the user is an admin.

### 3. Authorizing

The authorization itself happens when your application's entry points are intercepted with the [`authorize`](https://github.com/jusbrasil/pycan/blob/master/pycan/__init__.py#L78) method:

```python
from pycan import authorize

try:
  pycan.authorize(action, target, user=user, context=context)
except UnauthorizedResourceError, e:
  abort(403)
```

Notice that `action`, `target`, `user` and `context` are mapped to the exact same objects used in the calls to the [`can`](https://github.com/jusbrasil/pycan/blob/master/pycan/__init__.py#L11) method.

The call to [`authorize`](https://github.com/jusbrasil/pycan/blob/master/pycan/__init__.py#L78) will raise an UnauthorizedResourceError if the authorization rule defined by the call to [`can`](https://github.com/jusbrasil/pycan/blob/master/pycan/__init__.py#L11) does not evaluate to True or if an authorization rule for that given action/target pair does not exist.

It is up to your program to handle that exception. For instance, in a web app, you could have the app return a 403.
