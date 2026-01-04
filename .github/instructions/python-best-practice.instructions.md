---
description: 'PEP 8-compliant Python coding standards and best practices'
applyTo: '**/*.py'
---

# Python Best Practices

## Project Context

- Target Language: Python 3.10 or later
- Style Guide: PEP 8
- Type Checker: Mypy
- Formatter: Black
- Import Organizer: isort

## Coding Style

### Indentation and Whitespace

- Use 4-space indentation (no tabs)
- Limit line length to 79 characters
- Insert 2 blank lines between functions and classes
- Insert 1 blank line between methods within a class

```python
# Good example
def first_function():
    """First function."""
    pass


def second_function():
    """Second function."""
    pass


class MyClass:
    """Sample class."""

    def method_one(self):
        """First method."""
        pass

    def method_two(self):
        """Second method."""
        pass
```

### Operators and Spaces

- Insert spaces around operators and after commas
- Do not insert spaces immediately after opening brackets

```python
# Recommended
result = function(arg1, arg2) + another_function(arg3, arg4)

# Not recommended
result=function( arg1,arg2 )+another_function( arg3,arg4 )
```

## Naming Conventions

| Target | Convention | Example |
|--------|------------|---------|
| Class names | `UpperCamelCase` | `MyClass`, `DataProcessor` |
| Function/method names | `lowercase_with_underscores` | `my_function`, `calculate_total` |
| Variable names | `lowercase_with_underscores` | `user_name`, `total_count` |
| Constants | `UPPER_CASE_WITH_UNDERSCORES` | `MAX_SIZE`, `DEFAULT_TIMEOUT` |
| Module names | `lowercase` (underscores allowed) | `mymodule`, `data_parser` |
| Private attributes | `_leading_underscore` | `_internal_value` |

```python
# Recommended
class UserManager:
    """User management class."""

    MAX_USERS = 1000  # Constant

    def __init__(self):
        self._users = []  # Private attribute

    def add_user(self, user_name: str) -> bool:
        """Add a user."""
        if len(self._users) < self.MAX_USERS:
            self._users.append(user_name)
            return True
        return False
```

### Method Argument Names

- Use `self` for the first argument of instance methods
- Use `cls` for the first argument of class methods

## Import Best Practices

### Import Order

1. Standard library imports
2. Third-party library imports
3. Local module imports

Insert a blank line between each group.

```python
# Recommended
import os
from typing import List

import numpy as np
import requests

from myproject.core import database
```

### Import Restrictions

- Do not use wildcard imports (`from module import *`)

**Reason**: Namespace pollution, difficulty detecting undefined names, unclear dependencies

### Import Placement

- Place imports at the top of the module
- Use imports within functions or classes only to avoid circular dependencies

## Functions and Documentation

### Writing Docstrings

- Write docstrings for all public functions, methods, and classes

```python
def calculate_sum(numbers: List[int]) -> int:
    """Calculate the sum of a list of numbers.

    Args:
        numbers: List of integers to sum

    Returns:
        Sum of all numbers

    Raises:
        ValueError: If the list is empty
    """
    if not numbers:
        raise ValueError("List is empty")
    return sum(numbers)
```

### Docstring Structure

- Line 1: Concise summary starting with a capital letter and ending with a period
- Line 2: Blank line
- Line 3 onwards: Describe Args, Returns, and Raises

### Type Hints

- Use type hints actively

```python
from typing import List, Dict, Optional, Union

def process_users(
    user_ids: List[int],
    options: Optional[Dict[str, str]] = None
) -> Dict[int, str]:
    """Process user IDs and return user information.

    Args:
        user_ids: List of user IDs to process
        options: Optional parameter dictionary

    Returns:
        Dictionary with user IDs as keys and user names as values
    """
    if options is None:
        options = {}

    return {user_id: f"User_{user_id}" for user_id in user_ids}
```

### Default Arguments

- Do not use mutable objects (lists, dictionaries, etc.) as default values

```python
# Not recommended
def append_to_list(item, my_list=[]):
    my_list.append(item)
    return my_list

# Recommended
def append_to_list(item, my_list=None):
    if my_list is None:
        my_list = []
    my_list.append(item)
    return my_list
```

**Reason**: Default values are evaluated only once at definition time and can cause unexpected behavior

## Error Handling

### Handle Specific Exceptions

- Catch specific exception classes
- Avoid broad `Exception` catches

```python
# Recommended
try:
    with open('config.json', 'r') as f:
        config = json.load(f)
except FileNotFoundError:
    logger.error("Configuration file not found")
    config = get_default_config()
except json.JSONDecodeError as e:
    logger.error(f"JSON parse error: {e}")
    raise
except Exception as e:
    logger.error(f"Unexpected error: {e}")
    raise
```

### else and finally Clauses

- Use the else clause to describe processing when no exception occurs
- Use the finally clause to always execute cleanup processing

### Resource Management with with Statements

- Use with statements for resources such as files, database connections, and locks

```python
# Recommended
with open('file.txt', 'r') as f:
    content = f.read()
    process(content)
```

### Custom Exceptions

- Inherit from `Exception`
- End the name with "Error"

```python
class UserNotFoundError(Exception):
    def __init__(self, user_id: int):
        self.user_id = user_id
        super().__init__(f"User ID {user_id} not found")
```

### Exception Chaining

- Use the `from e` syntax for exception chaining to clarify the cause

## Data Structures and Algorithms

### String Concatenation

- Use `str.join()` to concatenate multiple strings
- Avoid `+=` concatenation in loops

```python
# Recommended
result = ', '.join(str(item) for item in items)

# Not recommended
result = ''
for item in items:
    result += str(item) + ', '
```

### List Comprehensions and Generator Expressions

- Use list comprehensions for concise loops
- Use generator expressions for large amounts of data
- Utilize dictionary and set comprehensions

```python
squares = [x**2 for x in range(10)]
squares_gen = (x**2 for x in range(10))  # Memory efficient
even_squares = [x**2 for x in range(10) if x % 2 == 0]
```

### Choosing Appropriate Data Structures

| Data Structure | Use Case | Characteristics |
|----------------|----------|-----------------|
| `list` | Ordered collection | Mutable, index access |
| `tuple` | Immutable ordered collection | Immutable |
| `set` | Collection of unique elements | Fast search, unordered |
| `dict` | Key-value pairs | Fast search |
| `collections.deque` | Double-ended queue | Fast operations at both ends |
| `collections.defaultdict` | Dictionary with default values | Safe even when keys are missing |
| `collections.Counter` | Counter | Tallies element occurrences |

```python
from collections import defaultdict, Counter, deque

user_groups = defaultdict(list)
user_groups['admin'].append('Alice')

word_counts = Counter(['apple', 'banana', 'apple'])

queue = deque([1, 2, 3])
queue.append(4)
```

## Classes and Object-Oriented Programming

### Class Attributes and Instance Attributes

- Update class attributes explicitly with `ClassName.attribute`
- Define instance attributes within `__init__` as `self.attribute`

### Using Properties

- Use the `@property` decorator to control attribute access

```python
class User:
    """User class."""

    def __init__(self, name: str, age: int):
        self._name = name
        self._age = age

    @property
    def name(self) -> str:
        """Get user name."""
        return self._name

    @property
    def age(self) -> int:
        """Get age."""
        return self._age

    @age.setter
    def age(self, value: int) -> None:
        """Set age (with validation)."""
        if value < 0:
            raise ValueError("Age must be 0 or greater")
        self._age = value
```

### Inheritance and Instance Checking

- Use `isinstance()` for type checking
- Use `issubclass()` for subclass checking

## Performance and Optimization

### General Principles

- Prioritize algorithm optimization
- Choose appropriate data structures (utilize the `collections` module)
- Leverage built-in functions (fast C implementations)
- Identify bottlenecks through profiling (`cProfile`, `timeit`)
- Avoid excessive abstraction

```python
# Recommended
total = sum(numbers)
squares = [x**2 for x in range(1000)]
```

### Measurement and Profiling

- Use `timeit` for execution time measurement
- Use `cProfile` for bottleneck identification

## Logging

- Use the `logging` module instead of `print()`

```python
import logging

# Logger configuration
logger = logging.getLogger(__name__)
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

# Recording logs
logger.debug('Debug information')
logger.info('Starting process')
logger.warning('Warning: Configuration not found')
logger.error('An error occurred')
logger.critical('Critical error')
```

### Log Level Usage

| Level | Usage |
|-------|-------|
| DEBUG | Detailed diagnostic information (during development) |
| INFO | General informational messages |
| WARNING | Warning messages (works but requires attention) |
| ERROR | Error messages (some functionality does not work) |
| CRITICAL | Critical errors (program continuation difficult) |

## Security

### Input Validation

- Do not use `eval()` (security risk)
- Use `ast.literal_eval()` for literal evaluation

```python
# Recommended
import ast
user_input = "[1, 2, 3]"
safe_value = ast.literal_eval(user_input)
```

### Managing Sensitive Information

- Use the `secrets` module for random number generation
- Use `hashlib.pbkdf2_hmac` for password hashing

### Safe File Path Handling

- Use `pathlib` to construct paths
- Prevent path traversal with `Path.is_relative_to()`

## Code Quality Tools

### Recommended Tools

- Black: Automatic code formatting
- isort: Automatic import organization
- flake8: PEP 8 compliance checking
- mypy: Static type checking

### Pre-commit Hook Configuration

- Configure Black, isort, and flake8 in `.pre-commit-config.yaml`

## Checklist

### Style and Formatting
- [ ] Use 4-space indentation
- [ ] Limit line length to 79 characters
- [ ] Follow naming conventions
- [ ] Correct import order
- [ ] No wildcard imports

### Documentation
- [ ] Docstrings for public functions, methods, and classes
- [ ] Args, Returns, and Raises described in docstrings
- [ ] Type hints used

### Error Handling
- [ ] Specific exception handling
- [ ] Resource management with with statements
- [ ] Proper exception handling and re-raising

### Code Quality
- [ ] No mutable objects in default arguments
- [ ] Use logging module
- [ ] No `eval()` usage
- [ ] Appropriate data structure selection

## Reference Resources

- [PEP 8 -- Style Guide for Python Code](https://peps.python.org/pep-0008/)
- [Python Official Documentation](https://docs.python.org/ja/3/)
- [The Zen of Python (PEP 20)](https://peps.python.org/pep-0020/)
- [Type Hints (PEP 484)](https://peps.python.org/pep-0484/)
