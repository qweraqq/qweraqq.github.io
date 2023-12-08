---
layout: post
title: "Static Code Analysis"
excerpt_separator: <!--more-->
date: 2023-11-11 00:00:00 +0800
author: xiangxiang
categories: security
tags: [sast security codeql]
---
{% highlight text %}
       Data Flow
Source ----------> Sink
{% endhighlight %}

<!--more-->
* auto-gen TOC:
{:toc}

## 0x00 refs
- [CodeQL zero to hero part 1: the fundamentals of static analysis for vulnerability research](https://github.blog/2023-03-31-codeql-zero-to-hero-part-1-the-fundamentals-of-static-analysis-for-vulnerability-research/)
- [Representation and Analysis of Software](https://ics.uci.edu/~lopes/teaching/inf212W12/readings/rep-analysis-soft.pdf)
- [Apache Dubbo: All roads lead to RCE](https://securitylab.github.com/research/apache-dubbo/)
- [CVE-2018-11776: How to find 5 RCEs in Apache Struts with Semmle QL](https://www.twblogs.net/a/5b7e983b2b717767c6aaa918)

## 0x01 static analysis fundamentals
### 1.1 The problem: finding vulnerabilities
main cause of vulnerabilities: 
**untrusted, user-controlled input being used in sensitive or dangerous functions of the program**

### 1.2 data flow, sources, and sinks
To represent these in static analysis, we use terms such as data flow, sources, and sinks
- sources: User input generally comes from entry points to an application—the origin of data. These include parameters in HTTP methods, such as GET and POST, or command line arguments to a program
  
- sinks: Continuing with our SQL injection, an example of a dangerous function that should not be called with unsanitized untrusted data could be `MySQLCursor.execute()` from the MySQLdb library in Python or Python’s `eval()` built-in function which evaluates arbitrary expressions. These dangerous functions are called "sinks". Note that just because a function is potentially dangerous, it does not mean it is immediately an exploitable vulnerability and has to be removed. Many sinks have ways of using them safely

- data flow: For a vulnerability to be present, the unsafe, user-controlled input has to be used without proper sanitization or input validation in a dangerous function. In other words, there has to be a code path between the source and the sink, in which case we say that data flows from a source to a sink—there is a **data flow** from the source to the sink


{% highlight text %}
 +---------+               +-------+
 |         |      data     |       |
 | sources |  -----------> | sinks |
 |         |      flow     |       |
 +---------+               +-------+
untrusted input         dangerous functions
          
{% endhighlight %}

### 1.3 Finding sources and sinks
- manually by using the *nix utility `grep`vulnerabilities
  + The first problem that we found with grepping was that it returned results from comments and function names. Filtering out these results could easily be solved by performing lexical analysis—a well-known compiler technology
  + tools report on dangerous sinks, which might not have been used with untrusted data or in a way causing a vulnerability. There are still too many false positives.

- Early static analysis tools–lexical pattern matching
Another important feature that was introduced in static analysis tools at that time was a knowledge base containing information about dangerous sinks, and matched the sink name from the tokens. 

we can (given that a tool supports this functionality) detect sources and sinks automatically without too many false positives


### 1.4 Syntactic pattern matching, abstract syntax tree, and control flow graph
- The earliest static analysis tools leveraged one technology from compiler theory—lexical analysis. 
- One of the ways to increase precision of detecting vulnerabilities is via leveraging more techniques common in compiler theory. 
- As time progressed, static analysis tools adopted more technologies from compilers—such as parsing and abstract syntax trees (AST). 
- Although there exist a lot of static analysis methods, we are going to explore one of the **most popular—data flow analysis with taint analysis**.


After the code is scanned for tokens, it can be built into a more abstract representation that will make it easier to query the code. One of the common approaches is to parse the code into a parse tree and build an [abstract syntax tree (AST)](https://en.wikipedia.org/wiki/Abstract_syntax_tree)


To make our analysis even more accurate, we can use another representation of source code called control call graph (CFG). A control flow graph describes the flow of control, that is the order in which the AST nodes are evaluated in all possible runs of a program, where each node corresponds to a primitive statement in the program. These primitive statements include assignments and conditions. Edges going out from a node denote a possible successor of that statement in the same run of the program. Thanks to the control flow graph, we can track how the code flows throughout the program and perform further analysis


### 1.5 Data flow analysis and taint tracking
- A call graph (CFG) is a representation of potential control flow between functions or methods
- Problem: As an example, if a string is concatenated with another string, then a new string with a different value is created and data flow analysis would not track it.

- To solve this problem, we use taint tracking

- It works similarly to data flow analysis, but with slightly different rules. Taint tracking marks certain inputs—sources—as “tainted” (here, meaning unsafe, user-controlled), which allows a static analysis tool to check if a tainted, unsafe input propagates all the way to a defined spot in our application, such as the argument to a dangerous function.


- Another popular form is the Single Static-Assignment (SSA), in which the control flow graph is modified so that every single variable is assigned exactly once. The SSA form considerably improves the precision of various data flow analysis methods.

- Oftentimes, static analysis tools use several representations for their analyses, so they can capture advantages of using each of these representations and deliver more precise results.


## 0x02 CodeQL Data flow graph
- [https://codeql.github.com/docs/writing-codeql-queries/about-data-flow-analysis/](https://codeql.github.com/docs/writing-codeql-queries/about-data-flow-analysis/)

### 2.1 v.s. AST
Unlike the abstract syntax tree, the data flow graph does not reflect the **syntactic structure** of the program, but **models the way data flows through the program at runtime**. 
- Nodes in the abstract syntax tree represent syntactic elements such as statements or expressions. 
- Nodes in the data flow graph, on the other hand, represent semantic elements that carry values at runtime.
- Some AST nodes (such as expressions) have corresponding data flow nodes, but others (such as if statements) do not. This is because expressions are evaluated to a value at runtime, whereas if statements are purely a control-flow construct and do not carry values. There are also data flow nodes that do not correspond to AST nodes at all.

### 2.2 edges
- Edges in the data flow graph represent the way data flows between program elements

```
in the expression x || y there are data flow nodes corresponding to the sub-expressions x and y, as well as a data flow node corresponding to the entire expression x || y. There is an edge from the node corresponding to x to the node corresponding to x || y, representing the fact that data may flow from x to x || y (since the expression x || y may evaluate to x). Similarly, there is an edge from the node corresponding to y to the node corresponding to x || y.
```

- Local and global data flow differ in which edges they consider
  + local data flow only considers edges between data flow nodes belonging to the same function and ignores data flow between functions and through object properties. 
  + Global data flow, however, considers the latter as well
  + Taint tracking introduces additional edges into the data flow graph that do not precisely correspond to the flow of values, but model whether some value at runtime may be derived from another, for instance through a string manipulating operation.

### 2.3 challenges
Computing an accurate and complete data flow graph presents several challenges:
- It isn’t possible to compute data flow through standard library functions, where the source code is unavailable.
- Some behavior isn’t determined until run time, which means that the data flow library must take extra steps to find potential call targets.
- Aliasing between variables can result in a single write changing the value that multiple pointers point to.
The data flow graph can be very large and slow to compute.
- The data flow graph can be very large and slow to compute.

To overcome these potential problems, two kinds of data flow are modeled in the libraries:
- Local data flow, concerning the data flow within a single function. When reasoning about local data flow, you only consider edges between data flow nodes belonging to the same function. It is generally sufficiently fast, efficient and precise for many queries, and it is usually possible to compute the local data flow for all functions in a CodeQL database.
- Global data flow, effectively considers the data flow within an entire program, by calculating data flow between functions and through object properties. Computing global data flow is typically more time and energy intensive than local data flow, therefore queries should be refined to look for more specific sources and sinks.

### 2.4 CodeQl Normal data flow vs taint tracking

- In the standard libraries, we make a distinction between ‘normal’ data flow and taint tracking. 

- The normal data flow libraries are used to analyze the information flow in which data values are preserved at each step.

- In QL, taint tracking extends data flow analysis by including steps in which the data values are not necessarily preserved, but the potentially insecure object is still propagated. These flow steps are modeled in the taint-tracking library using predicates that hold if taint is propagated between nodes.


```
For example, if you are tracking an insecure object x (which might be some untrusted or potentially malicious data), a step in the program may ‘change’ its value. So, in a simple process such as y = x + 1, a normal data flow analysis will highlight the use of x, but not y. However, since y is derived from x, it is influenced by the untrusted or ‘tainted’ information, and therefore it is also tainted. Analyzing the flow of the taint from x to y is known as taint tracking.
```

## 0x03 MISC
### 3.1 如何对比工具
- 语言的支持
- 开发框架的支持及自带规则的数量 (自带的规则)
- data-flow的支持 比如semgrep仅支持local data flow https://semgrep.dev/docs/writing-rules/data-flow/taint-mode/
- 性能
- 使用门槛
- CICD集成

### 3.2 哪些漏洞可以通过静态代码扫描发现
- injection
- 反序列化
- xss

### 3.3 哪些漏洞无法通过静态代码扫描发现
- 业务逻辑(流程相关)漏洞

