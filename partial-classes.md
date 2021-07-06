---
title: "Freestanding Library: Partial Classes"
document: 000
date: 2021-05-11
audience:
  - Library Evolution Working Group
  - Library Working Group
author:
  - name: Emil Meissner 
    email: <e.meissner@seznam.cz>
  - name: Ben Craig
    email: <ben dot craig at gmail dot com>
---

# Introduction
This proposal is part of a group of papers aimed at improving the state of Freestanding.
It marks (parts of) `std::array`, `std::string_view`, `std::variant`, and `std::optional` as such.
A future paper might add `std::bitset` (as was the original goal in [@P2268R0])

# Motivation and Scope
All of the added classes are fundamentally compatible with Freestanding,
except for a few methods that throw (e.g. `array::at`). We explicitly
`=delete` these undesirable methods.

The main driving factor for these additions the immense usefulness of these types in practice.

## Scope
We refine [freestanding.membership] by specifying the notion of
partial classes, and accordinly specify the newly (partially) freestanding classes as such.

## Implementation experience
### The Existing Standard Library
We've forked libc++, and deleted all not freestanding methods.
Except for some methods on `string_view` (which are implemented in terms of the
deleted `string_view::substring`), this did not require any changes in the implementation.
All test cases (except for the deleted methods) passed
after some rather minor adjustments (e.g. replacing `get<0>(v)` with `*get_if<0>(&v)`), confirming
that all these types are usable without the deleted methods.

### In Practice
Since we aren't changing the semantics of any of the classes
(except deleted non-critical methods), it is fair to say
that all of the (implementer *and* user) experience gathered as part of Hosted
applies the same to Freestanding.

The only question is, whether these classes are compatible with Freestanding. To which the answer is
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

# Justification for deletions
Every deleted method is throwing.
We omit `string_view`'s associated `operator<<` since we don't add `basic_ostream`.


# Wording

## Change in [conventions]

Add new paragraphs to [freestanding.membership]

::: add

> [5]{.pnum} A *freestanding member* is a member declaration of a freestanding class template that is implemented in freestanding implementations.
>
> [6]{.pnum} A *partially freestanding class template* is a freestanding class template, where at least one, but not all members are freestanding 
> members.
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
> [7]{.pnum} Each freestanding member in the header synopsis of a partially freestanding class template
> is followed by a comment including *freestanding*.
>
> [6]{.pnum} A *fully freestanding class template* is a freestanding class template, in which every member is a freestanding member.
> In the associated header synopsis for such a class template, the class template's declaration is followed with a comment
> that includes *freestanding*.
>
> [ *Example* 
>
>   ```cpp
>   template<class... Types> class variant; //freestanding
>   ```
>
> -*end example*]
>
> [8]{.pnum} Members of a partially freestanding class template that are not freestanding members may be deleted in freestanding implementations.
> In that case, a freestanding implementation shall explicitly delete that member. In the associated header synopsis of the given partially
> freestanding class template, the deleted member's declaration is followed with a comment that includes *freestanding* and *delete*.
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

> |         | Sublcause           | Header(s)  |
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

Please append a `//freestanding, delete` to the following entities:

- `bad_optional_access`

## Changes in [optional.optional.general]
Instructions to the editor:

Please append a `//freestanding` to every entity except:

- every reference qualified overload of `value`

Please append a `//freestanding, delete` to the following entities:

- every reference qualified overload of `value`

## Changes in [variant.syn]
Instructions to the editor:

Please append a `//freestanding` to every entity except:

- `bad_variant_access`
- every overload of `get`

Please append a `//freestanding, delete` to the following entities:

- `bad_variant_access`
- every overload of `get`

## Changes in [string.view.synop]
Instructions to the editor:

Please append a `//freestanding` to every entity except:

- `operator<<`

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