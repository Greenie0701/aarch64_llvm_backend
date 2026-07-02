# C++ Concepts to Know Before Studying the AArch64 LLVM Backend

This document lists all the C++ concepts you should be familiar with before diving into the AArch64 backend source code in LLVM.

Reference: https://learn.microsoft.com/en-us/cpp/cpp/

---

## Core OOP

1. Classes and structs
2. Inheritance (single and multiple)
3. Virtual functions
4. Pure virtual functions
5. `override` keyword
6. `final` keyword
7. Abstract classes
8. Access specifiers (`public`, `private`, `protected`)
9. `friend` classes and functions
10. Constructor chaining
11. Initializer lists
12. Destructors and virtual destructors

---

## Templates

13. Function templates
14. Class templates
15. Non-type template parameters
16. Template specialization
17. Template method pattern
18. Variadic templates
19. `typename` vs `class` in templates
20. `isa<>`, `cast<>`, `dyn_cast<>` (LLVM template utilities)

---

## Modern C++ (C++11/14/17)

21. `auto` keyword
22. Range-based for loops
23. `nullptr`
24. `std::optional`
25. `std::unique_ptr`
26. `std::shared_ptr`
27. Move semantics
28. Rvalue references (`&&`)
29. `std::move`
30. Lambda expressions
31. Structured bindings (`auto [a, b] = ...`)
32. `std::tie`
33. `std::make_tuple` / `std::tuple`
34. `std::make_pair` / `std::pair`
35. `constexpr`
36. `static_assert`
37. Delegating constructors
38. `= default` and `= delete`
39. Initializer list constructors

---

## Type System

40. `const` correctness
41. `mutable` keyword
42. `static` members and methods
43. `inline` functions
44. `explicit` constructors
45. Type casting (`static_cast`, `reinterpret_cast`, `const_cast`)
46. `sizeof` operator
47. `decltype`
48. Type aliases (`using` and `typedef`)

---

## Enumerations

49. `enum`
50. `enum class` (scoped enums)
51. Enum with underlying types
52. Bitfield enums

---

## Memory & Pointers

53. Raw pointers
54. Pointer arithmetic
55. References vs pointers
56. `nullptr` checks
57. Stack vs heap allocation
58. `new` and `delete`
59. Memory alignment (`alignas`, `Align`)
60. `std::numeric_limits`

---

## STL Containers

61. `std::vector`
62. `std::array`
63. `std::map`
64. `std::unordered_map`
65. `std::set`
66. `std::string`
67. `StringRef` (LLVM type)
68. `ArrayRef` (LLVM read-only array view)
69. `SmallVector` (LLVM optimized vector)
70. `SmallPtrSet` (LLVM pointer set)
71. `DenseMap` (LLVM hash map)
72. `BitVector` (LLVM bit array)
73. `StringMap` (LLVM string-keyed map)

---

## STL Algorithms & Utilities

74. `std::transform`
75. `std::any_of` / `std::all_of`
76. `std::find` / `std::find_if`
77. `std::sort`
78. `std::min` / `std::max`
79. `erase_if`
80. Iterator protocol (`begin`, `end`)
81. `iterator_range`
82. Range adaptors

---

## Functions & Callables

83. Function pointers
84. `std::function`
85. Functors (callable objects / `operator()`)
86. Callbacks and higher-order functions

---

## Namespaces

87. Namespace declaration and usage
88. Nested namespaces
89. Anonymous namespaces
90. `using namespace`
91. `using` declarations

---

## Preprocessor

92. `#define` macros
93. `#include` guards (`#ifndef / #define / #endif`)
94. `#pragma once`
95. Conditional compilation (`#ifdef`, `#ifndef`)
96. `#undef`
97. Macro functions
98. X-macros pattern (used in `.inc` files)

---

## Operator Overloading

99. `operator|=`
100. `operator==` / `operator!=`
101. `operator[]`
102. `operator()`
103. `operator<<` (for debug output)

---

## Miscellaneous

104. `assert` macro
105. `static` local variables
106. Forward declarations
107. Anonymous (`unnamed`) namespaces
108. `[[nodiscard]]` attribute
109. Bit manipulation (`&`, `|`, `^`, `~`, `<<`, `>>`)
110. Integer overflow behavior
111. `std::numeric_limits<int>::max()`
112. `LLVM_DEBUG` macro
113. `llvm_unreachable`
114. `#include "*.inc"` generated file pattern
 
