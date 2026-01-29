# Test Policy

## Github Actions

### Running Unit Tests

The way the project was developed make it so that every time a pull request is closed on the main branch (which is dev), then unit tests are ran for each module.

```
on:
  pull_request:
    types: [ closed ]
    branches: [ "dev" ]
```

As the project is an ISO file meant to be run in Qemu, some parts can't be tested.  
For those that can, the unit tests are built on a ubuntu virtual machine.

```
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      with:
          submodules: true
          token: ${{ secrets.EIP_EPITECH_TOKEN }}
```

Submodules are then pulled.

```
    - name: Pull submodules
      run: |
        git submodule update --init --recursive
      env:
        GITHUB_ACTOR: ${{ github.EIP_EPITECH_TOKEN }}
```

The dependencies required by the project are installed in this Linux environment.

```
    - name: Install lightweight dependencies
      run: |
        sudo apt-get update
        sudo apt-get install -y nasm gcc make
```

Makefiles of this project contain a TESTING variable to differentiate between the build process of the ISO and the build process of the unit tests.

```
    - name: Build and run tests
      run: |
        make TESTING=1 test_run

    - name: Run tests
      run: |
        ./test_run
```

## Unit Testing using the Unity Framework

The Unity Framework was chosen for its ease of use and good integration with the project.
To make use of it:
- the unity.h and unity_internal.h files must be imported by unit test binaries.
- the unity.c file must be compiled with unit test binaries.

Each unit test binary has to define a setUp() and tearDown() function.
This project defines them this way:

```
void setUp(void) {}
void tearDown(void) {}
```

A Unit Test binary must use the UNITY_BEGIN macro to initialise the Unity Framework and the main function must return UNITY_END.
Then, a Unit Test can be run using the RUN_TEST Macro.  
The RUN_TEST Macro accepts a function pointer accepting void arguments and returning void.

```
int main(int argc, char **argv)
{
    UNITY_BEGIN();
    if (TEST_PROTECT()) {
        RUN_TEST(test_cos_strlen);
        RUN_TEST(test_cos_strcmp);
    }
    UNUSED(argc);
    UNUSED(argv);
    return UNITY_END();
}
```

Using the Unity Asserts, any number of cases can be tested against.
For example,

```
void test_cos_strlen(void)
{
    const char str[13] = "Hello World!";

    TEST_ASSERT_EQUAL_INT(0, cos_strlen(""));
    TEST_ASSERT_EQUAL_INT(12, cos_strlen(str));
}
```

References:

[Unity Test Framework Getting Started Guide](https://github.com/ThrowTheSwitch/Unity/blob/master/docs/UnityGettingStartedGuide.md)
[Unity Test Framework Official Repository](https://github.com/ThrowTheSwitch/Unity)