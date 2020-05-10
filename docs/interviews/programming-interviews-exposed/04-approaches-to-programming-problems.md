# 04 Approaches to programming problems

The problems used in interviews have specific requirements. They must be short enough to be explained and solved reasonably quickly, yet complex enough that not everyone can solve them.

Therefore, it’s unlikely that you’ll be asked any real-world problems. _**Almost any worthy real-world problem would take too long to even explain, let alone solve. Instead, many of these problems require algorithmic tricks or uncommonly used features of a language.**_

When you begin solving a problem, don’t start writing code immediately. First, make sure you completely understand the problem. When you’re convinced you have the right algorithm, explain it clearly. Writing the code should be one of your final steps.

## Solving the problems
### Basic steps

1. Make sure you understand the problem.
2. When you understand the question, try a simple example (???).
3. Focus on the algorithm and data structures you will use to solve the problem.
4. After you figure out your algorithm and how you can implement it, explain your solution
to the interviewer.
5. While you code, explain what you’re doing.
6. Ask questions when necessary.
7. After you write the code for a problem, immediately verify that the code works by tracing through it with an example.
8. Make sure you check your code for all error and special cases, especially boundary conditions.

### Getting stuck

- Go back to an example.
- Try a different data structure.
- Consider the less commonly used or more advanced aspects of a language.

## Analyzing your solution
### Big-O Analysis
_**It is a form of runtime analysis that measures the efficiency of an algorithm in terms of the time it takes for the algorithm to run as a function of the input size.**_ It’s not a formal benchmark, just a simple way to classify algorithms by relative efficiency when dealing with very large input sizes.

Big-O analysis, however, is concerned with the asymptotic running time: the limit of the running time as n gets very large. It’s only when n become large that the differences between algorithms are noticeable. As $n$ approaches infinity, the difference between n and n + 2 is insignificant, so the constant term can be ignored. Similarly, for an algorithm running in n + n2 time, the difference between n2 and n + n2 is negligible for a very large n.

$\frac{n!}{k!(n-k)!} = \binom{n}{k}$