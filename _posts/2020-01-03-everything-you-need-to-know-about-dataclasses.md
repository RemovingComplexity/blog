---
layout: post
title:  "Everything you need to know about dataclasses"
date:   2021-01-03 20:01:00 -0600
categories: article
excerpt: Python classes are powerful. Use dataclasses to create better classes with less code.
tags: 
  - Python
  - Fundamentals
  - Dataclasses
seo:
  type: Article
author_name: Josue Balandrano Coronel
author: rmcomplexity
image: /assets/images/python_dataclasses.png
---


* Table of Contents
{:toc}

Python [data classes](https://docs.python.org/3/library/dataclasses.html) makes it super easy to write better classes by automatically implementing handy dunder methods like `__init__`, `__str__` (string representation) or `__eq__` (equals `==` operator).  [Data classes](https://docs.python.org/3/library/dataclasses.html) also make it easier to create frozen (immutable) instances, serialize instances and enforce type hints usage.

The main parts of a [data class](https://docs.python.org/3/library/dataclasses.html) are:

- `@dataclass` decorator which returns the same defined class but modified
- `field` function which allow for per-field customizations.

> **Note**: throughout this article we will be using different variations of this `Response` class. This class is meant to be a simplified representation of an HTTP response.

## How to create a data class

To create a data class all we need to do is use the `@dataclass` decorator on a custom class like this:

```python
fom dataclasses import dataclass

@dataclass
class Response:
    status: int
    body: str
```

The previous example creates a `Response` class with a `status` and `body` attributes. The `@dataclass` decorator by default gives us these benefits:

- Automatic creation of the following dunder methods:
    1. `__init__`
    2. `__repr__`
    3. `__eq__`
    4. `__str__`
- Enforcing type hints usage. If a field in a data class is defined without a type hint a `NameError` exception is raised.
- `@dataclass` does not create a new class, it returns the same defined class. This allows for anything you could do in a regular class to be valid within a data class.

We can appreciate [data classes'](https://docs.python.org/3/library/dataclasses.html) benefits by taking a look at the previously defined `Response` class.

**Instance initialization:**

```python
>>> resp = Response(status=200, body="OK")
```

**Correct representation of a class:**

```python
>>> import logging 
>>> logging.basicConfig(level=logging.INFO)
>>> resp = Response(status=200, body="OK")
>>> logging.info(resp)
... INFO:root:Response(status=200, body='OK')
```

**Instance equality**

```python
>>> resp_ok = Response(status=200, body="OK")
>>> resp_500 = Response(status=500, body="Error")
>>> resp_200 = Response(status=200, body="OK")
>>> resp_ok == resp_500
... False
>>> resp_ok == resp_200
... True
```

> **Note**: we can customize the implementation of each dunder method. We'll see how later in this article.

## Field definition

There are two ways of defining a field in a data class.

1. Using type hints and an optional default value

```python
from dataclasses import dstaclass

@dataclass
class Response:
    body: str
    status: int = 200
```

The previous class can be instantiated by passing only the `message` value or both `status` and `message` 

```python
>>> import logging
>>> logging.basicConfig(level=logging.INFO)
>>> # Create 200 response
>>> resp_ok = Response(body="OK")
>>> logging.info(resp_ok)
... INFO:root:Response(body='OK', status=200)
>>> # Create 500 response
>>> resp_error = Response(status=500, body="error")
>>> logging.info(resp_error)
... INFO:root:Response(body='error', status=500)
```

2. Using the `field` method. This is recommended when there's a need for more fine grained configuration on a field.

By using the `field` method we can:

### Specify a default value

When using the `field` method we can specify a default value by passing a `default` parameter:

```python
from dataclasses import dataclass

@dataclass
class Response:
    body: str
    status: int = field(default=200)
```

In Python it is not recommended to use [mutable values as argument defaults](https://docs.python-guide.org/writing/gotchas/#mutable-default-arguments). This means it's not a good idea to define a data class like this (the following example is not valid)

```python
from dataclasses import dataclass

@dataclass
class Response:
    status: int
    body: str
    headers: dict = {}
```

If we could use the previous code every instance of `response` would share the same `headers` object and that's not good.

Fortunately data classes help us prevent this by raising an error when something like the example above is used. And if we need to add an immutable object as a default value we can use `default_factory`. 

The `default_factory` value should be a function with no arguments. Commonly used functions include `dict` or `list` :

```python
from dataclasses import dataclass, field

@dataclass
class Response:
    status: int
    body: str
    headers: dict = field(default_factory=dict)
```

We can then use this class like so:

```python
>>> import logging
>>> logging.basicConfig(level=logging.INFO)
>>> # Create 200 response
>>> resp = Response(status=200, body="OK")
>>> logging.info(resp)
... INFO:root:Response(status=200, body='OK', headers={})
```

> **Note**: for mutable default values use `default_factory`

### Include or exclude fields in automatically implemented dunder methods

By default every defined fields are used in `__init__`, `__str__`, `__repr__`, and `__eq__`. The `field` method allows to specify which fields are used when implementing the following dunder methods:

`__init__`

```python
from dataclasses import dataclass

@dataclass
class Response:
    body: str
    headers: dict = field(init=False, default_factory=dict)
    status: int = 200
```

This data class will implement an `__init___` method like this one:

```python
def __init__(self, body:str, status: int=200):
    self.body = body
    self.status = status
    self.headers = dict()
```

This version of the `Response` class will not allow for a `headers` value on initialization. Here's how we could use it:

```python
>>> import logging
>>> logging.basicConfig(level=logging.INFO)
>>> # Create 200 response
>>> 
>>> resp = Response(body="Success")
>>> logging.info(resp)
... INFO:root:Response(body='Success', headers={}, status=200)
>>>
>>> # passing a headers param on initialization will raise an srgument error.
>>> resp = Response(body="Success", headers={})
... TypeError: __init__() got an unexpected keyword argument 'headers'
>>>
>>> # 'headers' is an instance attribute and can be used after initialization.
>>> resp.headers = {"Content-Type": "application/json"}
>>> logging.info(resp)
... INFO:root:Response(body='Success', headers={'Content-Type': 'application/json'}, status=200)
```

> **Note**: Fields that are not used in `__init__` method can also be populated after init using `__post_init__`.

`__repr__` and `__str__`

```python
from dataclasses import dataclass

@dataclass
class Response:
    body: str
    headers: dict = field(repr=False, init=False, default_factory=dict)
    status: int = 200
```

Now, the `Response` class will not print the value of `headers` when an instance is printed.

```python
>>> resp = Response(body="Success")
>>> logging.info(resp)
... INFO:root:Response(body='Success', status=200)
```

`__eq__` 

```python
from dataclasses import dataclass, field

@dataclass
class Response:
    body: str
    headers: dict = field(compare=False, init=False, repr=False, default_factory=dict)
    status: int = 200
```

This version of the `Response` class will not take the `headers` value into consideration when comparing if an instance is equal to another.

```python
>>> resp_json = Response(body="Success")
>>> resp_json.headers = {"Content-Type": "application/json"}
>>> resp_xml = Response(body="Success")
>>> resp_xml.headers = {"Content-Type": "application/xml"}
>>> resp_json == resp_xml
... True
```

Both objects are equal because only the `status` and `body` values are considered when checking for equality and not the `headers` value.

> **Note**: when setting `compare` to `False` on a field it will not be used to automatically implement any comparable methods (`__lt__`, `__gt__`, etc...). More on comparisons later.

### Add field-specific metadata

We can add metadata to a field. The metadata is a mapping and it's meant to be used by 3rd party libraries. The data classes implementation does not use field metadata at all.

> **Note**: If you decide to use field-specific metadata, be mindful, other 3rd party libraries could overwrite any value. It's recommended to use a specific key to avoid collisions.

```python
from dataclasses import dataclass, field
from typing import Any

@dataclass
class Response:
    body: Any = field(metadata={"force_str": True})
    headers: dict = field(init=False, repr=False, default_factory=dict)
    status: int = 200
```

This `Response` class assigns a mapping with the key `force_str` as metadata. The metadata mapping can be used as configuration to force using the string representation of whatever is passed as `body`.

To access a field's metadata the `fields` method can be used. 

```python
>>> from dataclasses import fields
>>> resp = Response(body="Success")
>>> fields(resp)
...(Field(name='body',type=typing.Any,default=<dataclasses._MISSING_TYPE object at 0x7f955a0e97f0>,default_factory=<dataclasses._MISSING_TYPE object at 0x7f955a0e97f0>,init=True,repr=True,hash=None,compare=True,metadata=mappingproxy({'force_str': True}),_field_type=_FIELD),
 Field(name='headers',type=<class 'dict'>,default=<dataclasses._MISSING_TYPE object at 0x7f955a0e97f0>,default_factory=<class 'dict'>,init=False,repr=False,hash=None,compare=True,metadata=mappingproxy({}),_field_type=_FIELD),
 Field(name='status',type=<class 'int'>,default=200,default_factory=<dataclasses._MISSING_TYPE object at 0x7f955a0e97f0>,init=True,repr=True,hash=None,compare=True,metadata=mappingproxy({}),_field_type=_FIELD))
```

The `fields` method returns a tuple of `Fields` objects. It can be used on an instance or a class.

To retrieve the `body` field we can use a comprehension and `next` 

```python
>>> body_field = next(
    (field
     for field in fields(resp)
     if field.name == "body"),
    None
)
>>> logging.info(body_field)
... INFO:root:Field(name='body',type=typing.Any,default=<dataclasses._MISSING_TYPE object at 0x7f955a0e97f0>,default_factory=<dataclasses._MISSING_TYPE object at 0x7f955a0e97f0>,init=True,repr=True,hash=None,compare=True,metadata=mappingproxy({'force_str': True}),_field_type=_FIELD)
>>> logging.info(body_field.metadata)
... INFO:root:{'force_str': True}
```

## Customize object initialization using `__post_init__`

The `@dataclass` decorator automatically implements an `__init__` method. By using `__post_init__` we can add custom logic on initialization without having to re-implement `__init__`.

```python
from dataclasses import dataclass, field 
from typing import Any  
from sys import getsizeof

@dataclass
class Response:
    body: str  
    headers: dict = field(init=False, compare=False, default_factory=dict)
    status: int = 200

    def __post_init__(self):
        """Add a Content-Length header on init"""
        self.headers["Content-Length"] = getsizeof(self.body)
```

When the previous class is instantiated the content length is automatically calculated.

```python
>>> import logging
>>> logging.basicConfig(level=logging.INFO)
>>> # Create 200 response
>>> resp = Response("Success")
>>> logging.info(resp)
... INFO:root:Response(body='Success', headers={'Content-Length': 56}, status=200)
```

We can also access field specific metadata in `__post_init__` 

```python
from dataclasses import dataclass, is_dstaclass, asdict

@dataclass
class Response:
    body: Any = field(metadata={"force_str": True})
    headers: dict = field(init=False, compare=False, default_factory=dict)
    status: int = 200

    def stringify_body(self):
        """Returns a string representation of the value in body"""
        body = self.body
        if is_dataclass(body):
            body = asdict(body)
        if isinstance(body, dict):
            return json.dumps(body)
        if not isinstance(body, str):
            return str(body)
        return body

    def __post_init__(self):
        """Custom int logic.
           
        - Check if body is configured to force value as string  
        - Calculate body's length and add corresponding header. 
        """
        body_field = self.__dataclass_fields__["body"]
        if body_field.metadata["force_str"]:
            self.body = self.stringify_body()
        self.headers["Content-Length"] = getsizeof(self.body)
```

And we can use this class like this:

```python
>>> import logging
>>> logging.basicConfig(level=logging.INFO)
>>> # Create 200 response
>>> resp = Response(body={"message": "Success"})
>>> logging.info(resp)
... INFO:root:Response(body='{"message": "Success"}', headers={'Content-Length': 71}, status=200)
```

The `body` value is automatically serialized into a string and stored in the calss on initialization.

The previous example is mainly to show custom initialization logic. In reality you might not want to store the string representation of a response body, instead it's better to make the class serializable.

> **Note**: you can use `asdict` to transform a data class into a dictionary, this is useful for string serialization.

We can also specify fields which will not be attributes of an instance but will be passed onto the `__post_init__` hook by using `dataclasses.InitVar`

```python
from dataclasses import dataclass, is_dstaclass, asdict, InitVar

@dataclass
class Response:
    body: Any
    headers: dict = field(init=False, compare=False, default_factory=dict)
    status: int = 200
    force_body_str: InitVar[bool] = True

    def stringify_body(self):
        """Returns a string representation of the value in body"""
        body = self.body
        if is_dataclass(body):
            body = asdict(body)
        if isinstance(body, dict):
            return json.dumps(body)
        if not isinstance(body, str):
            return str(body)
        return body

    def __post_init__(self, force_body_str):
        """Custom int logic.
           
        - Check if body is configured to force value as string  
        - Calculate body's length and add corresponding header. 
        """
        if force_body_str:
            self.body = self.stringify_body()
        self.headers["Content-Length"] = getsizeof(self.body)
```

We can easily configure if the value of `body` will be stored as string or not:

```python
>>> import logging
>>> logging.basicConfig(level=logging.INFO)
>>> # Create 200 response where 'body' will be stored as a dict.
>>> resp = Response(body={"message": "Success"}, force_body_str=False)
>>> logging.info(resp)
... INFO:root:Response(body={'message': 'Success'}, headers={'Content-Length': 232}, status=200)
>>> # Create 200 response where 'body' will be stored as a string.
>>> resp_str = Response(body={"message": "Success"})
>>> logging.info(resp)
... INFO:root:Response(body='{"message": "Success"}', headers={'Content-Length': 71}, status=200)
```

## Data classes that we can compare and order

By default a data class implements `__eq__`. We can pass an `order` boolean argument to the `@dataclass` decorator to also implement `__lt__` (less than), `__le__` (less or equal), `__gt__` (greater than) and `__ge__` (greater or equal).

The way these rich [comparison methods](https://docs.python.org/3/reference/datamodel.html#object.__lt__) are implemented take every defined field and compare them in the order they are defined until there's a value that's not equal.

```python
from dataclasses import dataclass

@dataclass(order=True)
class Response:
    body: str
    status: int = 200
```

The previous data class can now be compared using `>=`, `<=`, `>` and `<` operands. The best use case for this is when sorting:

```python
>>> resp_ok = Response(body="Success")
>>> resp_error = Response(body="Error", status=500)
>>> sorted([resp_ok, resp_error])
... [Response(body='Error', status=500), Response(body='Success', status=200)]
```

In this example `resp_error` goes before `resp_ok` because the unicode value of `E` is less than the unicode value of `S`.

> **Note**: implementing rich comparison methods allow us to easily sort objects.

The implemented comparison methods will check the value of `body`, if both are equal it will continue to `status`. If the class had more fields the rest of the fields would be checked in order until a non-equal value is found.

The previous example is valid but it does not make much sense to sort `Response` objects based on the `body` and `status` values. It makes more sense to sort them on the length of the body. We can specify which fields to use in comparison by using the `field` method:

```python
from dataclasses import dataclass, field 
from sys import getsizeof

@dataclass(order=True)
class Response:
    body: str = field(compare=False)
    status: int = field(compare=False, default=200)
    _content_length: int = field(compare=True, init=False)

    def __post_init__(self):
        """Calculate and store content length on init"""
        self._content_length = getsizeof(self.body)
```

In the previous example we specified which fields are used when implementing comparison methods by passing a boolean `compare` parameter to the `field` method.

This class will now be sorted by the size of the value of `body`. We can also judge if an instance is larger than another judging by the size of the value of `body`.

```python
>>> resp_ok = Response(body="Success")
>>> resp_error = Response(body="Error", status=500)
>>> sorted([resp_ok, re sp_error])
...[Response(body='Error', status=500, _content_length=54), Response(body='Success', status=200, _content_length=56)]
>>> # resp_error is smaller than resp_ok because
>>> # "Error" is smaller than "Success"
```

One downside of this implementation is that two given instances will be equal as long as the size of the `body` attribute is the same. 

```python
>>> resp_ok = Response(body="Success")
>>> resp_error = Response(body="Failure")
>>> resp_ok == resp_error
... True
>>> # both instances are equal because Success and Failure have the same amounts of chars
>>> # and getsizeof() returns the same size for both strings.
```

For equality it would be better to also check if the value of the `body` attribute is the same:

```python
from dataclasses import dataclass, field 
from sys import getsizeof

@dataclass(order=True)
class Response:
    _content_length: int = field(compare=True, init=False)
    body: str = field(compare=True)
    status: int = field(compare=False, default=200)

    def __post_init__(self):
        """Calculate and store content length on init"""
        self._content_length = getsizeof(self.body)
```

By moving the `_content_length` field definition above `body` the length of the content will be used first for any comparisons. We also set the `body` field as a `compare` field. When checking for equality if the content length is the same the actual value of `body` will be checked, making for a better way to check for equality.

```python
>>> resp_ok = Response(body="Success")
>>> resp_error = Response(body="Failure")
>>> resp_ok == resp_error
... False
```

This also works for sorting since response instances with the same content length will be sorted by the weight of the characters. Sorting will always yield the same order.

```python
>>> sorted([resp_ok, resp_error])
... [Response(_content_length=56, body='Failure', status=200), Response(_content_length=56, body='Success', status=200)]
```

> **Note**: the order in which fields are defined matter for comparisons.

## Frozen (or immutable) instances

We can create frozen instances by passing `frozen=True` to the `@dataclass` decorator.

```python
from dataclasses import dataclass, field

@dataclass(frozen=True)
class Response:
    body: str
    status: int = 200
```

This is helpful when you want to make sure read-only data is not mistakenly modified by your code or 3rd party libraries. If you try to modify a value a `FrozenInstanceError` exception will be raised:

```python
>>> resp_ok = Response(body="Success")
>>> resp_ok.body = "Done!"
... dataclasses.FrozenInstanceError cannot assign to field 'body'
```

In Python we cannot really have [immutable objects](https://discuss.python.org/t/immutability-in-python-is-really-hard/2536). If you make an effort you can still modify a frozen data class instance:

```python
>>> import logging
>>> logging.basicConfig(level=logging.INFO)
>>> # Check values of 'resp_ok'
>>> logging.info(resp_ok)
... INFO:root:Response(body='Success', status=200)
>>>
>>> object.__setattr__(resp_ok, "body", "Done!")
>>> # We have modified a "frozen" instance
>>> logging.info(resp_ok)
... INFO:root:Response(body='Done!', status=200)
```

This is unlikely to happen but it's worth knowing.

> **Note**: use a frozen data class when using read-only data to avoid unwanted side-effects.

> **Note**: You cannot implement `__post_init__` hook in a frozen data class.

## Updating an object instance by replacing the entire object.

The data classes module also offers a `replace` method which created a new instance using the same class. Any updates are passed as parameters:

```python
from dataclasses import dataclass, replace

@dataclass(frozen=True)
class Response:
    body: str
    status: int = 200

>>> import logging
>>> logging.basicConfig(level=logging.INFO)
>>> # Create 200 response
>>> resp_ok = Response(body="Success")
>>> logging.info(resp_ok)
... INFO:root:Response(body='Success', status=200)
>>> # Replace instance
>>> resp_ok = replace(resp_ok, body="OK")
>>> logging.info(resp_ok)
... INFO:root:Response(body='OK', status=200)
```

The value of `body` is updated and the value of `status` is copied over. Any reference to `resp_ok` is now pointing to the new, updated object.

> **Note**: using `replace` ensures `_init_` and `__post_init__` are run with the updated values.

## Adding class attributes

In Python a class can have a [class attribute](https://realpython.com/lessons/class-and-instance-attributes/), the difference from instance attributes are mainly these two:

1. Class attribute are defined outside `__init__`
2. Every instance of the class will share the same value of a class attribute.

We can define class attributes in a data class by using the pseudo-field `typing.ClassVar`

```python
from dataclasses import dataclass
from typing import ClassVar, Any
from sys import getsizeof 
from collections.abc import Callable

@dataclass
class Response:
    body: str 
    _content_length: int = field(default=0, init=False)
    status: int = 200
    getsize_fun: ClassVar[Callable[[Any], int]] = getsizeof

    def __post_init__(self):
        """Calculate content length by using getsize_fun"""
        self._content_length = self.getsize_fun(self.body)
```

In this version of `Response` we can specify a function used to calculate the content's size. By default `sys.getsizeof` is used.

```python
from functools import reduce

def calc_str_unicode_weight(self, string: str):
    """Calculates strn weight by adding each character's unicode value"""
    return reduce(lambda weight, char: weight+ord(char), string, 0)

@dataclass
class ResponseUnicode(Response):
    getsize_fun: ClassVar[Callable[[Any], int]] = calc_str_unicode_weight

>>> import logging
>>> logging.basicConfig(level=logging.INFO)
>>> # Create 200 response, using getsizeof to calculate content length
>>> resp_ok = Response(body="Success")
>>> logging.info(resp_ok)
... INFO:root:Response(body='Success', _content_length=56, status=200)
>>> # Override function to use when calculating content length
>>> resp_ok_unicode = ResponseUnicode(body="Success")
>>> logging.info(resp_ok_unicode)
... INFO:root:ResponseUnicode(body='Success', _content_length=729, status=200)
```

To overwrite the functino used to calculate the content lenght we subclass `Response` and pass the function we want as `getsize_fun`

> **Note**: fields that use `ClassVar` are not used in data class mechanism like `__init__`, equality or comparison dunder methods.

## Inheritance in data classes

When using inheritance with data classes fields are merged, meaning child classes can overwrite field definitions. Everything else works the same since the `@dataclass` decorator returns an old regular Python class.

```python
from dataclasses import dataclasses

@dataclass
class Response:
    body: str
    status: int
    headers: dict

@dataclass
class JSONResponse(Response):
    status: int = 200
    headers: dict = field(default_factory=dict, init=False)

    def __post_init__(self):
        """automatically add Content-Type header"""
        self.headers["Content-Type"] = "application/json"
```

In the previous example the parent class `Response` defined the basic fields and the children class `JSONResponse` overwrites the `headers` field and sets a default value for the `status` field.

```python
>>> import logging
>>> logging.basicConfig(level=logging.INFO)
>>> # Create 200 response
>>> resp_ok = JSONResponse(body=json.dumps({"message": "OK"}))
>>> logging.info(resp_ok)
... INFO:root:JSONResponse(body='{"message": "OK"}', status=200, headers={'Content-Type': 'application/json'})
```

## Hash-able object

The `@dataclass` decorator will automatically [implement `__hash__` method](https://docs.python.org/3/reference/datamodel.html#object.__hash__) if the parameters `frozen` and `eq` are `True`. `frozen` is `False` by default and `eq` is `True` by default.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Response:
    body: str
    status: int = 200
```

We can now use any instance of this class as a key in a `dict` or in a `set`. For I stance, we can create a mapping of responses to users 

```python
>>> import logging
>>> logging.basicConfig(level=logging.INFO)
>>> # Create 200 response
>>> resp_ok = Response(body="Success")
>>> # Create 500 response
>>> resp_error = Response(body="Error", status=500)
>>> # Create a mapping of response -> usernames
>>> responses_to_users = {
...     resp_ok: ["j_mccain", "a_perez"],
...     resp_error: ["d_dane", "b_rodriguez"]
... }
>>> logging.info(responses_to_users[resp_ok])
... INFO:root:['j_mccain', 'a_perez']
```

We can force a `__hash__` function implementation even if we don't set `frozen` and `eq` to `True` by passing `force_hash=True` to the `@dataclass` decorator. This should only be used if you are 100% sure you need the functionality.

## A use case for data classes

Throughout this article we've made different updates to a `Response` class which represents a simplified HTTP response object. Let's put everything together.

For simplicity we're gonna write every class and function we're going to use in the same file, in really these should be spread out into sensible modules.

> **Note**: This is still not a 100% real HTTP response class, but it has enough information to see how we can use data classes

```python
import logging
from sys import getsizeof
from typing import ClassVar, Any, Optional, cast
from collections.abc import Callable
from dataclasses import dataclass, field, InitVar, asdict

logging.basicConfig(level=logging.DEBUG)

class APIException(Exception):
    """Custom API exception."""

    def __init__(self, message: str, **kwargs):
        self.message = message;
        self.data: Dict[str: Any] = kwargs

class PositiveNumberValidator:
    """Descriptor to make sure a value is a positive number"""

    def __set_name__(self, owner, name):
        self.name = f"_{name }"

    def __get__(self, obj, objtype=None):
        return getattr(obj, self.name)

    def __set__(self, obj, value):
        self.validate(value)
        setattr(obj, self.name, value)

    def validate(self, value):
        if not isinstance(value, int):
            raise AttributeError(f"value of '{self.name}' must be a number")
        if value < 0:
            raise AttributeError(f"value of '{self.name}'' must be a positive number")

@dataclass(eq=False)
class Pager:
    """Pager class.

    This class is to hold any pager related data sent in the response.
    The prev and next paramteres are meant to be links sent in the 'Link' header.
    """

    page: int = cast(int, PositiveNumberValidator())
    prev: str 
    next: str

@dataclass(order=True)
class HTTPResponse:
    """Parent HTTPResponse

    This class can:
      1. Ordered and compared by its content size and body using regular operators
      2. Pass a 'content_type' string to be used as header value.
      3. Update headers directly as a regular dictionary.
      4. Customize the function to calculate content-length.
    """
    _content_length: int = field(init=False)

    body: Any
    pager: Optional[Pager] = field(default=None, compare=False)
    headers: dict = field(default_factory=dict, init=False, compare=False)
    status: int = field(default=200, compare=False)

    content_type: InitVar[str] = "text/html"

    getsize_fun: ClassVar[Callable[[Any], int]] = getsizeof

    def __post_init__(self, content_type):
        """automatically calculate header values."""
        self._content_length = self.getsize_fun(self.body)
        self.headers["Content-Length"] = self._content_length
        self.headers["Content-Type"] = content_type

@dataclass(frozen=True)
class JSONBody:
    """Class to hold data sent in a JSON Response.

    This class is immutable to avoid any unwanted modification of data.
    """
    message: str
    data: dict

    @classmethod
    def from_exc(cls: type, exc: APIException):
        """Initializes a JSONBody object from an exception."""
        return cls(message=str(exc), data=getattr(exc, "data", {}))

@dataclass
class JSONResponse(HTTPResponse):
    """Class to represent a JSON Response. Child of HTTPResponse."""
    body: JSONBody
    content_type: InitVar[str] = "application/json"
```

Here, we created a parent class `HTTPResponse` which will hold the basic data needed to send an HTTP response. Then, we created a `JSONResponse` class which inherits from `HTTPResponse` and overwrites `body` and `content_type` attributes. Overwriting these attributes allow us to specify a different default content type and a different type for the `body`. There is also a `Pager` class which is used to hold any data related to pagination that's sent in the response. The `Pager` class uses a descriptor to validate that `page` is always a positive number. And we also have a `JSONBody` which can be initialized by passing a `message` and a dictionary for `data` or can be initialized by passing an `APIException` instance. `APIException` is a custom exception we created to store an exception message and also some data related to said exception.

**Here's some basic examples how we can use these classes**:

1. We can create a `JSONResponse` only by passing a `JSONBody` instance:
    ```python
    >>> import logging
    >>> logging.basicConfig(level=logging.INFO)
    >>> body = JSONBody(message="Success", data={"values": ["value1", "value2"]})
    >>> resp = JSONResponse(body)
    >>> logging.info("resp: %s", resp)
    ... INFO:root:resp: JSONResponse(_content_length=48, body=JSONBody(message='Success', data={'values': ['value1', 'value2']}), pager=None, headers={'Content-Length': 48, 'Content-Type': 'application/json'}, status=200)
    ```
    This is already a powerful class and the code needed to implement it is quite short
2. We can also pass a `Pager` instance to make it a more robust response:
    ```python
    >>> pager = Pager(1, prev="?prev=0", next="?next=2")
    >>> resp = JSONResponse(body, pager=pager)
    >>> logging.info("resp: %s", resp)
    ... INFO:root:resp: JSONResponse(_content_length=48, body=JSONBody(message='Success', data={'values': ['value1', 'value2']}), pager=Pager(prev='?prev=0', next='?next=2'), headers={'Content-Length': 48, 'Content-Type': 'application/json'}, status=200)
    ```
3. We can easily conver data classes to dictionaries or tuples even when using nested data classes:
    ```python
    >>> from dataclasses import asdict, astuple
    >>> logging.info("serialized resp: %s", asdict(resp))
    ... INFO:root:serialized resp: {'_content_length': 48, 'body': {'message': 'Success', 'data': {'values': ['value1', 'value2']}}, 'pager': {'prev': '?prev=0', 'next': '?next=2'}, 'headers': {'Content-Length': 48, 'Content-Type': 'application/json'}, 'status': 200}
    >>> logging.info("resp as tuple: %s", astuple(resp))
    ... INFO:root:resp astuple: (48, ('Success', {'values': ['value1', 'value2']}), ('?prev=0', '?next=2'), {'Content-Length': 48, 'Content-Type': 'application/json'}, 200)
    ```

With a more real example we can see the strengths and weaknesses of data classes.

### Benefits of using data classes

1. We can create powerful classes with less code.
2. Type hints are enforced for every class and instance attribute.
3. We can customize how special dunder methods are implemented.
4. We can use data classes in the same way we use regular classes. In the previous example we used descriptors and class methods without an issue.
5. Inheritance can be used to make it easier to use data classes.
6. It's esier to serialize instsances to dictionaries or tuples.
7. We can mix regular classes and data classes.

### Disadvantages of using data classes

1. When creating data classes that can be compared and ordered the order in which you define the fields matters. Read-ability can take a hit because of this. It is recommended to try and separate fields by type. In the `HTTPResponse` class we have first private attributes, then instance attributes, init only parameters and class attributes.
2. Field definition order also matters when using default values. Since `__init__` 's arguments are implemented using the same order the fields are defined, we have to first define attributes without default values and then attributes with default values.
3. When using `frozen=True` we cannot update values in `__post_init__`
4. We have to manually optimize attribute access if needed. Meaning, adding `__slots__`. [Real Python has a great example of this.](https://realpython.com/python-data-classes/#optimizing-data-classes)

I hope this article sheds some light on how and when to use data classes. If you like it, please follow this blog and make sure to follow me on [twitter](http://twitter.com/rmcomplexity).
