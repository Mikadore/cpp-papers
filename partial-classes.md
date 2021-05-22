---
title: "Freestanding Library: Partial Classes"
document: <TBD>
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
It marks (parts of) `std::array`, `std::string_view`, `std::variant`, and `std::optional` (hereinafter referred to as "the added classes") as such.
A future paper might add `std::bitset` (as was the original goal in [@P2268R0])

# Motivation and Scope
All of the added classes are fundamentally compatible with Freestanding,
except for a few methods that throw (e.g. `array::at`). We explicitly 
`=delete` these undesirable methods. 

The main driving factor for these additions the immense use of these types in practice.

## Scope
We refine [freestanding.membership] by specifying the notion of 
partial classes, and accordinly specify the added classes as (partially) freestanding.

## Implementation experience
### The Existing Standard Library
We made a fork of libc++. `=delete`ing all the methods was trivial, 
except for some methods on `string_view` (which are implemented in terms of the
`=delete`d `string_view::substring`).
All test cases (except for the `=delete`d methods) passed
after some rather minor adjustments (e.g. replacing `get<0>(v)` with `*get_if<0>(&v)`).

### In Practice
Since we aren't changing the semantics of any of the classes
(except `=delete`ing non-critical methods), it is fair to say
that all of the (implementer *AND* user) experience gathered as part of Hosted
applies the same to Freestanding.

Therefore, we shouldn't be asking whether the *design* works,
rather, whether it's technically feasible. To which the answer is
yes! For example, the [@ETL] offers direct mappings
of the `std` types. Even in kernel level libraries, like Serenity's
[@AK] use these utilities.

# Design decisions

## [conventions] changes
The predecessor to this paper used `//freestanding, partial` to signal a class
require be only partially implemented, in conjunction with `//freestanding, omit` signaling 
a declaration is not required to be present in freestanding.

This paper chooses to go the more explicit way, that is marking each required declaration 
inside a class as `//freestanding`. It simultaneously introduces the concept of having two
different declarations with the same name on freestanding vs hosted.

# Wording

## Change in [conventions]
Add new paragraphs to [freestanding.membership]

::: add
> [5]{.pnum} A declaration may be specified differently for freestanding implementations, in this case
> a hosted implementation shall use the variant not marked as freestanding, whereas a freestanding implementation
> shall use the variant marked as freestanding.
> 
> [ *Example:*  
> ```
>     constexpr reference at(size_type n) = delete; // freestanding
>     constexpr reference at(size_type n);
> ```
> On a freestanding implementation `at` is deleted, whereas on a hosted one `at` is just a normal function.
> -end example]
>
> [6]{.pnum} A partially freestanding class or partially freestanding class template is a type which has at least one member that is either 
> declared freestanding, or that differs from hosted as described in (5).
> [ *Note:*
>   These classes might be referred to as "partial classes" outside the standard.
> -end note]
>
> [7]{.pnum} Partially freestanding classe's or partially freestanding class template's members follow the same convetions
> as file scope declarations in regards to being marked as freestanding.
>
> [ *Example:*
>
>   ```
>   class A {               // freestanding 
>     int a();              // freestanding
>     int b();
>     int b() = delete;     // freestanding
>     using Result = int;   // freestanding
>   };
>   ```
>
> Here, class `A` is a partially freestanding class,
> whose member type `Result` and member function `a` are both required on freestanding,
> but whose member function `b` is deleted on freestanding.
> -end example]
:::

## Changes in [compliance]
Add new rows to Table 24:

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