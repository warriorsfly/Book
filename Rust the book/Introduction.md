

## Introduction

> Note: This edition of the book is the same as [The Rust Programming Language]() available in print and ebook format from [No Starch Press]().

Welcome to *The Rust Programminng Language*, an introductory book about Rust. The Rust programming language helps you write faster, more reliable software. High-level ergonomics and low-level control are often at odds in programming language design; Rust challenges that conflict. Through bablancing powerful technical capacity and a great developer exprience,Rust gives you the option to control low-level details(such as memory usage) without all the hassle traditionally associated with such control.



## Who Rust Is For

Rust is ideal for many people for a variety of reasons Let‘s look at a few of The most important groups.



## Teams Of  Developers

Rust is proving to be a productive tool for collaborationng among large teams of developers with varying levels of systems programming knowledge. Low-level code is prone to a variety of subtle bugs, which in most other languages can be caught only through extensive testing and careful code review by experienced decelopers. In Rust, the compiler plays a gatekeeper role by refusing to compile code with these elusive bugs, including concurrency bugs. By working alongside the compiler, the team can spend their time focusing on the program's logic rather than chasing down bugs.

Rust also brings contemporary developer tools to the systems programming world:

- Cargo, the included dependency manager and build tool, makes adding, compiling, and managing dependencies painless and consistent across developers.
- Rustfmt ensures a consistent coding style across developers.
- The Rust Language Server powers Integrated Development Environment(IDE) integration for code completion and inline error messages.

By using these and other tools in the Rust ecosystem, developers can be productive while writing systems-level code.

## Students

Rust is for students and those who are interested in learning about systems concepts. Using Rust,  many people have learned about topics like operating systems development. The community is very welcoming and happy to answer student questions. Through efforts such as this book, the Rust teams want to make systems concepts more accessible to more people , especially those new to programming.

## Companies

Hundreds of companies, large and small, use Rust in production for a variety of taskS. Those tasks include command line tools, web services, DevOps tooling, embedded devices, audio and video analysis and transcoding, cryptocurrencies， bioinformatics, serch engines, Internet of Things applications, machine learning, and even magor parts of the Firefox web browser.

## Open Source Developers

Rust is for people who wants to build this Rust programming language, community, developer tools, and libraries. We'd love to have you contribute to the Rust language.

## People Who Value Speed and Stability

Rust is for people who crave speed and stability in language. By speed, we mean the speed of the programs that you can create with Rust and the speed at which Rust lets you write them. The Rust compiler's checks ensure stability through feature additions and refactoring. This is in contrast to the brittle legacy code in language without these checks, which developers are often afraid to modify. By striving for zero-cost abstractions, higher-level features that compile to lower-level code as fast as code written manually, Rust endeavors to make safe code be fast code as well.

The Rust language hopes to support many other users as well; those mentioned here are merely some of the biggest stakeholders. Overall, Rust's greatest ambition is to eliminate the trade-offs that ergonomics. Give Rust a try and see if its choices work for you.

## Who This Book Is For

This book assumes that you've written code in another programming language but doesn't make any. Assumptions about which one. We've tried to make the material broadly accessible to those from a wide variety of programming backgrounds. We don't spend a lot of time talking about what programming is or how to think about it. If you're entirely new to programming, you would be better served by reading a book that specifically provides an introduction to programming.

## How to Use This Book

In general, this book assumes that you're reading it in sequence from front to back. Later chapters build on concepts in earlier chapters, and earlier chapters might not delve into details on a topic; we typically revisit the topic in a later chapter.

You'll find two kinds of chapters in this book; concept chapters and project chapters. In concept chapters, you'll learn about an aspect of Rust. In project chapters, we'll build small programs together, applying what you've learned so far. Chapter 2, 12, and 20 are project chapters; the rest are concept chapters.

Chapter 1 explains how to install Rust, how to write a Hello, world! Program, and how to use Cargo, Rust's package manager and buuild tool. Chapter 2 is a hands-on introduction to the Rust language. Here we cover concepts at a high level, and later chapters will provide additional detail. If you want to skip Chapter 3,which covers Rust features similar to those of other programming languages, and head straight to Chapter 4 to learn about Rust's ownership system. However, if you're a particularly meticulous learner who prefers to learn every detail before moving on the next, you might want to skip Chapter 2 and go straight to Chapter 3, returning to Chapter 2 when you'd like to work on a project applying the details you've learned.

