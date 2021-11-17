---
title: "Freestanding Library: Partial Classes"
document: P2407R0
date: 2021-07-11
audience:
  - Library Evolution Working Group
author:
  - name: Emil Meissner 
    email: <e.meissner@seznam.cz>
  - name: Ben Craig
    email: <ben dot craig at gmail dot com>
---

# Changes from previous revisions
## Changes from R0
- Add wording for feature test macros
- Mention monadic optional and `string_view::contains` 

# Introduction
This proposal is part of a group of papers aimed at improving the state of freestanding.
It marks (parts of) `std::array`, `std::string_view`, `std::variant`, and `std::optional` as such.
A future paper might add `std::bitset` (as was the original goal in [@P2268R0])

# Motivation and Scope
All of the added classes are fundamentally compatible with freestanding,
except for a few methods that throw (e.g. `array::at`). We explicitly
`=delete` these undesirable methods.

The main driving factor for these additions is the immense usefulness of these types in practice.

## Scope
We refine [freestanding.membership] by specifying the notion of
partial classes, and accordingly specify the newly (partially) freestanding classes as such.

### About \<bitset\>
As mentioned in the introduction, this paper does not deal with bitset.
Bitset is unique in that a relatively big part of its interface depends
on `std::basic_string`. We do not currently have a sound plan to make bitset work as nicely as we'd like to. This situation is made worse
by a significant amount of bitset's member functions that throw.

## Implementation experience
### The Existing Standard Library
We've forked libc++, and `=delete`d all not freestanding methods.
Except for some methods on `string_view` (which are implemented in terms of the
deleted `string_view::substring`), this did not require any changes in the implementation.
All test cases (except for the deleted methods) passed
after some rather minor adjustments (e.g. replacing `get<0>(v)` with `*get_if<0>(&v)`), confirming
that all these types are usable without the deleted methods.

### In Practice
Since we aren't changing the semantics of any of the classes
(except deleted non-critical methods), it is fair to say
that all of the (implementer *and* user) experience gathered as part of hosted
applies the same to freestanding.

The only question is, whether these classes are compatible with freestanding. To which the answer is
yes! For example, the [@ETL] offers direct mappings
of the `std` types. Even in kernel-level libraries, like Serenity's
[@AK] use a form of these utilities.

# Design decisions

## Deleting behavior

Our decision to delete methods we can't mark as freestanding
was made to keep overload resolution the same on freestanding as hosted.

An additional benefit here is, that users of these classes, who might expect
to use a throwing method, which was not provided by the implementation, will get
a more meaningful error than the method simply missing. This also means we can
keep options open for reintroducing the deleted functions into freestanding. (e.g. `operator<<(ostream, string_view)`,
should `<ostream>` be added).

## [conventions] changes

The predecessor to this paper used `//freestanding, partial` to mean a class (template)
is only required to be partially implemented, in conjunction with `//freestanding, omit` meaning
a declaration is not in freestanding.

In this paper, we keep marking not fully freestanding classes templates as `//freestanding, partial`,
requiring all members of such a class template to be individually marked as freestanding, or not.
This is done to keep things explicit.
We also introduce `//freestanding, delete`, to mean a declaration shall be deleted on freestanding.

## On `std::visit`
In this paper, we mark `std::visit` as freestanding, even though it is theoretically throwing.
However, the conditions for `std::visit` to throw are as follows:

> It is possible for a variant to hold no value if an exception is thrown during a type-changing assignment or emplacement.

This means a variant will only throw on visit if a user type throws (library types don't throw on freestanding).
In this case, `std::visit` throwing isn't a problem, since the user's code is already using, and (hopefully) handling exceptions.

This however has the unfortunate side-effect that we need to keep `bad_variant_access` freestanding.

## Notes on variant and value categories
By getting rid of `std::get`, we force users to use `std::get_if`. Since `std::get_if` returns a pointer,
one can only access the value of a variant by dereferencing said pointer, obtaining an lvalue, discarding
the value category of the held object.
This is unlikely to have an impact on application code, but might impact highly generic library code.

# Justification for deletions
Every deleted method is throwing.
We omit `string_view`'s associated `operator<<` since we don't add `basic_ostream`.

# Monadic optional and string_view::contains
Since this paper was first published, `std::string_view` got a new `contains` member function,
and `std::optional` got `transform`, `and_then`, and `or_else`. All these functions are not throwing,
and there are no other problems regarding freestanding. We therefore opt for them being marked as freestanding.


# Wording

This paper's wording is based on the current working draft, [@N4878], and it assumes
that the wording in [@P1642R5] and [@P2338R0]  has been applied.

## Change in [conventions]

Add new paragraphs to [freestanding.membership]

::: add

> [5]{.pnum} A *freestanding member* is a member declaration of a freestanding class template that is implemented in freestanding implementations.
>
> [6]{.pnum} A *partially freestanding class template* is a freestanding class template, where at least one, but not all members are 
> freestanding members.
> In the associated header synopsis for such a class template, the class template's declaration is followed with a comment
> that includes *freestanding* and *partial*.
>
> [ *Example:*
>
>   ```cpp
>   template<class T, size_t N> struct array; //freestanding, partial
>   ```
>
> -*end example*]
>
> [7]{.pnum} Each freestanding member in the synopsis of a partially freestanding class template
> is followed by a comment including *freestanding*.
>
> [8]{.pnum} *Deleted freestanding members* are member functions of a partially freestanding class template that are designated as such.
> Deleted freestanding members are not freestanding members. In the partially freestanding class template's synopsis, deleted freestanding
> members are followed with a comment that includes
> *freestanding* and *delete*. Deleted freestanding members shall either meet the requirements of a hosted implementation, or be deleted.
>
> [ *Example:*
>
>   ```cpp
>       constexpr reference       at(size_type n); // freestanding, delete
>   ```
>
> -*end example*]

:::

## Changes in [compliance]
Add new rows to Table 24:

> |         | Subclause           | Header(s)  |
> | ------------- |:-------------:| -----:|
> | [...]      | [...] | [...] |
> | ?.? [optional] | Optional Objects | \<optional\>
> | ?.? [variant] | Variants | \<variant\>
> | ?.? [string.view] |  String view classes | \<string_view\>
> | ?.? [array] | Sequence containers | \<array\>
> | [...] | [...] | [...]

## Changes in [optional.syn]
Instructions to the editor:

Please append a `//freestanding` to every entity except:

- `bad_optional_access`
- `optional`

Please append a `//freestanding, partial` to the following entities:

- `optional`

## Changes in [optional.optional.general]
Instructions to the editor:

Please append a `//freestanding` to every entity except:

- every reference qualified overload of `value`

Please append a `//freestanding, delete` to the following entities:

- every reference qualified overload of `value`

## Changes in [variant.syn]
Instructions to the editor:

Please append a `//freestanding` to every entity except:

- every overload of `get`

Please append a `//freestanding, delete` to the following entities:

- every overload of `get`

## Changes in [string.view.synop]
Instructions to the editor:

Please append a `//freestanding` to every entity except:

- `basic_string_view`
- `operator<<`

Please append a `//freestanding, partial` to the following entities:

- `basic_string_view`

## Changes in [string.view.template.general]
Instructions to the editor:

Please append a `//freestanding` to every entity except:

- `at`
- `copy`
- `substr`
- The following overloads of `compare`:
  - `compare(size_type pos1, size_type n1, basic_string_view s)`
  - `compare(size_type pos1, size_type n1, basic_string_view s, size_type pos2, size_type n2)`
  - `compare(size_type pos1, size_type n1, const charT* s)`
  - `compare(size_type pos1, size_type n1, const charT* s, size_type n2)`

Please append a `//freestanding, delete` to the following entities:

- `at`
- `copy`
- `substr`
- The following overloads of `compare`:
  - `compare(size_type pos1, size_type n1, basic_string_view s)`
  - `compare(size_type pos1, size_type n1, basic_string_view s, size_type pos2, size_type n2)`
  - `compare(size_type pos1, size_type n1, const charT* s)`
  - `compare(size_type pos1, size_type n1, const charT* s, size_type n2)`

## Changes in [array.syn]
Instructions to the editor:

Please append a `//freestanding` to every entity except:

- `array`

Please append a `//freestanding, partial` to the following entities:

- `array`

## Changes in [array]
Instructions to the editor:

Please append a `//freestanding` to every entity except:

- `at`

Please append a `//freestanding, delete` to the following entities:

- `at`

## Feature test macros

This part of the paper follows the guide lines as specified in [@P2198R2].
Add the following macros to [version.syn]:
```cpp
#define __cpp_lib_freestanding_array        20XXXXL //also in <array>
#define __cpp_lib_freestanding_optional     20XXXXL //also in <optional>
#define __cpp_lib_freestanding_string_view  20XXXXL //also in <string_view>
#define __cpp_lib_freestanding_variant      20XXXXL //also in <variant>
```

---
references:
  - id: AK
    citation-label: AK
    title: "Serenity OS AK Library"
    author:
      - family: Kling
        given: Andreas
    URL: https://github.com/SerenityOS/serenity/tree/master/AK
  - id: ETL
    citation-label: Embedded Template Library
    title: "Embedded Template Library"
    author:
      - family: Wellbelove
        given: John
    URL: https://www.etlcpp.com/
---