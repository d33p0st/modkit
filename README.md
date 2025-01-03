[![PyPI Downloads](https://static.pepy.tech/badge/modkit)](https://pepy.tech/projects/modkit)

# Overview

`modkit` is a collection of tools that can be used for better programming
experience and automation.

## Features

- `@override` decorator for python3.8+
- New `Property` class that automates property definition and takes less number of lines of code to define.
- New `Possibility` class that helps extend return types with custom methods/properties which in turn helps in better readability of code.

## Usage

1. `@override` decorator and functionality.

    The `@override` decorator was introduced in python3.12+ under the `typing` module. However, for python<3.12, it is not available.

    The `override` class under `modkit` supports python3.8+ and brings
    additional features rather than just helping in maintaining an error
    free codebase.

    The `override` class contains two static methods that can be used as
    decorators: `cls` and `mtd`. The `@override.mtd` acts as a decorator
    for the child class methods which are being over-ridden from the
    parent class. It sets the `__override__` attribute of the methods,
    it has been decorated with, as `True`.

    The `@override.cls` decorator on top of a child class, checks for
    all methods that have their `__override__` attribute set as `True`
    and it generates errors accordingly. It will raise
    `MethodOverrideError` if the method set as override is not present in
    the parent class. Additionally, it will also check if the current class
    is a child class or not (inheritance is implemented or not).

    Apart from methods, the properties need not be decorated with `mtd`,
    the `@override.cls` over the class will automatically check if the
    property is over-ridden or not. If a over-ridden property does not include a `setter` or `deleter` or both, the parent property's
    `setter` and `deleter` will be copied to the child class.

    ### Use case

    1. Import the `override` class from `modkit.typing`

        ```python
        from modkit.typing import override
        ```
    2. Define a Base Class.

        ```python
        class Base:
            def __init__(self) -> None:
                self.value = 10
            
            def base_method(self) -> None:
                self.value += 1

            @property
            def base_property(self) -> int:
                return self.value
            
            @base_property.setter
            def base_property(self, value) -> None:
                self.value = value
            
            @base_property.deleter
            def base_property(self) -> None:
                self.value = 0
        ```
    3. Define the Child Class with `override` class.

        ```python
        @override.cls # valid
        class Child(Base):
            @override.mtd # valid
            def base_method(self, incrementor: int) -> None
                self.value += incrementor
            
            @property # valid
            def base_property(self) -> int:
                return self.value + 10
            
            # since no setter, deleter are set,
            # it will use parent's setter and deleter.

            @override.mtd # invalid, not present in base class.
            def child_method(self) -> str:
                return str(self.value)
        ```
    4. The `look_in` classmethod of `override` class.

        The `override` class checks for methods and properties present in
        the child class with respect to the recent parent of the child.
        However, this behavior can be changed to check for the topmost
        base class (not object).

        Example:
        ```python
        override.look_in('topmost') # other values: `recent`

        class Top:
            def top_method(self) -> None:
                pass
        
        @override.cls
        class Mid(Top):
            @override.mtd # valid
            def top_method(self) -> str:
                pass
        
            def mid_method(self) -> None:
                pass
        
        @override.cls
        class Bottom(Mid):
            @override.mtd 
            # invalid, mid_method is present in mid,
            # and not in the top most base class that is Top
            def mid_method(self) -> str:
                pass

            @override.mtd # valid
            def top_method(self) -> int:
                pass
        ```

        To avoid conflicts in most cases, use the default setting,
        or go back to the default setting using `override.look_in('recent')`.
2. `Property` class for automating `property` creation.

    Usage Warning: `Property` class internally uses `property` to maintain,
    create and dispatch property `getter`, `setter` and `deleter`, however,
    it does not support static type checking as this an external class.

    The `Property` class can be used to quickly and neatly define properties
    of a class.

    1. By binding an attribute.

        The `property` class can be used to create a property that is bound
        to an attribute, with easy `setter` and `deleter` creation or blocking.

        ```python
        from modkit.classtools import Property

        class Example:
            def __init__(self, value: int) -> None:
                self._value = value
            
            # note that, this is the only way the Property can be defined,
            # it cannot be defined inside a method, or another property.
            value = Property(
                attribute='_value', # bind to _value,
                setter=True, # enable setter,
                deleter=True, # enable deleter,
                deleter_deletes_attribute=False,
                # this prevents the deleter from actually deleting the
                # attribute (del self._value). Instead, it sets self._value
                # to None.
                # However, if deleter_deletes_attribute is set to True,
                # the _value attribute will be deleted.
            )
        ```

        It also supports blocking `setter` or `deleter` or both.

        ```python
        from modkit.classtools import Property

        class Example2:
            def __init__(self, value: int) -> None:
                self._value = value
            
            value = Property(
                attribute='_value', # bind to _value,
                setter=False, # block setting it,
                deleter=False, # block deletion,
                error=AttributeError, # choose any exception to raise,
                # any custom exception can also be used.
                # as long as it is a child class of Exception or Exception
                # itself,
                setter_error_arguments=("Cannot set _value",),
                # choose setter error arguments for the exception.
                deleter_error_arguments-("Cannot delete _value",),
                # choose deleter error arguments for the exception.
            )

            # This property can only be accessed and cannot be set or
            # deleted. Trying to do the same will raise AttributeError.
        ```
    2. By explicitly defining `getter`, `setter`, and `deleter`.

        ```python
        from modkit.classtools import Property

        class Example3:
            def __init__(self, value: int) -> None:
                self._value = value
            
            value_is_none = Property(
                getter=lambda cls: cls._value is None, # returns bool.
                setter=None, # blocking setter, or define it just like getter.
                deleter=None, # blocking deleter.
                error=ValueError, # set, delete will raise ValueError,
                setter_error_arguments=("Read-only property.",),
                deleter_error_arguments=("Read-only property.",),
            )

            # This will create a property named value_is_none which returns
            # bool, based on _value.

            # using previously defined property.
            value_is_not_none = Property(
                getter=lambda cls: not cls.value_is_none,
                setter=None,
                deleter=None,
                error=AttributeError,
                setter_error_arguments=("Read-only.",),
                deleter_error_arguments=("Read-only.",),
            )

            # creating with setter and deleter.
            value = Property(
                getter=lambda cls: cls._value,
                setter=lambda cls, value: setattr(cls, '_value', value),
                deleter=lambda cls: setattr(cls, '_value', None),
            )
        ```
3. The `Possibility` class to extend return types functionality.

    Suppose there is a function that returns an `int` and that is it for that, it cannot do anything else.

    I wanted to check if the `int` value returned is a positive number
    or negative. A normal code will do it like this:

    ```python
    # a function that returns an int
    def func() -> int:
        return 10
    ```

    ```python
    # checking positive or negative
    >>> func() > 0
    True
    ```

    The `Possibility` way:

    1. Import the `Possibility` abstract class.

        ```python
        from modkit.classtools import Possibility
        ```
    2. Create a new return type, say, `Extra`.

        ```python
        class Extra(Possibility):
            # leave the __init__ method untouched.
            # The Possibility class has three properties
            # parent, attribute (attribute_name) 
            # and value (attribute's value).

            # __bool__ method needs to be defined (as it is an
            # abstract method.), we will do it later.

            # let us create a method named, is_positive.
            def is_positive(self) -> bool:
                return self.value >= 0
            
            # the __bool__ method
            def __bool__(self) -> bool:
                return self.value is not None
                # this implements the `if <class-here>` statement
                # so u can set any logic here.
        ```
    3. Modify the Function.

        ```python
        # The __init__ method of Extra will take 3 parameters.
        # parent: the parent class.
        # attribute: The attribute name
        # value: The value.

        # for now, we will leave, parent, and attribute.
        def func() -> Extra[int]:
            return Extra(parent=None, attribute='', value=10)
        ```
    4. Assess the result.

        ```python
        >>> func() # will return Extra
        <Extra object at XXXXXXX>

        >>> func().is_positive() # if the value is positive.
        True

        >>> func()() # get the value.
        10
        ```
    5. Using in a class.

        We will use the `Extra` class we created above.

        ```python
        class Example:
            def __init__(self, value: int) -> None:
                self._value = value
            
            @property
            def value(self) -> Extra[int]:
                return Extra(
                    parent=self,
                    attribute='_value',
                    value=self._value
                )
        ```

        Let us see the results.

        ```python
        >>> obj = Example(100)
        >>> obj.value # simply using this will return the Extra class object
        <Extra object at XXXXXX>
        >>> obj.value() # even though it is a property, () calling it will return the value
        100
        # if it was a method, you have to call it twice, like value()().
        >>> obj.value.is_positive() # better readability.
        True
        ```

## Contributing

Contributions are welcome, create and issue [here](https://github.com/d33p0st/modkit/issues) and create a pull request [here](https://github.com/d33p0st/modkit/pulls).