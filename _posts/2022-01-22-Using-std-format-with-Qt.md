---
layout: post
title: Using std::format with Qt
---

C++20 has rolled, with a lot of cool and exciting features, but if you're from the Qt gang (or really any other over-10M-locs-codebase gang), then you likely have a lot of your own half-baked solutions for the problems that C++20 tries to solve.

Qt had string formatting since forever, but unfortunately it's far from perfect, and there is no easy way to fix it. Breaking the API is not really an option when what you want to change is a vocabulary type that's literally used by ~million devs worldwide \[[^1]\]. Imagine trying to fix `std::string::substr` to return `std::string_view`. Yeah, not in this timeline.

What exactly is wrong with `QString` formatting you ask? See for yourself:

{% gist a53ce88eed2e3019260add8e3f7e2256 %}

Everything's looking good so far, and the returned value will be exactly what you would expect it to be (which is `"Could not deserialize value 'x' as int"`). Let's try this:

{% gist 2755e92faa958764d40013515e987764 %}

Did you notice that I’ve changed `%2` to `%3` in the code? What do you think this code will return? Correct, the same string as the code above! Uuuuuh.

How did we end up with this? Well, C++ didn’t always have all the nice things we have today, and when `QString` API was first created the only other option besides chaining `arg` calls was using variadic functions (yes, the ones where you have to jump through hoops with `va_start` and `va_end`), and those are even worse. So what `arg` does is it iterates through the string, finds the lowest numbered place marker, and replaces it. Yes, this implies O(NK) complexity for a chain of K `arg` calls.


### Basic std::format support

Let’s try using `std::format` instead. Here is a short API reference:

- `std::format` ties all the stuff that we need together, but it formats into an `std::string`, so it’s not an option.
- `std::format_to` takes an output iterator and either a narrow or a wide format string. There is no unicode support yet, so we’re stuck with wide strings for now.
- `std::formatter` can be specialized to support formatting custom types.

As we’re stuck with `wchar_t`, we’ll need a converting output iterator that would write into a `QString`:

{% gist e45f696b125a9f45f755767f853976f8 %}

This in turn can then be used with `std::back_insert_iterator` to format any type supported by the STL into a `QString`:

{% gist 5643060a8d46011ddb742445bafb48a9 %}

We still can’t format `QString`s though, and this will require some additional work. We need to implement a custom `std::formatter`, and the easiest option would be to delegate to one of the STL formatters, but right now our only option is a formatter for `std::wstring_view`, and `QString` is not a wide string. Thus, we’re gonna be needing this monstrosity:

{% gist 3a35cb10980e075a1057e0f400e4b5cb %}

Yeah, we’re doing a copy on systems where `wchar_t` is 4 bytes, but we’re at the limit of what the current `std::format` API can do — we’re stuck with either `char` or `wchar_t`, and there is no unicode support in sight.

Thus tying it all together into a formatter we get this:

{% gist cf02a2d7e5305e6dfb6d396bb66c7683 %}

So far so good. But we want more.


### Formatting via QDebug

It could be nice if we could format any Qt type that provides an output operator into a `QDebug`. And it’s actually pretty straightforward to do:

{% gist 7205097f993936d22115c53b9799c688 %}

And now we can use pretty much any Qt type with `sformat`. Even `QByteArray`, which might be unintended, but is easy to fix if that’s not what you need.

However, the code we have here is pretty inefficient. It would be great if we could reuse `QDebug` object when formatting a long format string with several place markers, but `std::format_to` doesn’t offer an easy way to do this. And it would also be great to get rid of that back-and-forth between different encodings on systems where `wchar_t` is 4 bytes.


### Making it efficient

The loophole to do what we want exists in the API in the form of `std::vformat_to` function. While `std::format_to` type-erases the iterator type that we pass, `std::vformat_to` doesn’t, and passes it as-is into the formatter, making it accessible via `std::basic_format_context::out()`. And this is basically our only way of passing additional context into the formatter, including that `QDebug` instance that we wanted to reuse. We’ll need to roll out a custom output iterator, but at this point it shouldn’t scare you.

However, now that we’ve tied ourselves to a specific output iterator type, we can do more. Let’s say we know that we’re formatting a `QString` into a `QString`. Why do we even need to go through the iterator at all? We can just call `QString::append` directly. Yes, this doesn’t take format specifications into account, but we could add them back later.

Same goes for the code that we have for `debug_streamable` types. We know we’re formatting into a `QString`, why not stream into it directly? These ideas lead to the following implementation:

{% gist 681e5f024d02c5339d34d63185df2401 %}

`StringFormatState` sets up an output `QString` and a `QDebug` instance that’s writing into it. Note that `QDebug` uses `QTextStream` internally, so this is as efficient as you can get without resorting to reimplementing `QDebug` operators yourself. What’s cool is that formatting `debug_streamable` types is then as simple as just streaming them into the `QDebug` object.

For all these perks we do have to pay the price though:

- Our formatters now need custom iterators to work. Formatting `QString`s via `std::format` is no longer possible. But fixing this is pretty straightforward — just use `if constexpr` to fall back into the old implementation if a different iterator type is passed into our formatter.
- We’ve dropped the support for format specifications, but those should be pretty straightforward to add back. Unfortunately, we won’t be able to reuse the implementation in the STL as it doesn’t expose the bits we’ll need.


### Try it out!

You can check out the complete code [on my github](https://github.com/captainurist/qnob/blob/5ef09ca145aee149fa2a68c12c253664a3c50818/src/util/format.h). I didn’t move it into a separate repository as the scope of what we did here is too small to warrant it. The code is under the MIT license, so feel free to copy-paste it into your project.

The implementation that I have also has support for `QByteArray`s, and efficiently converts between utf8 and utf16 when needed. The latter part required hooking into Qt internals, so you might run into linker errors if you’re not linking with Qt statically. In this case you can either mark the needed functions in `qstringconverter_p.h` with `Q_CORE_EXPORT` and rebuild Qt, or just reimplement the conversions yourself using less efficient public APIs.

\ 

\ 

---

[^1]:  I’m not lying [about the 1 million](https://www.qt.io/stock/qt-group-oyj-managers-transactions-1491998400000).




