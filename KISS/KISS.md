# KISS - keep it simple (stupid)


- Write boring code: Easy to read, easy to change, easy to delete, easy to test ( (c) someone in internet)


- Few highlights from creator of [CodeSense](https://codescene.com/) - multilang static analysis tool:
  - **code quality** is about **understandability**. ... most of our development time is spent trying to understand existing code ... that we know how to change it. ... the easier our code is to reason about, the cheaper it is to modify.
  - Quick and dirty code is all fun until we need to modify it. That's where the costs come because we need to re-learn a part of the solution domain that is no longer fresh in our head. Better to make **all** code as clean as possible. That way we avoid unpleasant future surprises.
  - Of all the bugs I've been guilty of, this one's the worst. **No one** reviewed my code or **discussed the proposed solution with me**. Worse, there was no independent test or verification.
  - Invest in contract tests on any API that I consume 🤔
  - Smart solutions rarely are that smart ... smart solutions tends to introduce edge cases
  - **We should plan for failure**. And when things go wrong, we need to be able to respond fast. Most of these bugs were simple to fix, although **tracking them down was time-consuming and painful**. Good system design - on all levels - is critical to the response times that really matter.


- About abstraction from https://wiki.c2.com/?TooMuchAbstraction
  - The primary **cost of abstraction is indirection**. In my experience, you should only **abstract when** the added abstraction **clarifies things more than** the added indirection **confuses** them.
  - Goal is the **code to be as explicit as possible and express the intent**.
    - Term abstraction seems to be very much overloaded term today for some it means any level of explicitness in the code. E.g. extracting a method is thought of as an abstraction, while I wouldn’t say that’s the case 
      - for some it means any level of explicitness in the code. E.g. extracting a method is thought of as an abstraction, while I wouldn’t say that’s the case
      - for some, abstraction means just putting an interface in front of the class without really abstracting anything:
        ```
        ICustomer
        ```


- Complexity slowdown us [article](https://itnext.io/death-by-code-when-developers-lose-the-fight-against-complexity-95a10706a610)
  - ...development gets more complex, more complex to understand the code and more complex to change the code. This is a perfect environment for mistakes, bugs and confusion...
  - **Tomorrow’s problems** in development are caused by **today’s** solutions and **code**.


- from [google's review handbook](https://google.github.io/eng-practices/review/):
  - Design: Is the code **designed well enough** to **be easy to understand and maintain**?
  - Complexity: Is the code so complex that it’s hard to understand? Could it be simplified even further without making it less obvious?

