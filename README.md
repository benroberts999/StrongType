# StrongType

A light-weight easy-to-use single-file header-only template class for strong typing. Uses c++17 (see below for c++11 version).

It works using scoped enums; the enum value (+scope) are used to ensure uniqueness for a given user-defined strong type.
The class template, defined in the qip namespace, takes an enum class value, and a base type.
The base type is confined to be an arithmetic type (float, double, int, long, etc.) to allow for operator overloading.
The 'using' declaration is optional, but makes things much easier.

```cpp
using MyType = qip::StrongType<EnumClass::value, BaseType>;
```

The newly-defined Strong Type will, for the most part, behave just like an instance of the underlying base type would.
In particular, the usual arithmetic operations (+, -, \*, /, +=, \*= etc.) are all defined, and they work with iostreams.
The main difference is that all implicit conversions are banned.

## Examples

Usage is best shown with examples. Consider this dummy problem, where we define strong types for Energy, Mass, and Velocity.

```cpp
#include "StrongType.hpp"
enum class MechanicsTypes { energy, mass, velocity };
// Use of enum class ensures each StrongType is unique

using Energy   = qip::StrongType<MechanicsTypes::energy, double>;
using Mass     = qip::StrongType<MechanicsTypes::mass, double>;
using Velocity = qip::StrongType<MechanicsTypes::velocity, double>;
```

Say we have a function that takes a mass and a velocity, and calculates the kinetic energy:
```cpp
// "Old" version, using doubles
double kinetic_energy_old(double m, double v) {
  return 0.5 * m * v * v;
}

// Using user-defined strong types
Energy kinetic_energy(Mass m, Velocity v) {
  return Energy{0.5 * m.as_base() * (v * v).as_base()};
}
```

Using the 'weak' type old version:
```cpp
double mass = 2.0;
double velocity = 10.0;
double energy = kinetic_energy_old(velocity, mass); // Lies make baby Newton cry
```
Did you notice that the mass/velocity parameters were the wrong way around? The compiler sure didn't. This code will run just fine, but will produce incorrect results. This is a particularly difficult error to debug (particularly when the functions become more complicated than the above example).

Instead, consider the strong typed version:
```cpp
Mass m{5.0};
Velocity v{2.0};
Energy ek = kinetic_energy(m, v); // Compiles, gives correct result

auto ek1 = kinetic_energy(v, m); // Fails to compile
auto ek2 = kinetic_energy(Mass{5.0}, Velocity{2.0}); // Compiles
auto ek3 = kinetic_energy(Velocity{2.0}, Mass{5.0}); // Fails to compile
auto ek4 = kinetic_energy(5.0, 2.0); // Fails to compile, no implicit conversion
```
Note: when we do the wrong thing here, we get a compile-time error, with a sensible error message, e.g.:
```
error: could not convert '5.0e+0' from 'double' to 'Mass' {aka 'qip::StrongType<MechanicsTypes::mass, double>'}
```

Implicit construction/conversions are banned to prevent accidental conversions that may lead to errors:
```cpp
  double old_mass = 17.6;
  Mass new_mass = old_mass;   // Fails to compile
  Mass new_mass{old_mass};    // OK, explicit construction
  double mass_dbl = new_mass; // Fails to compile
```

The arithmetic operations all work as they would for the base type, as do comparisons and iostreams:
```cpp
Velocity v1{1.0};
Velocity v2{1.0};
v1 += v2; //OK, arithmetic operations work as expected
v2 *= 3;  //OK, scalar multiplication

// But you cannot mix types (except for scalar literals)
Mass m2{5.0};
auto a = v2 * m2; // Fails to compile

// Comparisons and iostreams also work as though would for BaseType
std::cout << std::boolalpha;
std::cout << v1 << " " << v2 << " " << (v1 < v2) << "\n";
```

Because its common, and unlikely to be an error, you can directly compare with literals (rvalues):
```cpp
std::cout << (v1 > 50.0) << "\n"; //OK

// but not with other (named) types (lvalues)
double fifty = 50.0;
std::cout << (v1 > fifty) << "\n"; //won't compile

// Unless, of course, you are explicit:
std::cout << (v1.as_base() > fifty) << "\n"; //OK
```

If you need to, you may access the base type:
```cpp
Energy::BaseType x = 2.0;
static_assert(std::is_same_v<decltype(x), double>);
```

There are a number of options for getting access to the underlying data using the BaseType
```cpp
Energy e2{5.0};
auto z1 = e2.value;     // direct access
auto z2 = e2.as_base(); // returns a reference to value
auto z3 = double(e2);   // double in this case - using explicit conversion
auto z3b = int(e2);     // Fails to compile - no implicit conversions
auto z4 = static_cast<double>(e2); // OK
auto z5 = Energy::BaseType(e2);    // OK
```

--------------------------------------------------------------------------------

## Make it work with c++14

The template uses a few c++17 features which add some niceness. The exact same functionality can be achieved for c++11 very simply with a small change.
The c++17 version is:

```cpp
// c++17 version
template <auto enumV, typename BaseT> struct StrongType {
private:
  static_assert(std::is_arithmetic_v<BaseT>,
                "StrongType only available for arithmetic types");
  static_assert(
      std::is_enum_v<decltype(enumV)>,
      "StrongType must be instantiated with scoped enum (enum class)");
  using StrongT = StrongType<enumV, BaseT>; // type alias

// Rest of struct body ... (remains unchanged)

};
```
c++14 cannot auto-deduce the enum type, so we have to make the following changes:
```cpp
// c++14 version:
template <typename enumT, enumT enumV, typename BaseT> struct StrongType {
private:
  static_assert(std::is_arithmetic<BaseT>::value,
                "StrongType only available for arithmetic types");
  static_assert(
      std::is_enum<enumT>::value,
      "StrongType must be instantiated with scoped enum (enum class)");
  using StrongT = StrongType<enumT, enumV, BaseT>; // type alias

// Rest of struct body ... (remains unchanged)

};
```

Now, we also need to pass the enum type as a template parameter
```cpp
#include "StrongType.hpp"
enum class MechTypes { energy, mass, velocity };
using Energy   = qip::StrongType<MechTypes, MechTypes::energy, double>;
using Mass     = qip::StrongType<MechTypes, MechTypes::mass, double>;
using Velocity = qip::StrongType<MechTypes, MechTypes::velocity, double>;
```

### Make it work with c++11

As above, but also remove all of the 'constexpr' keywords
