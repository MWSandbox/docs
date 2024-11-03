# Udemy Python course

## Basics

- You can use mathematical expressions on string:
    - Multiply string:
    ```
    a = "a"
    aaa = 3 * a
    print(aaa) //will result in output aaa
    ```
    - add strings:
    ```
    a = "a"
    aa = a + a
    print(aa) //will result in output aa
    ```
 - Comparison:
    - `==` is similar to equals in Java (same content)
    - `is` is similar to == in Java (same object according to reference)
    - `id(object)` can be used to get id of an object that would be used for `is` comparison
    - assigning one object to another (e.g. `a = b`) keeps the same object id
- Destructuring variables:
    - Multiple variables can be assigned in a single operation: `a, b = 1, 2`
    - This will also work if right side is a two element tuple: `x, y = position`
    - When ignoring a single value, the `_` can be used: `x, _ = position`
    - `*` notation can be used to collect all other values.
        - `x, y, z = position`
        - `x, *other = position` -> other would contain a tuple of (y, z)
        - `*other, z = position`
- Keywords
    - `pass`: Do nothing
    - `None`: Similar to null in Java

## Data structures
 - Lists: []
    - Access to elements via []
    - .append(element)
    - .remove(element)
    - in keyword to check existence (can also be used in strings)
    - List comprehensions:
        - `doubled = [num * 2 for num in numbers]`
        - `evenNumbers = [num for num in numbers if num % 2 == 0]`
    - `map()` can be used to apply operation on each list entry - e.g.: `doubled = map(double, sequence)`
- Tuples: ()
    - Tuples in python are unmodifiable lists
    - brackets can be omitted if intention is clear. E.g.: `tuple = 5, 11` 
- Sets: {}
    - .add(element)
    - .difference(set2) takes set1 and removes all its elements from set2 
    - .union(set2) unites two sets
    - .intersection(set2) find similarities in both sets
- Dictionaries: {"key": "value"}
    - New elements can be added by `dict[new_key] = new_value`
    - All keys can be accessed by iterating over dict: `for key in dict:`
    - All keys and values can be accessed by: `for key, value in dict.items():`
        - `dict.items()` will in fact return a tuple that is destructured into key and value
    - All values can be accessed by: `for value in dict.values():`
    - Dictionary comprehensions:
        - ```
            users = [(0, "Bob", "password"), (1, "Rolf", "passw4rd")]
            username_to_complete_user = {user[1]: user for user in users}
          ```

## Loops
- `for i in range(4)`

## Functions
- Functions have their own local space for variables separated from the global space -> Do not re-define a global variable within a function
- Parameters can be provided any order by providing the parameter names - e.g.: `say_hello(surname="Bob", name="Smith")`
- Default values can be provided for functions, so they could be omitted when calling the function
    - E.g.: `def add(x, y=8):`
- Functions could return different types of data
- An arbitrary number of arguments can be accepted by specifying `def function(*args):`
- A function with multiple arguments can be called easier by destructuring the arguments:
    - ```
        def add(x, y):
            return x + y
        
        nums = [3, 5]
        add(*nums)
      ```
    - This als works for named parameters and dicts:
      ```
        nums = {"x": 3, "y": 5}
        add(**nums)
      ```
    - Double `**` notation can also be used in function definition to build an arbitrary number of named parameters into a dict.

### Lambda Functions
- Definition: `add = lambda x, y: x + y`
- Definition single line: `(lambda x, y: x + y)(5,7)`
- Call it by running `add(5, 7)`

### Special Methods
- Are identified by to leading `__`
- `__init__` similar to constructor in Java
- `__str__` similar to toString in Java - will be called automatically when print() is called on an object
- `__repr__` used to print out an umabigious representation of an object
    - Used in certain places like debuggers
    - Will also be printed out by `print()` when there is no `__str__` available

## OOP
- Classes can be defined with:
    ```
    class Student:
        def __init__(self): # Similar to constructor in Java
            self.name = "Rolf"
    ```
- Object can be created:
    ```
    student = Student()
    ```
- Instance methods can be added to classes, but require `self` argument, additional arguments can be added also to `__init__`
- Class methods can be added the following way:
    ```
    @classmethod
    def class_method(cls):
    ```
    - Will pass the Class as parameter
    - Can be used as factory to create new instances using `__init__`:
    ```
    class Book:
        TYPES = ("hardcover", "paperback")

        @classmethod
        def paperback(cls, name):
            return cls(name, cls.TYPE[1])
    ```
- Static methods can be added the following way:
    ```
    @staticmethod
    def static_method():
    ```
- Class properties can be added:
    ```
    class Book:
        TYPES = ("hardcover", "paperback")

    Book.TYPES
    ```





