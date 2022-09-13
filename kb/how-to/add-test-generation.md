# Generating Test Cases

Mbed TLS includes a Python framework for test case generation. This systematically generates test cases when building tests, and allows cases to be added or modified with fewer changes to the codebase.

## Components of the framework

The framework is based around classes derived from `BaseTarget`, referred to as Target classes. These are used to define how test cases are generated, and the destination file for generated tests. A command line interface is used to list and generate test data files using the framework.

### BaseTarget

This class declares the common attributes and methods necessary for generating test cases, and is derived from for all other Target classes. This class is documented in `scripts/mbedtls_dev/test_generation.py`.

The method `generated_tests()` is defined to generate tests for the current class, and all classes derived from it.

### File Targets

File Targets are child classes of `BaseTarget` which represent a generated data file. It is required that `target_basename` is set in these classes, typically in the form `"xyz.generated"` for a corresponding `"xyz.function"` file. File Targets may also implement other common attributes or methods for test functions in the associated file, as required.

Calling `generated_tests()` on a File Target will yield all `TestCase` objects for the corresponding data file, "`target_basename`.data".

### Function Targets

Function Targets are derived from a File Target, and represent a test function. These require all abstract methods of `BaseTarget` to be implemented, and generates test case data for the represented function. The parent class of a Function Target can be a File Target, an Abstract Target, or another Function Target.

### Abstract Targets

Abstract Targets are derived from a File Target, and do not represent a test function. These may be used to provide common attributes or methods for Function Targets derived from the class. An example is `BignumOperation` in `tests/scripts/generate_bignum_tests.py`.

### Command line interface

`TestGeneration.main()` is the command line entry point for the framework. This provides options for generating all, or specified, data files. The interface also handles creating a list of generated output files (`TARGETS`) and writing of test cases to file via `TestGeneration.TestGenerator`. These features are also used in `tests/scripts/generate_psa_tests.py`, which does not use the same Target framework.

## Adding new tests

There are two options for adding new tests:
 - Editing an existing script such as `tests/scripts/generate_bignum_tests.py`.
 - Adding a new script.

If adding a new script, files will not automatically be generated when building tests, unless the build system is updated.

### Adding a new script to the build system

When adding a new script to the build system, the script will need to be added similarly to `generate_bignum_tests.py`, requiring the following:
 - Add script to `/scripts/make_generated_files.bat`.
 - Add script to `/tests/scripts/check-generated-files.sh`.
 - For Make and CMake: Add calls/targets for the new script where `generate_bignum_tests.py` is currently called.

## Creating a new test script

This is only required if an existing script is not being extended. This example will show the steps for creating a test generation script for `test_suite_mpi.function`, similarly to `generate_bignum_tests.py`.

### Setting up Python script

To create the script, import `test_generation` from `mbedtls_dev` and call `test_generation.main()` as `__main__`. The `scripts/` directory must be added to the system path before importing from `mbedtls_dev`, either manually or by importing `scripts_path` in `tests/scripts/`.

Default example, with script in `tests/scripts/`:
```python
import sys

import scripts_path
from mbedtls_dev import test_generation

if __name__ == '__main__':
    test_generation.main(sys.argv[1:])
```
Alternative example, with script in `mbedtls/`:
```python
import os
import sys

sys.path.append(os.path.join(os.path.dirname(__file__), "scripts"))
from mbedtls_dev import test_generation

if __name__ == '__main__':
    test_generation.main(sys.argv[1:])
```

To run the script from `mbedtls/`:
```bash
$ python tests/scripts/generate_xyz_tests.py
```
Running the script will generate output files for all defined File Targets. As none are defined, nothing will be generated.

### Adding a File Target

To add a File Target, create a subclass of `BaseTarget`, with `target_basename` set to `test_suite_xyz.generated`, where `test_suite_xyz.function` contains the functions we are creating test cases for.

For the example, add after imports:
```python
class BignumTarget(test_generation.BaseTarget, metaclass=ABCMeta):
    #pylint: disable=abstract-method
    """Target for bignum test case generation."""
    # Using .gen in the example to avoid clashing with `generate_bignum_tests.py`
    target_basename = "test_suite_mpi.gen"
```
Note, `metaclass=ABCMeta` and `pylint` comments are not required, but are added to explicitly indicate that this is an abstract class.

Running the script will now create the `test_suite_mpi.gen.data` file, and running with `--list` will list this file as the available `TARGET`. The file will contain no test cases at this point.

## Adding test case generation for a function

To add test cases for a function, a Function Target should be added to the script. Function Targets are derived from a File Target, and generate test cases to be written to the target file. The example used is for creating bignum addition tests using the function `mpi_add_mpi`. This will use `BignumTarget` as the File Target.

Note: typing will be used in the following snippets, for clarity. This will require the types to be imported from `typing` if used.

### Creating the class

The class must derive from a File Target, in this example we will derive directly from `BignumTarget`. The following class attributes should be set:
 - `count = 0`: this resets the test case counting mechanism. This can be ignored if case numbering is not needed for test cases. In this case, `description()` should be redefined to remove the count.
 - `test_function`: the test function we are generating cases for.
 - `test_name`: a short descriptive name for the test, this may be `test_function` if appropriate. Forms the first part of the description line.

For the example, add after BignumTarget:
```python
class BignumAdd(BignumTarget):
    """Test cases for bignum addition."""
    count = 0
    test_function = "mpi_add_mpi"
    test_name = "MPI add"
```

### Class initialization

To initialize an instance of the class, we require input values. For the example, two input values are used, `val_a` and `val_b`. As there are possible inputs such as zero-length zeros (`""`), strings are used as inputs, and are stored as-is. This ensures the values are correctly passed in test cases. To calculate the expected result, these inputs are converted and stored in `int` forms, as this is a multi-precision type in Python.

For the example:
```python
    def __init__(self, val_a: str, val_b: str) -> None:
        self.arg_a = val_a
        self.arg_b = val_b
        # int(val, 16) converts from a hex string to int
        # null strings should be treated as zero in calculations
        self.int_a = int(val_a, 16) if val_a else 0
        self.int_b = int(val_b, 16) if val_b else 0
```

The class now stores values as strings, and as multi-precision integers. Alternatively, the expected result could be calculated during initialization. However, to keep output separate from input, a `result()` method will be used for calculation. For addition:
```python
    def result(self) -> str:
        # Return as a quoted hexadecimal string, without leading 0x
        return "\"{:x}\"".format(self.int_a + self.int_b)
```

### Implementing BaseTarget methods

Abstract methods declared in `BaseTarget` must be implemented in the class.

`arguments()` returns the list of arguments required by the test function. Any C-string arguments must be quoted. For the `MPI add` tests:
```python
    def arguments(self) -> List[str]:
        return [
            "\"{}\"".format(self.arg_a),
            "\"{}\"".format(self.arg_b),
            self.result()
        ]
```

`generate_function_tests()` handles the initialization of class instances with required inputs, and yielding of `test_case.TestCase` objects from each instance. This is a `classmethod`, and acts on the class, rather than a specific instance.

In the example, a list of strings is used for inputs, and a test case is generated for each combination. To avoid nested loops and repeated pairs of inputs, such as ("0", "1") and ("1", "0"), the standard module `itertools` is used.

```python
import itertools 
...
    @classmethod
    def generate_function_tests(cls) -> Iterator[test_case.TestCase]:
        # Create a list of input values
        input_values = ["", "0", "1", "123"]
        # combinations_with_replacement generates a list of all unique combinations,
        # including where inputs are the same i.e. ("0", "0")
        for a_value, b_value in itertools.combinations_with_replacement(
            input_values, 2
        )
            # Initialize our instance with these values
            class_instance = cls(a_value, b_value)
            # yield the TestCase object for this instance
            yield class_instance.create_test_case()
```

### Other BaseTarget attributes and methods

`description()` returns a string to describe the test case, and is defined in `BaseTarget` to return the string "`test_name` #`count` `case_description`", where `case_description` is an optional string describing the specific test case. For some test cases, it may be beneficial to set `case_description` to inform a reader of the purpose of the case. This can also be generated when not explicitly set, to always include context. For bignum addition, the calculation can be used as the case description, by overriding the `description()` method:

```python
    def description(self) -> str:
        if not self.case_description:
            self.case_description = "{} + {}".format(
                # hex() converts to a hex string with leading 0x
                "0 (null)" if self.arg_a == "" else hex(self.int_a),
                "0 (null)" if self.arg_b == "" else hex(self.int_b)
            )
        # Call description in the parent class
        return super().description()
```

`dependencies` is a list of macros required for the test case to run. This can be always set in the class if a macro is always required, or conditionally set in the class. For example, if a test function depends on `MBEDTLS_GENPRIME`, add `dependencies = ["MBEDTLS_GENPRIME"]` to the class.

## Complete Example Test Script

The full test script from the example snippets:

```python
import itertools
import sys

from typing import List, Iterator

import scripts_path
from mbedtls_dev import test_generation


class BignumTarget(test_generation.BaseTarget, metaclass=ABCMeta):
    #pylint: disable=abstract-method):
    """Target for bignum test case generation."""
    target_basename = 'test_suite_mpi.generated'


class BignumAdd(BignumTarget):
    """Test cases for bignum addition."""
    count = 0
    test_function = "mpi_add_mpi"
    test_name = "MPI add"

    def __init__(self, val_a: str, val_b: str) -> None:
        self.arg_a = val_a
        self.arg_b = val_b
        # int(val, 16) converts from a hex string to int
        # null strings should be treated as zero in calculations
        self.int_a = int(val_a, 16) if val_a else 0
        self.int_b = int(val_b, 16) if val_b else 0

    def arguments(self) -> List[str]:
        return [
            "\"{}\"".format(self.arg_a),
            "\"{}\"".format(self.arg_b),
            self.result()
        ]

    def description(self) -> str:
        if not self.case_description:
            self.case_description = "{} + {}".format(
                # hex() converts to a hex string with leading 0x
                "0 (null)" if self.arg_a == "" else hex(self.int_a),
                "0 (null)" if self.arg_b == "" else hex(self.int_b)
            )
        # Call description in the parent class
        return super().description()

    def result(self) -> str:
        # Return as a quoted hexadecimal string, without leading 0x
        return "\"{:x}\"".format(self.int_a + self.int_b)

    @classmethod
    def generate_function_tests(cls) -> Iterator[test_case.TestCase]:
        # Create a list of input values
        input_values = ["", "0", "1", "123"]
        # combinations_with_replacement will create all combinations,
        # including where inputs are the same i.e. ("", "") 
        for a_value, b_value in itertools.combinations_with_replacement(
            input_values, 2
        ):
                # Initialize our instance with these values
                class_instance = cls(a_value, b_value)
                # yield the TestCase object for this instance
                yield class_instance.create_test_case()


if __name__ == '__main__':
    test_generation.main(sys.argv[1:])
```

Running this script should create 10 test cases in `test_suite_mpi.generated.data`. As this clashes with the `generate_bignum_tests.py` output, you may want to replace this file with this script. The test file will then be generated from this script, and can be ran with `make check`.

## Extending existing test classes

Some test functions may need to be added which are similar to other functions which are already tested. This may be a very similar or variant function, such as `mpi_add_mpi` and `mpi_add_mpi_abs`, the latter of which is absolute. Other similarities may be in formatting or inputs, such as the variety of test functions for operations which take inputs A, B, and output X only.

### Variants

For variants of a test function, inheritance can be used to reduce repetition of code. For example, the test function `mpi_add_mpi_abs` has similar arguments and results to `mpi_add_mpi`, so a new Function Target (`BignumAddAbs`) can inherit from the existing `BignumAdd` class. The necessary changes for the new Target are then:
 - `count` should be reset to 0 for the class.
 - `test_function` should be set to `mpi_add_mpi_abs`.
 - `test_name` should be set.
 - `int` forms of the inputs (`int_a` and `int_b`) should be made absolute.

```python
class BignumAddAbs(BignumAdd):
    """Tests for absolute variant of bignum addition."""
    count = 0
    test_function = "mpi_add_mpi_abs"
    test_name = "MPI add (abs)"

### Either override __init__()
    def __init__(self, val_a, val_b) -> None:
        super().__init__(val_a, val_b)
        self.int_a = abs(self.int_a)
        self.int_b = abs(self.int_b)

### Or override result()
    def result(self) -> str:
        return "\"{:x}\"".format(abs(self.int_a) + abs(self.int_b))
```

These tests will then be generated with the same input values as `mpi_add_mpi`, and use the same methods for generating descriptions and the list of arguments.

### Using Abstract Targets
  
Abstract Targets are derived from File Targets, and are used to define common methods and attributes for multiple Function Targets. This can reduce repetition of code, and provide a common structure to conform to when adding test generation for similar test functions.

For example, in `tests/scripts/generate_bignum_test.py`, `BignumOperation` adds features which are common in binary bignum operation tests. It implements the following attributes/methods:
 - `symbol`: a symbol to represent the operation in the `case_description`.
 - `input_values` a list of common individual input values.
 - `input_cases` a list of pairs of input values, which can be used to test specific corner cases.
 - `__init__()` storing string and int forms of the input values.
 - `arguments()` using the common `arg_a`, `arg_b`, `result()` format.
 - `description()` adding `case_description` generation, using `symbol` in place of a hard coded symbol.
 - `get_value_pairs()` yielding input pairs, from `input_cases` and from combinations of `input_values`.
 - `generated_function_tests()` using `get_value_pairs` for its inputs.
 - `result()` an abstract method to generate the test result.

By deriving from `BignumOperation`, tests can be generated for binary bignum operations with simple Function Targets, which may only require class attributes to be set and `result()` to be defined. `BignumAdd` and `BignumCmp` are examples in `generate_bignum_test.py`.

