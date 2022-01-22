---
layout: post
title: Using std::format with Qt
---

C++20 has rolled, with a lot of cool and exciting features, but if you’re from the Qt gang (or really any other over-10M-locs-codebase gang), then you likely have a lot of your own half-baked solutions for the problems that C++20 tries to solve.


<script src="https://gist.github.com/captainurist/a53ce88eed2e3019260add8e3f7e2256.js"></script>

{% gist a53ce88eed2e3019260add8e3f7e2256 %}



```cpp
return BadCastException::tr("Could not deserialize value '%1' as %2").arg("x").arg("int");
```

