+++
title = "Modern C++: create your own::function"
date = "2024-12-10"
description = "implementation of std::functional"
taxonomies.tags = [
  "C++"
]
+++

Back to C++98, if people want to create a callback type, developer may write the following code.

```cpp
class ICallback {
public:
  virtual cb() = 0;
};
```

```cpp
using f = std::function<std::string(const char*)>;
```
