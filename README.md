# Propper

[![gcc](https://github.com/tancaj/propper/actions/workflows/gcc.yml/badge.svg?branch=master)](https://github.com/tancaj/propper/actions/workflows/gcc.yml) [![msvc](https://github.com/tancaj/propper/actions/workflows/msvc.yml/badge.svg)](https://github.com/tancaj/propper/actions/workflows/msvc.yml)

Propper is a small C++20 header only library for converting and validating data between JSON and structs/classes.

## Usage

To use propper you have to inlcude only one header file. All the library, except the annotation macros are behind the namespace `pr`.
```c++
#include <propper.hpp>
```


### Annotation

You have to annotate your structs or classes with the provided `DEFINE_PROPS` and `PROP` macros to create the neccessary metadata for the conversions.
```c++
#include <propper.hpp>
#include <string>

struct user
{
    std::string name;
    std::string email;
    std::string password;
    bool subscribed;
    int age;
    
    DEFINE_PROPS(
        PROP(user, name),
        PROP(user, email),
        PROP(user, password),
        PROP(user, subscribed),
        PROP(user, age),
    )
}
```

Propper supports only the following JSON schema data types:
- string (std::string)
- number (int, float, double)
- boolean (bool)
- object (structs or classes that are annotated with the `DEFINE_PROPS` and `PROPS` macros)
- array (std::vector of any type mentioned above)

```c++
#include <propper.hpp>
#include <string>
#include <vector>

struct user
{
    std::string name;
    std::string email;
    std::string password;
    bool subscribed;
    int age;
    
    DEFINE_PROPS(
        PROP(user, name),
        PROP(user, email),
        PROP(user, password),
        PROP(user, subscribed),
        PROP(user, age),
    )
}

struct site
{
    std::string name;
    std::vector<user> users;
    
    DEFINE_PROPS(
        PROP(site, name),
        PROP(site, users),
    )
}
```

### Conversion
To convert a JSON string to struct that is annotated like above, you have to use the `pr::from_json` function:
```c++
//...
std::string json = R"({...})";
auto [user_instance, model_state] = pr::from_json<user>(json);
```
The function returns a `std::pair`, in which the first object is an instance of the struct/class given as a template argument. The function always tries to convert all the annotated members from the JSON data. If the conversion fails for a member, it is simply initialized.
The second object of the `std::pair` is a `std::unordered_map` which contains all the errors that occured during conversion. See the details of error handling [in the error handling chapter.](#error-handling)

To convert back to JSON, you can use the `pr::to_json` function. The JSON will be minified and the order of the properties in the JSON string will be in alphabetical order.

```c++
//...
std::string json = pr::to_json(user_instance);
```

### Predicates
If you want to validate the incoming data from the JSON you can define multiple predicates for each member during annotations, that will be run against the incoming data. All the built in predicates can be found under the `pr::pred` namespace. 
```c++
#include <propper.hpp>
#include <string>

struct user
{
    std::string name;
    std::string email;
    std::string password;
    bool subscribed;
    int age;
    
    DEFINE_PROPS(
        PROP(user, name),
        PROP(user, email, pr::pred::regex<"fancy email validating regex">),
        PROP(user, password, pr::pred::regex<"fancy password validating regex">),
        PROP(user, subscribed),
        PROP(user, age, pr::pred::greater<0>, pr::pred::less<100>),
    )
}
```
If a predicate fails, its error will be present in the model state object, but the conversion will not stop. If multiple predicates are given, all the predicates will be run against the JSON data individually. In case of `std::vector` the predicates will be run against all the elements in the vector. The following predicates are available in propper:
- `greater`
- `less`
- `greater_eq`
- `less_eq`
- `range`
- `range_eq`
- `regex`
#### Custom predicates
If you need a custom predicates, you can easily write yourself. These predicates are just invocable compile time structs. All predicates must statisfy two constraints:
- `std::predicate` [concept](https://en.cppreference.com/w/cpp/concepts/predicate) with the supported types
- must have a function named `get_error()` that returns a compile time string

For example you could write the following predicate for floating point comparsion that suits your precision needs:
```c++
template<double _value> //since c++20
struct floating_greater
{
    constexpr floating_greater {}
    
    bool operator()(double value)
    {
        //your floating point comparsion logic
    }
    
    constexpr std::string get_error()
    {
        return "Your error message";
    }
}
```



### Error handling

When converting from json, the conversion returns the state of the returned object as well. This object is an `std::unordered_map`, where the keys of the map are the names of the annotated members, and the values are a `std::vector` filled with the errors related to the given member.

Two types of error could occur during conversion:
- direct conversion error (missing property in the JSON or type mismatch between the JSON property and the annotated members)
- predicate error
 
### Contribution
If you would like to play around with propper clone the repository recursively. You can generate the neccesary build files for the test and the sanbox projects with premake. The premake executable for different platforms are found here:
- `vendor/premake/linux`
- `vendor/premake/macos`
- `vendor/premake/windows`

You can find more information [here](https://premake.github.io/docs/Using-Premake) how to generate build files with premake.



