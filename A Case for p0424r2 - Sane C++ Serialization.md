# A Case for p0424 - Sane C++ Serialization


### Teaser

Probably every C# developer knows how versatile the Newtonsoft.Json or the standard C# serialization libraries are. The serialization with `DataContract` or `JsonProperty` is easy to write, maintain and quite efficient. When we compare it with what C++ have to offer, the latter looks very poorly. We have selection of dedicated reflection/serialization libraries e.g [Boost Serialization](https://www.boost.org/doc/libs/1_67_0/libs/serialization/doc/index.html), [RTTR](https://github.com/rttrorg/rttr), [FlatBuffers](https://google.github.io/flatbuffers) (probably the best choice here) or even more recent attempt to write one [FlexibleReflection](https://github.com/preshing/FlexibleReflection/tree/part1). Unfortunately all of them come with different sets of drawbacks: they require the serialization methods to be written manually, force usage of ugly macros, require manual registration of the structures and their fields or involve additional compilation step for the schema definitions.

This may change with the new C++20 standard and the introduction of the string literals as the template parameters (see [p0424r2](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0424r2.pdf)). Here is a teaser of the code that we may be able to compile. First, let's define a few data structures:

    struct date_t {
      json_property<"day", int> day;
      json_property<"month", int> month;
      json_property<"year", int> year;
    };

    struct employee_t {
      json_property<"name", std::string> name;
      json_property<"employed_date", date_t> employed_date;
      json_property<"salary", int> salary;
      json_property<"level", std::string> level;
    };

    struct team_t {
      json_property<"name", std::string> name;
      json_property<"employees", std::vector<employee_t>> employees;
    };

Serialization is a walk in the park (just call `to_json` to get a json string):

    unittest("Serialize team description.") {
      team_t highly_trained_monkeys {
        "Highly Trained Monkeys",
        vector<employee_t> {
          { "Maxymilian Debeściak", { 10, 12, 1999 }, 13000, "Software Engineer" },
          { "Geralt of Rivia", { 13, 3, 1742 }, 12345, "QA Engineer / Witcher" },
          { "Jakub Wyndrowycz", { 1, 1, 1900 }, 100, "Amateur Exorcist" }
        }
      };

      const char* expected = R"foo(
      {
        "name": "Highly Trained Monkeys",
        "employees":
        [
          {
            "name": "Maxymilian Debeściak",
            "employed_date": { "day": 10, "month": 12, "year": 1999},
            "salary": 13000, "level": "Software Engineer"
          },
          {
            "name": "Geralt of Rivia",
            "employed_date": { "day": 13, "month": 3, "year": 1742},
            "salary": 12345,
            "level": "QA Engineer / Witcher"
          },
          {
            "name": "Jakub Wyndrowycz",
            "employed_date": { "day": 1, "month": 1,"year": 1900},
            "salary": 100,
            "level": "Amateur Exorcist"
          }
        ]
      })foo";

      assert_almost_eq(expected, to_json(highly_trained_monkeys)); // ignores white-spaces
    }

De-serialization isn't much harder either:

    unittest("De-serialize team description.") {
      auto team = from_json<team_t>(R"foo(
      {
        "name": "Highly Trained Monkeys",
        "employees":
        [
          {
            "name": "Maxymilian Debeściak",
            "employed_date": { "day": 10, "month": 12, "year": 1999},
            "salary": 13000, "level": "Software Engineer"
          }
        ]
      })foo");

      assert_eq(team.name, "Highly Trained Monkeys");
      assert_eq(team.employees.size(), 1u);
      assert_eq(team.employees[0].name, "Maxymilian Debeściak");
      assert_eq(team.employees[0].employed_date.day, 10);
      assert_eq(team.employees[0].salary, 13000);
      assert_eq(team.employees[0].level, "Software Engineer");
    }

There is no runtime space overhead:

    unittest("No runtime space overhead.") {
      static_assert(sizeof(date_t) == sizeof(int) * 3);
      static_assert(sizeof(employee_t) == sizeof(std::string) * 2 + sizeof(date_t) + sizeof(int));
    }

### Core Idea

So, how does it work you may ask. Well, the inspiration was an excellent talk by Antony Polukhin from CppCon 2016: [C++14 Reflections Without Macros, Markup nor External Tooling](https://www.youtube.com/watch?v=abdeAew3gmQ). For the simplicity in following discussion we will consider only serialization. Our `json_property` decorator is defined as:

    template <auto& X, class T> struct json_property : public T {
      constexpr static const char* name = X;
    };

    template <auto& X> struct json_property<X, int> {
      constexpr static const char* name = X;
      int value;

      operator int() { return value; }

      json_property& operator=(const int& that) {
        value = that;
        return *this;
      }
    };


 The core idea is to find a way to reflect a C++ structure decorated with `json_property` template:

    struct vec3 {
      json_property<"x", int> x;
      json_property<"y", int> y;
      json_property<"z", int> z;
    };

    struct circle_t {
      json_property<"position", vec3> position;
      json_property<"radius", int> radius;
    };


into an array of values of type:

    struct json_property_cref {
      template <auto& X, class T> json_property_cref(const json_property<X, T>& value)
        : name(value.name) {
        this->_data = &value;
        this->_cbck = [](const void* p) -> string {
          return json_convert<T>::to_json(*((const T*)p));
        };
      }

      string to_json() const {
        return _cbck(_data);
      }

      char const* const name;

    private:
      const void* _data;
      string (*_cbck)(const void*);
    };

The `json_property_cref` is a helper class that erases the type of the `json_property`. The type erasure allows to us to handle all the fields in the same way in later stages in the serialization pipeline. The type erasure class exposes only the pointer to the string with the name of the field and the `to_json` method that converts the field's value to the json string.

Now the funny part: we need to convert our input structure to an array of `json_property_cref`s. If we knew the number of the fields in the structure we could use the structured binding declarations in the following way (for `vec3`):

    template <class... U> static std::array<json_property_cref, N> A(U&... x) {
      return std::array { json_property_cref(x)... };
    }

    template <class U> static std::array<json_property_cref, N> unpack(U&& _v) {
      auto&& [a,b,c] = _v; return A(a,b,c);
    }

The problem is it will not work for our `circle_t` structure (it has two fields only). To solve that we can use Antony's idea ([talk](https://www.youtube.com/watch?v=abdeAew3gmQ)) for counting the fields of the structure. When we have the number of the fields `N` we can write a cascade of `if constexpr` to select the proper length:

    constexpr static int N = fields_count<std::decay_t<T>>();

    template <class U> static std::array<json_property_cref, N> unpack(U&& _v) {
           if constexpr (N == 0) { return A(); }
      else if constexpr (N == 1) { auto&& [a] = _v; return A(a); }
      else if constexpr (N == 2) { auto&& [a,b] = _v; return A(a,b); }
      else if constexpr (N == 3) { auto&& [a,b,c] = _v; return A(a,b,c); }
    }

With the above the serialization of the structure can be encapsulated in the following struct:

    template <class T> struct json_convert {
      constexpr static int N = fields_count<std::decay_t<T>>();

      static string to_json(const T& x) {
        auto unpacked = unpack(x);
        string json = "{";

        for (size_t i = 0; i < unpacked.size(); ++i) {
          json += "\"" + string(unpacked[i].name) + "\":" + unpacked[i].to_json();

          if (i + 1 != unpacked.size()) {
            json += ",";
          }
        }

        return json + "}";
      }

      template <class... U> static std::array<json_property_cref, N> A(U&... x) {
        return std::array { json_property_cref(x)... };
      }

      template <class U> static std::array<json_property_cref, N> unpack(U&& _v) {
             if constexpr (N == 0) { return A(); }
        else if constexpr (N == 1) { auto&& [a] = _v; return A(a); }
        else if constexpr (N == 2) { auto&& [a,b] = _v; return A(a,b); }
        else if constexpr (N == 3) { auto&& [a,b,c] = _v; return A(a,b,c); }
      }
    };

For serialization of primitive types and types that are not decorated with `json_property` (e.g. STL collections) explicit specializations of `json_convert` are required (an example for `int`):

    template <> struct json_convert<int> {
      static string to_json(int x) {
        return std::to_string(x);
      }
    };


The last piece is a handy function to resolve the type of the value we are serializing:

    template <class T> string to_json(const T& x) {
      return json_convert<T>::to_json(x);
    }


The above is enough to compile this example:

    struct vec3 {
      json_property<"x", int> x;
      json_property<"y", int> y;
      json_property<"z", int> z;
    };

    struct circle_t {
      json_property<"position", vec3> position;
      json_property<"radius", int> radius;
    };

    int main() {
      std::cout << to_json(circle_t { vec3 { 1, 2, 3 }, 4 }) << "\n";
      return 0;
    }

The output is, as expected:

    {"position":{"x":1,"y":2,"z":3},"radius":4}

A more elaborate implementation (which includes de-serialization) of the idea can be found in [this](https://github.com/ciechowoj/haste-json) repository. It handles the teaser code from the beginning of the post.

### Before You Get Too Excited

Obviously this experimental serialization method has it limitations - it only handles structures which can be de-constructed using structured bindings. Other types can be handled with custom `json_convert` specializations. Another limitation is that we need p0424 to be included into C++ standard, which I hope will happen in 2 years. Currently the workaround for C++17 is to use an ugly macro, something in this pattern (still, I consider it better than the "registration" macros used in existing libraries):

    #define JSON_PROPERTY_CONCAT(pref, line) pref##line

    #define JSON_PROPERTY_HELPER(line, name, type) \
      static constexpr char JSON_PROPERTY_CONCAT(__string_literal, line)[] = name; \
      json_property<JSON_PROPERTY_CONCAT(__string_literal, line), type>

    #define JSON_PROPERTY(name, type) \
      JSON_PROPERTY_HELPER(__LINE__, name, type)


The performance may be an issue as well (or may not - no performance measurements were done). One of the difficulties to overcome is the de-serialization of the structures. JSON doesn't guarantee the order of the fields, hence, during de-serialization a search for the field with proper name have to be performed. The current sample implementation uses linear search for that, but this obviously has a quadratic performance with respect to the number of the fields in the structure. A better idea would be to involve some kind of binary search of even better - hashing. Conceptually is should be possible to sort the field names or generate the hash map for them in compile time as the names of the fields are constexpr. The idea is promising bur requires some more research.

### Conclusion

The introduction of string literals as non-type template parameters may enable a set of new exciting possibilities and techniques to the future C++ libraries. It may help to fill a big gap between the methods used for reflection and serialization in C++ and other languages, at least till the time we get a proper reflection support from the language.
