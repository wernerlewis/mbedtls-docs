# Automating test generation

For coverage, test functions may require a large amount of test cases, which may be added manually calculating results for each. This often is done by writing a script each time, or by different contributors with different scripts. When adding new tests, this requires either manually adding extra tests, or updating a personal script. It is also common for multiple tests to use the same inputs for their test cases, which may lead to a large amount of copying, which is error-prone.

When these cases are changed in future, the script used may need to be rewritten, unless it has been kept locally or in commits. Alternatively, tests may be updated manually, causing inputs to no longer be consistent across tests, which could mean some corner cases are missed.

Mbed TLS includes a Python framework for test case generation. This provides a standard interface to generate test cases automatically, which can be ran when running tests. This allows for tests to be updated and added more easily.

## Framework components

Test case generation is based on the following components:
 - BaseTarget: a class declaring common attributes and methods necessary to implement a test class, as well as defining methods used in test generation.
 - File Targets: child classes of `BaseTarget` which set an output file for tests. These define the `TARGETS` which can be passed to the script on command line, allowing generation of specific files.
 - Function Targets: classes derived from a File Target, which generate test cases for a function.
 - Abstract Targets: classes derived from a File Target, which provide common attributes and methods for Function targets derived from this class.
 - Command line interface: command line options and features for writing generated cases to file.

### BaseTarget

This class declares the common attributes and methods necessary for implementing test cases, is derived from when constructing tests using the framework. The class is documented in `scripts/mbedtls_dev/test_generation.py`.

`BaseTarget` includes the methods `generate_function_tests()` and `generate_tests()`. These implement the test generation part of the framework, for the current target class only, and for this class and all subclasses, respectively.

### File Targets

File Targets are child classes of `BaseTarget` which represent a generated data file, i.e `XYZTarget` could represent `xyz.generated.data`. `target_basename` must be set in these classes, and they can also implement other common features as required.

`FileTarget.generate_tests()` generates all `TestCase` objects for the output file.

### Test Function Targets

These are classes derived from a File Target, which represent a test function. This requires all abstract methods of `BaseTarget` to be implemented. Test function targets can also be subclassed, for similar implementations, such as absolute variants of an operation.

### Abstract Targets

Abstract targets do not represent a test function. These may be used to provide common attributes or methods, without declaring a specific test function to be the parent. An example can be found in `tests/scripts/generate_bignum_tests.py` `BignumOperation`.

### Command line interface

`TestGeneration.main()` is used as the command line entry point for the framework. This provides options for generating output for all or specific File Targets, and options to only list existing targets. The interface also manages File Targets and writing of test cases via `TestGeneration.TestGenerator`. These features are also used for `tests/scripts/generate_psa_tests.py`, which does not use the same framework.

## Adding new tests

There are two options for adding new tests:
 - Editing an existing script, i.e. `tests/scripts/generate_bignum_tests.py`.
 - Adding a new script.

If adding a new script, files will not automatically be generated when building tests, unless the build system is updated.

### Built in test generation

When adding a new script to the build system, it is generally helpful to look at the implementation for `generate_bignum_tests.py`. This will show where changes are needed and how targets/commands should be added.

The following changes are required:
 - Windows: Add script to `/scripts/make_generated_files.bat`.
 - Checks: Add script to `/tests/scripts/check-generated-files.sh`.
 - Make: Add targets `GENERATED_XYZ_DATA_FILES` and  to `tests/Makefile`, and append to `GENERATED_FILES`.
 - Make: Add target `generated_xyz_test_data` to `tests/Makefile`, with required dependencies. `GENERATED_XYZ_DATA_FILES` will depend on this target.
 - CMake: Add `execute_process` for the script with `--list-for-cmake`
 - CMake: Append files to `xyz_generated_data_files`.
 - CMake: Add custom command to run the script.
 - CMake: Add `test_suite_xyz_generated_data` target.
 - CMake: Update `dependency` to be set correctly for generated data files in the new script.

## Creating a new test script

This is only required if an existing `tests/scripts/generate_xyz_tests.py` script is not being extended. This example will show the basic requirements for test case generation for functions defined in `test_suite_mpi.function`.

### Setting up Python script

To create the script, `test_generation` must be imported from `mbedtls_dev` and `__main__` should be set to call `test_generation.main()`. The `scripts/` directory should be added to the system path before importing; if creating the script in `tests/scripts/` this can be done by importing `scripts_path`.

Example if file in `tests/scripts/` directory:
```python
import sys

import scripts_path
from mbedtls_dev import test_generation

if __name__ == '__main__':
    test_generation.main(sys.argv[1:])
```
If file in `mbedtls/` directory:
```python
import os
import sys

sys.path.append(os.path.join(os.path.dirname(__file__), "scripts"))
from mbedtls_dev import test_generation

if __name__ == '__main__':
    test_generation.main(sys.argv[1:])
```

To run the script:
```sh
$ python tests/scripts/generate_xyz_tests.py
```
Running this will generate output files for all File Targets derived from `BaseTarget`. As we have not added any classes yet, nothing will be generated.

### Adding a File Target

To add a File Target, create a subclass of `BaseTarget` to represent the file test cases will be written to. Typically `target_basename` should be set to `test_suite_xyz.generated`, where `test_suite_xyz.function` contains the functions we are creating test cases for.

For this example the following is added, after imports. Note that `metadata=ABCMeta` is not required, but just explicitly indicates that this is an abstract class.

```python
class BignumTarget(test_generation.BaseTarget, metaclass=ABCMeta):
    """Target for bignum test case generation."""
    target_basename = "test_suite_mpi.generated"
```

Running the script will now create the `test_suite_mpi.generated.data` file, and running with `--list` will display the target name. The file will contain no test cases at this point, but we can now use this for generating tests for functions in `test_suite_mpi.function`

## Adding test case generation for a function

To add test cases for a function, a function target should be added to your generate script. Function targets are subclasses of a file target, which generate test cases and write them to the specified file. The example used is a class for creating bignum addition tests for `mpi_add_mpi`. This will use `BignumTarget` as the file target, which outputs to `test_suite_mpi.generated.data`.

Note: typing will be used in the following snippets, for clarity. This will require the types to be imported from `typing` if used.

### Creating the class

The class must derive from a file target, in this example we will derive directly from `BignumTarget`. The following class attributes should be set:
 - `count = 0`: this resets the test case counting mechanism, but can be ignored if cases are not to be numbered. If ignoring, `descriptions()` should be overloaded.
 - `test_function`: the function in the `.function` file we are generating tests for.
 - `test_name`: a short descriptive name for the tests to be generated, this can be `test_function` if appropriate. Forms the first part of the description line.

```
class BignumAdd(BignumTarget):
    """Test cases for bignum addition."""
    count = 0
    test_function = "mpi_add_mpi"
    test_name = "MPI add"
```

### Class initialization

To initialize an instance of the class, we require input values. For our example, we will have two input values, `val_l` and `val_r`. For calculating the result, these inputs are converted to `int`, as this is a multi-precision type in Python. To allow us to include cases such as zero-length zeros (`""`), we should use strings for the inputs, and store the inputs as-is. This avoids issues where the internal representation differs, and ensures cases are correctly passed.

```python
    def __init__(self, val_l: str, val_r: str) -> None:
        self.arg_l = val_l
        self.arg_r = val_r
        # int (val, 16) converts from a hex string to int
        # null strings should be read as zero
        # a if x else b is equivalent to x ? a : b 
        self.int_l = int(val_l, 16) if val_l else 0
        self.int_r = int(val_r, 16) if val_r else 0
```

We now have both values in `arg_` form, as strings, as well as `int_` forms. Alternatively, the result can be calculated directly during initialization. However, to keep output separate from input, a `result()` method will be added to do this.

### Implementing BaseTarget methods

Abstract methods from `BaseTarget` will need to be implemented in the class. These are general purpose methods which will always be required for tests.

`arguments()` returns the list of arguments required by the test function. Any C-string arguments must be quoted. For our `MPI add` tests:

```python
    def arguments(self) -> List[str]:
        return [
            "\"{}\"".format(self.arg_l),
            "\"{}\"".format(self.arg_r),
            self.result()
        ]
```
This will require the `result()` method to be implemented. The format `{:x}` converts from `int` to a hexadecimal string representation, without leading `0x`.

```python
    def result(self) -> str:
        return "\"{:x}\"".format(self.int_l + self.int_r)
```

`generate_function_tests()` handles the creation each instance of the class, and is responsible for:
 - Initializing class instances with required inputs.
 - Yielding `test_case.TestCase` objects. This is done by calling `.create_test_case()` on the class instance.

Note: this is a `classmethod`, and acts on the class, rather than a class instance.

For our example, we will use a list of strings for our inputs, and create a test case for each combination. To avoid nested loops and repeated pairs of inputs, such as ("0", "1") and ("1", "0"), the module `itertools` can be used.

```python
import itertools
    
    {...}

    @classmethod
    def generate_function_tests(cls) -> Iterator[test_case.TestCase]:
        # Create a list of input values
        input_values = ["", "0", "1", "123"]
        # combinations_with_replacement will create all unique combinations,
        # including where inputs are the same i.e. ("", "")
        for l_value, r_value in itertools.combinations_with_replacement(
            input_values, 2
        )
            # Initialize our instance with these values
            class_instance = cls(l_value, r_value)
            # yield the TestCase object for this instance
            yield class_instance.create_test_case()
```

### Other BaseTarget attributes

`case_description` describes a particular test case. If not set it will be ignored, however it can be used to add context for each test case. By default the full description will be set as:

`test_name` #`count` `case_description`

In some corner case tests, you may want the option to explicitly pass a case description, to inform a reader what the goal of the test is. Alternatively it can be generated in a standard way when not set. For bignum addition, we may want to display the calculation by overloading the `description()` method:

```python
    def description(self) -> str:
        if not self.case_description:
            self.case_description = "{} + {}".format(
                # To avoid blank spaces with input ""
                "0 (null)" if self.arg_l == "" else self.arg_l,
                "0 (null)" if self.arg_r == "" else self.arg_r
            )
        # Call description in the parent class
        return super().description()
```

`dependencies` is a list of macros required for the test case. This can be set in the class if a test function always requires a macro, or conditionally set during initialization or in result calculation, as dependencies for a test case are set last in `create_test_case()`. Avoid using `.append()` in the first use of `dependencies` in an instance, as this will modify the class, instead set it to a new list.

## Example test script

The complete example test script:

```python
import itertools
import sys

import scripts_path
from mbedtls_dev import test_generation


class BignumTarget(test_generation.BaseTarget):
    """Target for bignum test case generation."""
    target_basename = 'test_suite_mpi.generated'


class BignumAdd(BignumTarget):
    """Test cases for bignum addition."""
    count = 0
    test_function = "mpi_add_mpi"
    test_name = "MPI add"

    def __init__(self, val_l: str, val_r: str) -> None:
        self.arg_l = val_l
        self.arg_r = val_r
        # int (val, 16) converts from a hex string to int
        # null strings should be read as zero
        # a if x else b is equivalent to x ? a : b 
        self.int_l = int(val_l, 16) if val_l else 0
        self.int_r = int(val_r, 16) if val_r else 0

    def arguments(self):
        return [
            "\"{}\"".format(self.arg_l),
            "\"{}\"".format(self.arg_r),
            self.result()
        ]

    def description(self) -> str:
        if not self.case_description:
            self.case_description = "{} + {}".format(
                "0 (null)" if self.arg_l == "" else self.arg_l,
                "0 (null)" if self.arg_r == "" else self.arg_r
            )
        # Call description in the parent class
        return super().description()

    def result(self) -> str:
        return "\"{:x}\"".format(self.int_l + self.int_r)

    @classmethod
    def generate_function_tests(cls):
        # Create a list of input values
        input_values = ["", "0", "1", "123"]
        # combinations_with_replacement will create all combinations,
        # including where inputs are the same i.e. ("", "") 
        for l_value, r_value in itertools.combinations_with_replacement(
            input_values, 2
        ):
                # Initialize our instance with these values
                class_instance = cls(l_value, r_value)
                # yield the TestCase object for this instance
                yield class_instance.create_test_case()


if __name__ == '__main__':
    test_generation.main(sys.argv[1:])
```

Running this script will now create 10 test cases in `test_suite_mpi.generated.data`, which can be built with `make check`. The test file can then be ran as `./test_suite_mpi.generated` from the `tests/` directory.

## Extending existing test classes

Some test functions may be added which are similar to other functions which are already tested. This may be a very similar or variant function, such as `mpi_add_mpi` and `mpi_add_mpi_abs`, the latter of which is absolute. Other similarities may be in formatting or inputs - there are a large number of operations which take inputs A, B and output X only.

### Variants

For variants of an existing test class, the existing class can be subclassed. This allows only the necessary changes to be made. For example, with `mpi_add_mpi_abs`, we can use the existing `BignumAdd` class, and the only necessary changes are:
 - `count` should be reset to 0.
 - `test_function` should be set.
 - `test_name` should be set.
 - int forms of the input `int_l` and `int_r` should be converted to absolute.

```python
class BignumAddAbs(BignumAdd):
    """Tests for absolute variant of bignum additions."""
    count = 0
    test_function = "mpi_add_mpi_abs"
    test_name = "MPI add (abs)"

### Either overload __init__()
    def __init__(self, val_l, val_r) -> None:
        super().__init__(val_l, val_r)
        self.int_l = abs(self.int_l)
        self.int_r = abs(self.int_r)

### Or overload result()
    def result(self) -> str:
        return "\"{:x}\"".format(abs(self.int_l) + abs(self.int_r))
```

These tests will be generated with the same input values as `mpi_add_mpi`.

### Using abstract targets to group similar functions

In some tests there are many common features, but the functions are not fundamentally similar, and it can become confusing to overload in the same way. For example, multiplication will use similar `__init__` and `arguments()` to `BignumAdd`, but `result()` and `description()` will vary. This may lead to confusion as more unrelated test classes share a parent.

In these cases, abstract target can be used. These derive from a File Target, but do not have a `test_function` set. Instead, these provide common functions and attributes, which may be abstract or defined. To implement this, we can identify common and similar features, and implement these to support more test functions.

For example, in `tests/scripts/generate_bignum_test.py`, `BignumOperation` provides the following implemented attributes/methods:
 - `symbol` a symbol representing the operation to use in the `case_description`. This replaces a hard-coded symbol in `
 - `input_values` a list of commonly used individual input values.
 - `input_cases` a list of pairs of input values, which can be used to test specific corner cases.
 - `arguments()` using the common `arg_l`, `arg_r`, `result()` format.
 - `description()` adding `case_description` generation
 - `value_description()` a method to convert an input value into a descriptive form.
 - `get_value_pairs()` a common generator for yielding input pairs, from `input_cases` and from combinations of `input_values`.
 - `generated_function_tests()` which now uses `get_value_pairs` for its inputs.

 Additionally the abstract `result()` method is to be defined by the function target.


