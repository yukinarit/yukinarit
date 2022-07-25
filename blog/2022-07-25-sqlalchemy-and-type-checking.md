# SQLAlchemy and type checking

The japanese translation of this post can be found at [SQLAlchemyで型チェックをがんばる](https://zenn.dev/yukinarit/articles/8cf10358be0086).

I've been using SQLAlchemy for a long time, but my understanding of type checking has been subtle, so I put it together.

* [Common misconceptions](#common-misconceptions)
    - [Static and runtime type checking](#static-and-runtime-type-checking)
* [Setup](#setup)
    - [Installation](#installation)
    - [mypy configuration](#mypy-configuration)
* [Specify the proper type for your model](#specify-the-proper-type-for-your-model)
    - [All columns are Optional by default](#all-columns-are-optional-by-default)
    - [There is a range of SQLAlchemy types.](#there-is-a-range-of-sqlalchemy-types)
    - [type annotations expected in class definitions](#type-annotations-expected-in-class-definitions)
* [Specify the type properly in the query](#specify-the-type-properly-in-the-query)
* [I still want runtime type checking too!](#i-still-want-runtime-type-checking-too-)
* [SQLAlchemy 2.0](#sqlalchemy-20)

## Common misconceptions

#### Static and runtime type checking

A common misconception among those unfamiliar with Python type checking, especially those coming from statically typed languages, is that most libraries, including SQLAlchemy, only provide static type checking.

For example, the following code that specifies a type in PEP484 will have mypy inspect the code and tell you the type error.
```python
@dataclass
class Foo:
    i: int

Foo('10') # <== Argument 1 to "Foo" has incompatible type "str"; expected "int"
```

However, if the type declaration is wrong or the wrong type comes at runtime, as shown below, mypy and the python interpreter will not tell you anything.

```python
Foo(json.loads('"10"')) # no error from mypy
```


If you want static type checking and runtime type checking, there is currently such a library (e.g. [pyserde](https://github.com/yukinarit/pyserde), [beartype](https://github.com/beartype/beartype), [pydantic](https://pydantic-docs.helpmanual.io/)).

For example, [pyserde](https://github.com/yukinarit/pyserde) allows runtime type checking when deserializing from JSON.

```python
@serde
@dataclass
class Foo
    i: int

# SerdeError: Foo.i is not instance of int
from_json(Foo, '{"i": "10"}', type_check=Strict)
```

This article will focus on how to do your best static type checking in SQLALchemy, and finally, I will also suggest a way to do runtime type checking.

## Setup


#### Installation

* SQLAlchemy 1.4

Specify the `mypy`extras when installing sqlalchemy. Then a package called [sqlalchemy2-stubs](https://github.com/sqlalchemy/sqlalchemy2-stubs) will be installed together. sqlalchemy2-stubs is a package that provides SQLAlchemy code type information for SQLAlchemy code.
```
pip install sqlalchemy[mypy].
```

* SQLAlchemy 2.X

```
pip install git+https://github.com/sqlalchemy/sqlalchemy.git
```
SQLAlchemy 2 series targets Python 3.6 and above, so sqlalchemy2-stubs is no longer needed.


#### mypy configuration

Enable the sqlalchemy plugin in mypy. The reason why you need the plugin is that mypy does not actually execute the python code when type checking, so libraries like SQLAlchemy that mess around with types at runtime cannot type check properly. So they provide a mypy plugin so that they can type-check SQLAlchemy's messy parts. pydantic does the same [mypy plugin](https://pydantic-docs.helpmanual. io/mypy_plugin/).

```ini
[mypy].
plugins = sqlalchemy.ext.mypy.plugin
```

By the way, according to Mike Bayer, the author of SQLAchemy, it is extremely hard to maintain mypy plugins.

> Mypy plugins are extremely difficult to develop, maintain and test, as a Mypy plugin must be deeply integrated with Mypy's internal The SQLAlchemy Mypy plugin has lots of limitations when used The SQLAlchemy Mypy plugin has lots of limitations when used with code that deviates from very basic patterns which are reported regularly.

https://docs.sqlalchemy.org/en/14/orm/extensions/mypy.html

## Specify the proper type for your model

#### All columns are Optional by default

In the following model, the NOT NULL `name` and `detail` and the PrimaryKey `id` should intuitively be Non Optional, but in SQLAlchemy [they are all `Optional` by default](https://docs. sqlalchemy.org/en/14/orm/extensions/mypy.html#introspection-of-columns-based-on-typeengine).

```python
class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    name = Column(String, nullable=False)
    detail = Column(JSON, nullable=False)
    money = Column(Numeric, nullable=True)
    updated_at = Column(DateTime, nullable=False)

user = User(id=None) # <= mypy cannot detect errors
````

I think this specification is subtle, but in the case of autoincrement, I may want PrimaryKey to be ``Optional``.

#### There is a range of SQLAlchemy types.

In the example above, `money` is `Numeric`, but the type recognized by mypy seems to be `Optional[Union[float, Decimal]]`. The reason why there are two candidates, `float` and `Decimal`, is that the type returned by SQLAlchemy changes depending on whether `asdecimal` is True or False, as shown below when defining a field.

```python
    # as Decimal
    money = Column(Numeric, nullable=True)
    # become float
    money = Column(Numeric(asdecimal=False), nullable=True)
````

#### type annotations expected in class definitions

It's kind of hard to program if all fields are `Optional` and `Union`. SQLAlchemy allows you to add PEP484-style type annotations when defining model classes, so you can specify the type you expect.

```python
class User(Base):
    __tablename__ = "users"

    id: int = Column(Integer, primary_key=True)
    name: str = Column(String, nullable=False)
    detail: dict[str, Any] = Column(JSON, nullable=False)
    money: Optional[Decimal] = Column(Numeric, nullable=True)
    updated_at: datetime = Column(DateTime, nullable=False)
````

The following code will then allow mypy to detect type errors.
````python
user = User(id=None) # <= Argument "id" to "User" has incompatible type "None"; expected "int"
````

## Specify the type properly in the query

As of ``1.4.39``, the Query API does not seem to recognize the type well, so you have to specify it yourself.

```python
u: User = s.query(User).one()
````

If you don't want to return by return value, you can specify it as follows
```python
u: User
for u in s.query(User):
    ...
```

It would be easier if the SQLAlchemy code would specify the type. SQLAlchemy v2 will be [much improved](https://github.com/sqlalchemy/sqlalchemy2-stubs/issues/214) in this area, so I'm looking forward to it.

## I still want runtime type checking too!

>
> **NOTE**: SQLAlchemy does not have any runtime checks either, SQLAlchemy (or DB API?) accepts values of different types as long as they are compatible. You can put `str` into a Decimal type as follows, and it will work with neither an `int` nor a `float` without an error.
>
> ```python
> u = User(money="1000")
> ````
>
> On the other hand, if I put in a string that is not a number, it will give me an error.
>
> ```python
> u = User(money="aaaa")
> ```
>
> I'm happy with this behavior, but I still want more strict type checking.
>

It is still useful to have runtime type checking as well, so here is the most likely way to do it.

SQLAlchemy models can also be defined with `dataclass`. (https://docs.sqlalchemy.org/en/20/changelog/whatsnew_20.html#native-support-for-dataclasses-mapped-as-orm-models)The advantage of using this method is that you can use ` dataclass` functionality and the ability to use various libraries that support `dataclass`.

```python
@mapper_registry.mapped
@beartype
@dataclass
class User:
    __table__ = Table(
        "users",
        mapper_registry.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String, nullable=False),
        Column("detail", JSON, nullable=False),
        Column("money", Numeric, nullable=True),
        Column("updated_at", DateTime, nullable=False)
    )
    id: Optional[int]
    name: str
    detail: dict[str, Any] (optional[Decimal])
    money: Optional[Decimal] (if any)
    updated_at: datetime
```

Try using the library [beartype](https://github.com/beartype/beartype), which provides runtime type checking.

![](images/2022-07-25-sqlalchemy-and-type-checking/beartype.png)

beartype is dataclass compliant and will do runtime type checking for you by simply adding the `@beartype` decorator.

Now you can run the following code to not only do static type checking with mypy, but also to error at runtime!
```python
u = User(id=None, name="foo", detail={}, money=1000, updated_at=datetime.now())
```

* mypy error.
```python
error: Argument "money" to "User" has incompatible type "int"; expected "Optional[Decimal]"
```
* runtime error
```python
beartype.roar.BeartypeCallHintParamViolation: @beartyped __create_fn__. __init__() parameter money=1000 violates type hinting.Optional[decimal.Decimal], as 1000 not <protocol "builtins.NoneType"> or <protocol "decimal.Decimal">.
```

## SQLAlchemy 2.0

I'm looking forward to trying out the new version 2.0 when it is released.
