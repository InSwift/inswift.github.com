---
layout: post
title: "Язык программирования Go, или: Почему все C-подобные языки отстой, кроме одного."
published: false
tags:
  - en2ru
  - go
  - language design
---

__Автор__: Jörg Walter &lt;<golang@syntax-k.de>&gt; 

__Оригинал__: <http://www.syntax-k.de/projekte/go-review>


# Введение

Сперва эта статья должна была стать обзором [Языка Go](http://www.golang.org)
разрабатываемого с 2007 силами Robert Griesemer, Rob Pike,
и Ken Thompson в Google. На данный момент к основной команде
присоединились Ian Lance Taylor, Russ Cox и Andrew
Gerrand. Это C-подобный язык с некоторыми возможностями
обычно свойственными динамическим языкам, а также некоторыми оригинальными
подходами (по крайней мере среди языков общего назначения) к параллелизму и
объектно-ориентированному программированию.
Язык в первую очередь предназначен для системного программирования.
Именно поэтому в этом обзоре Go сравнивается с другими C-подобными языками,
а не со скриптовыми. 

Пока я писал этот обзор, я обнаружил, что многие особенности Go необходимо
разобрать более детально, прежде чем их можно будет сравнивать.
Go просто другой. Нельзя его рассматривать с привычных позиций ООП.
Поэтому эта статья не только, и не столько обзор, сколько введение в язык Go.
Я сам новичёк в Go. Написание этого статьи помогло мне понять этот язык и
как он работает. Тем не менее помните, что я всё ещё не дописал своё первое
приложение на Go. Я писал на многих языках, поэтому я буду сравнивать
их особенности с языком Go.

Go молод. Разработчики объявили его стабильным лишь в этом году.
Этим обзором я также надеюсь вызвать обсуждения о направлении развития языка,
сделать его по-настоящему классным языком. Это включает в себя уделение
внимания к недостаткам языка, которые присутствуют в нём в данный момент.

## Слово о C-подобных языках

Я всегда интересовался новыми языками программирования. Обычно я использую
удобные динамические языки, такие как JavaScript, Perl, в последнее время Python.
Большую часть времени я предпочитаю читабельность, удобство сопровождения
и скорость разработки, чем высокую скорость выполнения.
(по результатам бенчмарков). Обычно нет необходимости в преждевременной оптимизации.
Также важна безопасность. Скриптовые языки избавляют от проблем вроде
переполнения буффера, уязвимостей форматной строки и прочих низкоуровневых
проблем (при условии что сам рантайм языка не подвержен им).

Но у перечисленых языков также есть и недостатки. Они плохо маштабируются.
В то время как мне приходиться иметь дело и с восьмибитными микроконтроллерами
на ARM, и смартфонами, и классическими декстопными приложениями. Обычно
для этого я использую C++, из-за чего мне приходится испытывать неудобства.
Нет удобных операций со строками, неуклюжие регекпы, требующие сторонних
библиотек, ручное управление памятью, и конечно же все кошмарные уязвимости
за последние 40 лет. Но чем он хорош, так это скоростью и экономией
к памяти, какую только вообще можно получить.

Похоже языкам использующихся для низкого уровня недостаёт чего-то черезвычайно
базового.
То что мы  сегодня используем для написания низкоуровневых инструментов и
операционных систем создано несколько десятилетий назад и плохо
соответствует решению сегодняшних проблем. Почему это так я покажу далее.
Это касается только языков следующих <strike>нисходящему</strike> C-подходу,
к которому привыкли большинство людей, но вы легко можете добавить свои заметки
о Pascal, Module, Oberon, Smalltalk, Lisp и любом другом языке, который хоть раз
был использован в ядре системы.

### C 

Я люблю C. По-настоящему люблю. Его простота делает его черезвычайно красивым.
Если только вы не перегибаете палку и не делаете что-нибудь глупое, вроде
написание [сложного GUI-тулкита с использованием языка C как основного](http://www.gtk.org).

Что хорошо, так это то, что у вас есть контроль за всем что происходит почти
на уровне ассемблера, что может быть важно для некоторых случаев.
Если только вы не используете оптимизации.
Тогда хитрое поведение C вам возвращается ввиде подарков вроде
(неочищенной памяти содержащей чувствительные данные __FIXME__) или изменение порядка
выражений несмотря на синхронизирующие барьеры. И этого нельзя избежать.
Если у нас есть адресная арифметика, (тогда необходима весьма неочевидная семантика __FIXME__)
для того чтобы оптимизации были на уровне.

Хотя среди недостатков: строки, управление памятью, массивы, ну... почти всё таит в себе
потенциальные уязвимости. Кроме того что это ещё и неудобно в использовании.
Поэтому хотя C и черезвычайно лёгок, насколько это возможно, но он
плохо подходит для проектов содержащих больше 10 тыс. строк кода.

### C++

C++ в чём-то лучше C, но в остальном довольно плох, особенно это касается
ненужной многословности синтаксиса. Он всегда был таким, и вам это известно.
Это превосходная альтернатива для классической разработки приложений, и
некоторые удобные и [лёгкие](http://www.fltk.org) а также [полнофункциональные](http://qt.nokia.com) GUI-тулкиты
используют его возможности.

Но если вы попробуете использовать прелести современных динамических языков
(лямбда-выражения, map/reduce, независимость от типа, ...) то обычно это происходит так:
Эй, круто! На C++ это тоже можно использовать! Нужно просто написать:
<script src="https://gist.github.com/1019580.js?file=gistfile1.cpp"></script>
О да. Именно. 

Поймите меня правильно, я люблю шаблоны, но использование STL в современном
C++ выглядит как классический пример "Если кувалда это всё что у тебя есть"-синдрома.
В GCC пришлось написать дополнительный код, упрощающий диагностику, именно поэтому
вы можете обнаружить что вот это 5-строчное сообщение было всего-навсего
невыполнением константности при использовании `std::string` метода.
Что ещё хуже, так это то что шаблоны могут быть чертовски медленными.
У вас хватало терпения дождаться когда [Boost](http://www.boost.org) наконец скомпилируется? Хорошая идея,
но плохая реализация.

### Objective C

<p>I feel a bit heretic here. I shouldn't diss anything coming from
NeXT. Still, I can't help it—the bolt-on feeling also applies to
Objective C. It's not nearly as pointlessly verbose as C++, but that bracket
syntax is like a parallel world entering C, just so the holy Syntax Of C is
not disturbed.</p> 

<p>You don't get to write impressive template casts as in C++ (this may be
an advantage ^^), and memory management still is kind-of-manual. At least in
C++ you could get reference counted pointers for free (if you happened to
find the one correct template syntax among thousands of incorrect ones).</p> 

<p>Objective C sidesteps this issue by officially offering optional
GC. Wait—it always was optional for C-oids. boehm-gc exists for how
many years? But standard fare are memory pools, which is a nice and worky
kludge for many situations, but a kludge nonetheless.</p> 

<h3>Java</h3> 

<p>You didn't really think I would forget Java in my rant about C-oid
languages, did you? Now, Java is almost the solution. Almost, were it not
for reality.</p> 

<p>We get binary platform independence, a detailed language specification
with various implementations and no significant traps, classic OO, garbage
collection, some truly dynamic features, lately even generics—and a
prohibitive memory consumtion and slow startup times.</p> 

<p>There's no actual need why this should be the case. Advanced JIT
compilers can (in theory) optimize better than any static ahead-of-time
compiler ever could. GCJ compiles down to machine code, if you want it
to. The VM is even suitable for a complete <a href="http://www.jopdesign.com">hardware implementation</a>. But the JDK
sets the foundation for bloat, and many contemporary Java projects sport a
byzantine complexity.</p> 

<p>Now, you can write quite comfortably in modern Java. The web services
offer some really nice abstractions. Up until you look under the hood and
discover a Rube Goldberg machine. Each layer builds upon last year's
favourite abstraction layer. You can't compromise backwards-compatibility,
right?</p> 

<p>Look at your average web application's lib directory. I wouldn't be
suprised to see a hundred JAR files there, all just for a simple search
database or shopping site which even PHP would do in 10k LOC. And should you
be adventurous, try to build it yourself. A world of fun! Setting up a Linux
system from scratch, <em>without</em> any <a href="http://www.linuxfromscratch.org">step-by-step instructions</a>, is
easier. Trust me, I have done both. Be sure you know how to spell
"dependency hell" forwards and backwards before you begin.</p> 

<p>And all that wrapped in a way too verbose syntax and an old-school object
model. There is nothing fundamentally wrong with this, but others do
better. State of the art is something else. Take a look at <a href="http://www.perl6.org">Perl 6</a>, which really tries to put results of
modern language design into a usable (for certain values of "usable")
language. And those first-ever-in-production features are still decades old!
Java is nowhere near this, except maybe for generics.</p> 

<h3>C#</h3> 

<p>I almost forgot this one. I actually did forget it, until feedback
reminded me of it. Frankly, I hardly know C#. As a language, it seems to be
nice, a great evolution of C and C++. What makes me stay away from it is the
non-free nature. Yes, there is Mono, but I wouldn't like to base my stuff on
a language that is there because of Microsoft's benevolent permission which
it could turn into patent-lawsuits any time. We all know the tricks that
company (well, actually any large company) has up it's sleeves.</p> 

<p>I don't see the point in writing non-cross-platform code, so with Mono
being too threatened for my taste, I stay away from it. Also, the CLR must
first earn enough reputation, while the strengths and weaknesses of the Java
VM are well-understood. I may be writing C# code some day, but it won't be
any day soon.</p> 

<p>With the lack of an <em>open</em> ecosystem of language tools and
alternate and/or special-purpose implementations, it's simply not fit to be
a systems programming language. It's a corporate-only gig.</p> 

<p>Oh, and the fact that a company wants to earn money with the development
of the language itself is not the road to a healthy evolution. Corporate
interests will some day enforce choices that are bad for the quality of the
language. There is a constant pressure for improvement, otherwise you can't
sell anything. As a contrast, look at TeX, which is more or less the same
for 30+ years, and as bug free as software can ever get. You can't really
compare both, but it shows where the spectrum ends, and C# is at the wrong
end.</p> 

<h3>JavaScript</h3> 

<p>JavaScript doesn't sound like it belongs here, since it is a fully
dynamic scripting language. It is one of the most widely deployed
programming languages, however, and it is C-oid as well. And, more
importantly, JS engines these days sport quite advanced optimized JIT
compilers, so performance can be in the same ballpark as with the other
languages mentioned.</p> 

<p>So, what's fundamentally wrong with JS? Not a lot. Only it is not
suitable for small systems. It is a great application-embedded language, it
is <a href="http://www.nodejs.org">suitable for writing network services</a> 
as well, but its design explicitly provides no way of interacting with the
outside world. The hosting application defines all interaction APIs in an
implementation-defined manner, so by definition it can't be the system's
hosting language.</p> 

<p>Moreover, the JIT and runtime requirements make its embeddability
limited. If you own a <a href="http://www.palm.com/us/products/phones/pre/index.html">Palm Pre</a>,
you already use JS as embedded language, and it is very convenient. Only
that 500MHz/256MB system is at the low end of what's useful. Perhaps the
lowest spec device using JS (or rather, ECMAScript) as it's main system
language is the <a href="http://www.chumby.org">Chumby</a>, playing Adobe
Flash Lite movies on 450MHz/64MB. Not exactly a universal language
there.</p> 

<h2>Wishlist</h2> 

<p>Dear Santa, all the low-level languages suck. For Christmas I want a
programming language that has the following features (with examples from
established languages), in loose order of importance:</p> 

<h3>General Design Principles</h3> 

<dl> 
<dt>1. Expressiveness</dt> 

<dd>I want a language that is high-level enough that I can actually
express my ideas and algorithms in, not one that wastes my time and
screen space with tedious management tasks or patterns which consist of
80% boilerplate code. The more important elements of expressiveness are
listed separately. A good test is writing a parser. If the resulting
code has a lot of similarity to the constructs you are parsing, that's
the right direction. (Example: Perl 6 grammars)</dd> 

<dt>2. Simplicity</dt> 

<dd>I want a language that is elegantly simple. Only a few concepts should
express all possibilities. Orthogonality is a key aspect of
this. Simplicity also makes the language easier to learn. Reuse
syntactical constructs for similar purposes and let user code interface
these constructs so they can create added value to these. Also, don't
force complicated structures upon the users. There's nothing wrong with
offering classes, a unit testing framework or doc-comments, but keep them
out of sight if the user doesn't want them. Ideally, make them DWIM.</dd> 

<dt>3. Equal Rights</dt> 

<dd>If the built-in associative array algorithm is inefficient for my
specific data set, I want to be able to create an alternate
implementation that is used exactly like the built-in one. The language
should not be privileged. Consequently, operators should be
overloadable. Oh what magic have I done in Python... Also, built-in data
types should be usable in the languages's object syntax. All of this
need not be dynamic, most of this can be statically evaluated syntactic
sugar. (Example: Perl's prototyped subs, Python's operator overloading)</dd> 

<dt>4. Meta-Programming</dt> 

<dd>This has two aspects. One is templates and/or generics, which are too
useful to not have. Even C had some evil hack mimicking this: the
preprocessor. Everyone needs this. The more advanced aspect is
compile-time executed code that generates code, like Lisp's macros or
Perl's source filters. Bonus points if this allows to create new language
constructs or a domain-specific language.</dd> 

<dt>5. Efficiency and Predictability</dt> 

<dd>If a language aims at system programming, it must be fairly efficient
and complexity must be predictable. I must be able to intuitively
estimate in what order of magnitude time and memory demands of a certain
operation lie. Core language features must be well-optimized. The
library may offer high-level constructs with more coding time efficiency
but less runtime efficiency. The object system must not require
expensive runtime support (neither in time or memory), static
optimizations should generally be possible.<p></p> 

Ideally, it should be possible to write a useful control application in a
few kiB of code and some hundred bytes of memory. Of course, efficiency
and features usually are the opposite ends of a tradeoff, so the language
may give me the chance to decide.</dd> 

<dt>6. Usability</dt> 

<dd>At the end of the day, the language must be usable. All those ideals
aside, foremost it must solve real problems. A little pragmatism doesn't
hurt. The language should be sufficiently close to other languages so you
can think in terms you are used to. If you deviate drastically, it must be
worth it. Honour the principle of least surprise!</dd> 

<dt>7. Module Library and Repository</dt> 

<dd>I want all the niceties I have grown used to in scripting languages
built-in or part of the standard library. A public package repository with
a decent portable package manager is even better. Typical packages include
internet protocols, parsing of common syntaxes, GUI, crypto, common
mathematical algorithms, data processing and so on. (Example: Perl 5
CPAN)</dd> 

</dl> 

<h3>Specific Features</h3> 

<dl> 
<dt>8. Data Structures</dt> 

<dd>I want my hashes! A language without associative arrays as
well-integrated data type is a joke. OO (or some other, convenient method
of encapsulation) is also a must, obviously. A boolean type is nice, but
more important is a sensible interpretation of any data type in a boolean
context. Fat bonus points for advanced data structures in the standard
library, like various trees, lists, sets, etc. If you've got a <a href="ftp://ftp.cs.umd.edu/pub/skipLists/skiplists.pdf">skip list</a>, you
are on the right track.<p></p> 

And make strings OOish, i.e. the same operations (say, <tt>len()</tt>) should work
on a string variable as on an array or an object that mimicks an
array. Bonus if all primitive data types sport the same OO syntax as all
other objects. No need to go Ruby though. Just make it syntactic
sugar. (Example: Python and its standard library)</dd> 

<dt>9. Control Structures</dt> 

<dd>Sounds obvious, but decent looping constructs allow breaking out of
multiple nested levels at once. Eliminate <tt>goto</tt>, really. The last
time I needed that was, like... okay, I admit, it was last week. I was
doing Retro-Show-Programming on a Commodore C64 at the <a href="http://www.computermuseum-oldenburg.de">Oldenburger
Computer-Museum</a>. That doesn't count. So scrap goto, but give me
<tt>foreach</tt>. Not much needed beyond that. Exceptions perhaps, but
please don't hammer each nail home with them. JavaScript's <tt>with</tt> 
statement I have never used, and while it is kind of a nice idea, I guess
it's a case of "Less is More". (Example: all the scripting languages do
fine here)<p></p> 

Actually, there is something I haven't seen anywhere yet. Lately, I have
encountered lots of loops where loop entry and condition testing/loop exit
did not occur adjacent to each other. So if there was a way to express a
loop which starts at a freely chosen point in the middle, that would be oh
so cool. Otherwise you will have to duplicate part of the loop. A bit like
Duff's device, just not for optimization but for making the code less
redundant.</dd> 


<dt>10. Expression Syntax</dt> 

<dd>Many people shun it, but the <tt>?:</tt> ternary operator is a good
idea, if used correctly. Python does <tt>foo if bar else baz</tt>, which
is a little more verbose but still okay. Most dynamic languages, however,
rock with their boolean operators AND and OR not just evaluating to
<tt>true</tt> and <tt>false</tt>, but to the actual value that was
considered <tt>true</tt>. Imagine the assignment <tt>value =
cmdline_option || "default"</tt>. That requires a decent boolean
interpretation of all data types, however.</dd> 

<dt>11. Functional Qualities of Expressions</dt> 

<dd>If I wanted to write Lisp, I would do so. I don't need a fully
functional programming language. But nothing beats a good
<tt>map()</tt>. Closures and anonymous functions (lambda expressions) are
a killer feature. Probably in the "too complex" area would be hyper
operators (as Perl 6 calls them) like <tt>any()</tt> and <tt>all()</tt> 
(as Python calls them), but they rock and give the chance for implicit
parallelization. Welcome to the new millennium, or at least to the
90s.</dd> 

<dt>12. Objects</dt> 

<dd>There are various models of object orientation out there, but the
minimum requirements are encapsulation and polymorphism. Some way of
composing classes should also exist, like inheritance and
mixins. Interfaces should exist, or multiple inheritance as a replacement.
Overloading is important, or at least give me default
arguments. Mentioning arguments, named arguments are way cool.</dd> 

<dt>13. Concurrency</dt> 

<dd>I use this term loosely. Generators and Coroutines somehow fit into
this as well. The point is not to have multi-threading built-in, but to
have multiple code executions work together conveniently. They don't need
to execute at the same time, but it should be possible to work on several
parts of your data set at the same time. Structuring your application
around event processing should be easy. If it's a mechanism for true
multiprocessing, all the better. (Example: Perl5's POE, Pythons
generators)</dd> 

<dt>14. Strings and Unicode</dt> 

<dd>It is f*cking 2011, we don't need anything else but Unicode
anymore. So please have a safe string type and Unicode all over, no
exceptions. I am sick of implicit conversions with accidental
double-encodings and <a href="http://www.geekwear.de/produkte/scheiss-encoding-shirt.html">whatnot</a>,
or manual conversions with no language support. I prefer UTF-8, by the
way. Who cares about constant-time indexing of strings? Use a char array
for that special case. Use regexes for the common cases. Have regexes
part of the regular string API!</dd> 

<dt>15. Low-Level Interface</dt> 

<dd>At some point, you will want to twiddle some bits manually. Especially
when targeting microcontrollers or embedded ARM cores, there should be a
way to get down to the bare metal. Ideally, it would be possible to
write an OS kernel in the language, with no assembler code at all
(except platform-specific startup code that can't be done any other
way)</dd> 

</dl> 

<h2>Enter Go</h2> 

<h3>Overview</h3> 

<p>I was a bit skeptical when I read about Google's new programming
language. I simply ignored the news. After all, the next New Great Language
is just around the corner. Some of them enjoy <a href="http://www.ruby-lang.org">a phase of hype, but then fade again</a>,
others <a name="d" href="http://www.digitalmars.com/d" id="d">stay in the spheres
of irrelevancy</a>, while yet others <a href="http://www.perl6.org">will be
ready for public consumption any decade now</a>.</p> 

<p>Some time later I stumbled over it again. This time I took a closer
look. One thing I didn't notice at first: One of the inventors is Ken
Thompson of Unix and Plan9 fame, and he was indirectly involved with C as
well. Now if a new programming language is designed by someone who already
served in the trenches of the great mainframe era, maybe there is something
to it.</p> 

<p>So what exactly can Go do what Perl, Python and JavaScript don't give me?
And what can it do what only those used to be able to do?  What makes it
different from all those failed or limited-success languages?  Will it still
be around in 30 years? And most importantly: Does it address my needs?</p> 

<h3>First Contact</h3> 

<p>The most important aspect of Go is the target audience. It was designed
as a systems programming language, so it aims at lower level software, maybe
even an OS kernel. Consequently, higher level constructs might be missing,
as they are complex and don't map well to hardware instructions. Funny thing
is, most of the convenience is actually there.</p> 

<p>Next thing you learn is, it is a language of C descent, so load your US
keyboard layout for easy access to curlies. But if you see some example
source code, it looks way less C-oid than expected. Less parens, not a
semicolon in sight, and few to no variable declarations, at least on first
sight. This is really light on syntax, with noticeably different keywords
and control structures, but still understandable.</p> 

<h3>Key Differences to C</h3> 

<p>For all those that are familiar with C/C++, let's have a quick comparison
of the differences:</p> 

<dl> 
<dt>No semicolons!</dt> 

<dd>No kidding! Well, actually there are semicolons, but they are
discouraged. It works like JavaScript, there is a simple rule that makes
the parser insert a semicolon at certain line ends. And that rule is
really simple, so it's easy to replicate in source-processing tools. For
that reason it is way less brittle than in JavaScript.</dd> 

<dt>The OTBS</dt> 

<dd>Next in category "pure heresy": Go defines a canonical indentation and
the One True Bracing Style. And <a href="http://www.gnu.org/prep/standards/standards.pdf">RMS is not going to
be happy about it</a>. This goes as far as supplying <tt>gofmt</tt>, a
tool to routinely reformat your source. Well, Java developers are used to
this, only they are stuck with a few braindead aspects (indent depth 2?
SRSLY?). Source formatting in Go can be summarized like this:
<ul> 
<li>Indent with tabs (which lets the
user twiddle his editor settings to get his comfortable amount of
horizontal white space)</li> 
<li>Braces go on the same line as the control statement they belong
to</li> 
<li>Continued lines must not end in closing braces or identifiers, i.e. make
the operator last on the old line, not first on the new line.</li> 
</ul> 

The third point is a consequence of the simple semicolon-insertion
rules. I wished there was a way around it, as I prefer having the
combining operator at the start of the continuation line, to emphasize
what's happening there instead of hiding it after lots of other stuff.<p></p> 

But apart from that, most of this is quite sensible. <a href="http://www.kernel.org/doc/Documentation/CodingStyle">Others have
explained this in greater detail</a>. The point is readability with less
visual noise. Like Python, only I think python has a tad too little visual
cues. Indentation alone isn't always sufficiently clear, so we get to keep
our beloved braces.</dd> 

<dt>Curlies mandatory</dt> 

<dd>Speaking of braces, there are no brace-free forms of <tt>if</tt> and
loops. Being a big fan of these, I think this is unfortunate. Coding style
purists may like that. But in the end, I don't care too much. For really
short statements, I can still do

<pre>if cur &lt; min { min = cur }</pre></dd> 

<dt>Less Parens</dt> 

<dd>One important technical reason for the previous point is the fact that
those control statements no longer have parens. Only Perl6 tries to
parse paren-less <tt>if</tt> without curlies, and we all know how
complicated Perl parsers used to be (and obviously, still are). So it's
actually a swap in what is mandatory and what not. Since you need braces
anyhow in most cases, this is quite sensible. It's unusual to read, you
have to adapt to not having those visual delimiters, but once accustomed
to it, Go code feels much lighter than C code.</dd> 

<dt>Explicit naming of types, functions and variables</dt> 

<dd>For introducing these, the keywords <tt>type</tt>, <tt>func</tt> and
<tt>var</tt> are used. This is much clearer, you get a better reading
"flow". For a more technical reason, read on.</dd> 

<dt>Implicit declarations, automatic typing</dt> 

<dd>Variables are always statically typed, just like in C. But they can
look as if they weren't. If you leave out the type, the type is instead
taken from the assignment. You may even leave out the declaration
altogether by using the new declare-and-initialize operator:

<pre>foo := Bar{1,2,3}</pre> 

This declares a variable called <tt>foo</tt> and assigns an object of type
<tt>Bar</tt> to it, using the object initializer syntax. This is
absolutely the same as

<pre>var foo Bar = Bar{1,2,3}</pre> 

It does not introduce dynamic typing, it does not allow changing a
declared variable's type, it does not remove the need to declare
variables, it does not allow you to declare variables twice. It's really
the same as before, semantically, but much lighter on syntax. Feels like
an ad-hoc scripting language but still gives you the benefits of static
typing.</dd> 

<dt>Variable declarations are backwards</dt> 

<dd>In Go, the type of a variable and the return type of functions follow
the name. In C, you can easily shoot yourself:

<pre> 
int* ptr_to_amount, amount; // a pointer and an integer
int* ptr1, ptr2; // uh-oh, this isn't what it seems to be
</pre> 

Go reorders things so types containing pointer and array
specifiers now read a lot better. And a single declaration now only
declares a single type of variable, so the above problem can't occur:

<pre> 
var ptr1, ptr2, ptr_to_amount *int
var amount int
</pre> 

This makes even more sense with the previous feature allowing you to leave
out the explicit type specification.</dd> 

<dt>Pointers without arithmetic</dt> 

<dd>There still are pointers, but they serve as plain references to values
now. All objects are value types, so assignment copies whole
objects. Pointers give you the reference semantics which is default for
Java. But there is no pointer arithmetic. You are forced to express array
accesses as such, and this eliminates a whole slew of security-related
problems. Yes, Sir, we do have bounds checking!<p></p> 

So pointers are no longer a central part of algorithms. They serve the
single purpose of specifying reference vs. value semantics. This is
simplified by the fact that pointers have no special dereferencing member
accessor. <tt>foo.Bar()</tt> works with pointers and values likewise.</dd> 

<dt>Garbage Collection</dt> 

<dd>The previous point makes much more sense if you were able to pass
around each and every pointer safely, and if you were able to take the
address of each and every value, like you cannot do in C and C++.<p></p> 

And you can: Memory management is handled by a garbage collector. Finally!
The benefit of an integrated garbage collector over a bolt-on one like
boehm-gc is that you can safely pass a pointer to a local variable or even
take the address of a temporary value, and it will just work! Yay!<p></p> 

For all those out there who are not up-to-date with garbage collection
research, you may be interested in the fact that GC is not just safer to
use as it completely avoids the myriad mistakes in using malloc/free. A
decent garbage collector can actually be faster than manual memory
management by postponing bookkeeping until there is time and completely
avoiding bookkeeping by reusing unused objects. Some of the most advanced
GCs combine this with better memory efficiency and cache usage because of
less fragmentation. This isn't the <a href="http://www.atarimagazines.com/compute/issue49/422_1_Garbage_Collection_On_Commodore_Computers.php">C64</a> 
anymore. <p></p> 

Go has a rather simple GC right now, but a more advanced implementation is
in the works. For most cases, GC is a win, but it does have a certain
overhead. In the few critical cases you can use arrays of preallocated
objects.</dd> 

<dt>Variable-Length Arrays</dt> 

<dd>This is not about arrays whose size is determined at runtime but
static thereafter. This is about things you would have to use
<tt>realloc()</tt> for. Arrays always have constant size, which is a
little step backward from GNU-extended C. But as a replacement you get
slices.<p></p> 

Slices look and feel like arrays, but in reality they just map to a
sub-range of a plain constant-size array. Since we have a garbage
collector, slices can reference an anonymous array. That way you get true
dynamically-sized arrays. There are built-in functions for resizing slices
and replacing the underlying array if it gets too small, so writing a
vector class on top of that is trivial.</dd> 

<dt>Reflection</dt> 

<dd>Go supports Reflection, i.e. you can look at arbitrary types and fetch
their type information, structure, methods and so on. This is in line with
Java and C++ RTTI but doesn't exist in C.</dd> 

<dt>Unsized Constants</dt> 

<dd>Constants can be untyped, or rather, unsized. A numeric constant can
be used in any context where that kind of number is valid, and it will use
the precision of the data type it is assigned to. Constant expressions are
calculated with full precision and only then cast to the destination
type. There are no constant size suffixes like in C.</dd> 

<dt>Error Handling</dt> 

<dd>Go has no exceptions. Wait—are you serious? This defies all
generally accepted knowledge about safe and stable programming! Isn't that
a huge step backwards?<p></p> 

Turns out it isn't. Be honest, when did you actually use exceptions to do
fine-grained error checking and handling? Most of the time, it's like
this:

<pre> 
try {
openSomeFile();
doSomeWorkOnTheData();
yadda...
yadda...
yadda...
closeItAgain();
} catch (IOException foo) {
alert("Something failed, but there's not enough time to do proper error handling");
}
</pre> 

This is verbose, introduces another indentation level for little benefit,
and it doesn't cure the root cause, the programmer's laziness. If you want
to handle errors for each call individually, verbosity gets so bad it
really impairs code clarity.<p></p> 

So we can as well do away with the code flow complexity and return to
old-school on-site error handling. Return values have been stigmatized for
years, but Go has this nice modern feature of multiple return values. So
forget the utter braindeadness of <a href="http://www.kernel.org/doc/man-pages/online/pages/man3/atoi.3.html">atoi()</a>,
we have proper out-of-band signaling. For those who care. For those who
don't, they didn't even care when Java tried to enforce error
handling.<p></p> 

Then there is <tt>panic</tt>. It is reserved for the "can't possibly
continue" style of errors. Serious initialization errors, conditions that
threaten the integrity of your data or your calculations, that kind of
problem. Unrecoverable errors, in short. The language runtime may also
create panics, like when array bounds are exceeded.<p></p> 

Of course, this brings us back to the problem of cleaning up used
resources. For this, the <tt>defer</tt> statement exists, and it is quite
a beauty, putting error handling where it belongs, right to the site of
the problem:

<pre> 
handle, err := openSomeFile()
if err != nil { return nil, err }
defer closeSomeFile(handle)

return happilyDoWork()
</pre> 

<tt>defer</tt> is almost like a <tt>finally</tt> clause in Java, but it
looks like a decorated function call. It makes sure that
<tt>closeSomeFile</tt> is called, no matter how the function is exited. As
a side effect, you can skip closing it upon success. Less code
duplication, concise and visible error handling. Multiple <tt>defer</tt>s
are allowed, properly called in LIFO order.<p></p> 

For those cases where you do want to continue after a panic, there is
<tt>recover</tt>. Together, <tt>panic</tt> and <tt>recover</tt> can be
(ab)used to create general-purpose exceptions again. Since they obfuscate
program flow, the official recommendation is that non-fatal panics should
never cross package boundaries. There is no ideal solution for error
handling, so you get good support for both, and you should choose the
variant that is less complex for the task at hand.
</dd> 

<dt>Control Structures</dt> 

<dd>There is only the <tt>for</tt> loop left which can behave like a
<tt>foreach</tt> thanks to the <tt>range</tt> keyword. Given that
<tt>while</tt> was always a special case of <tt>for</tt>, and that syntax
got lighter (see above), that's fine by me. What's even better: you can
put labels on your nested loops and <tt>break</tt> out of multiple levels
using them. Finally!<p></p> 

And you still can shoot yourself in the foot with <tt>goto</tt>. Well, not
really, as the more evil things are simply forbidden. But those who like
to cheat a little for simplicity's sake can do so.</dd> 

</dl> 

<p>All this comes at only little overhead. There are a few wrinkles, which I
will cover later, but as a whole it is a great improvement over C. This
alone would already be sufficient for many happy nights writing procedural
code without objects.</p> 

<h3>Extensions</h3> 

<p>The true strength of Go lies in that which cannot be mapped to C, C++ or
any other language mentioned so far. This is what makes Go really shine:</p> 


<dl> 

<dt>Type-Based Objects vs. Encapsulation</dt> 

<dd>There are no classes. Types and methods on types are kind of
independent of each other. You can declare methods on any type, and you
can declare any type to be a new type, similar to <tt>typedef</tt> in
C. The difference to either C or C++ is that such a newly named type gets
its own set of methods, and that primitive types (rather, those based on
them) can as well carry methods:

<pre> 
type Handle int64

func (this Handle) String() string {
return "This is a handle object that cannot be represented as String."
}

var global Handle
</pre> 

In this example, <tt>global.String()</tt> can now be called. In effect, we
get a simple object system with no virtual methods. It has zero runtime
overhead, as it is actually just syntactic sugar.</dd> 

<dt>Duck-Typing vs. Polymorphism</dt> 

<dd>Type declarations cannot be used to make two distinct types look
alike. They are distinct types, and in a strictly typed language this
doesn't let you create a kind of generic type that is polymorphic. One
popular example in almost every language is the conversion of a value into
its string representation. The <tt>Handle</tt> type declares such a method
as is the convention by Go, but it doesn't show how you can act on any
type that has such a String method.<p></p> 

C++ uses inheritance (possibly with abstract base classes) and cast operator
overloading for this. Java's <tt>toString</tt> is part of its root class
and thus inherited, while other calling conventions are expressed through
interfaces.<p></p> 

Go uses interfaces exclusively. Unlike Java, however, you don't declare
that a given type conforms to some interface. If it does, it automatically
is usable as that interface:

<pre> 
type Stringer interface {
String() string
}
</pre> 

That's all we need. Automatically, <tt>Handle</tt> objects are now also
usable as <tt>Stringer</tt> objects. If it walks like a duck, quacks like
a duck, and looks like a duck, then it is, for all practical purposes, a
duck. And now the best part: This works dynamically. The interface
declaration need not be imported or even known to the programmer.<p></p> 

Whenever a type is used as an interface, the runtime builds a table of
function pointers for that interface through its run-time reflection
capability. So here we do have some runtime overhead. It is optimized,
however, so penalties are relatively small. Interface tables are only
calculated when actually used, and only once for each type. The run-time
overhead is completely avoided if the involved types can be determined at
compile time. Method dispatch should be slightly faster than Apple's
(already quite cool) Objective C dispatcher.</dd> 

<dt>Embedding of Types vs. Inheritance</dt> 

<dd>Types serve as a kind of class, but there are crucial differences, as
there is no inheritance hierarchy. In the previous example,
<tt>Handle</tt> doesn't inherit any methods from <tt>int64</tt>. You can
get something similar to inheritance by declaring a <tt>struct</tt> type
which embeds base data types within its body: (example shamelessly stolen
from <a href="http://golang.org/doc/effective_go.html">"Effective Go"</a>)

<pre> 
type ReadWriter struct {
*bufio.Reader
*bufio.Writer
}
</pre> 

This type has all methods that <tt>bufio.Reader</tt> has, and all that
<tt>bufio.Writer</tt> has. Conflicts are resolved through a simple
rule. This is not multiple inheritance! Both base types exist as
independent data objects within the composite type, and each method from
the subtype only sees its own object. That way, you get perfectly
predictable behaviour without all the woes associated with classic
multiple inheritance. And without all the overhead - it again is more or
less syntactic sugar, making code more expressive without the runtime
cost.<p></p> 

This also works well with interfaces. The composite type conforms to all
interfaces that at least one constituent type conforms to. Of course, this
can lead to nested composition, making this a kind of inheritance. The
rules that resolve ambigious method references are quite simple, and
non-intuitive cases are simply forbidden: If two methods are the same at
the same nesting level, it is a compile-time error. Otherwise the one with
the lowest nesting level wins. As a result, the composite type may freely
override methods of its constituents.</dd> 

<dt>Visibility Control</dt> 

<dd>The primary unit of development is the package. One or more files
implement one package, and you get control over what is visible from
outside the package. There is no complex visibility system, you only get
visible or not. This is controlled by a typographic convention: Public
names start with a capital letter, private names with a lower-case
one. This works for everything that is named.<p></p> 

This is quite a pragmatic solution. Unlike the type system, which is at
least as powerful as the competition, here Go occupies a kind of middle
ground: Most scripting languages don't care about visibility at all or
rely on voluntary conventions, while old-school C-oids have detailed
accessibility controls. Still, it seems to be a good solution for Go's
object model. Since there is no classic inheritance and embedded objects
stay fully encapsuled, there is no need for a <tt>protected</tt> access
specification. How this works out in practice remains to be seen.</dd> 

<dt>No Constructors</dt> 

<dd>The object system has no special constructors. There is the notion of
the zero value, i.e. a zero-initialized value of all fields of your
type. You are expected to write your code in a way that the zero value is
a meaningful representation of a valid "blank" object. If that isn't
possible, you can provide package functions as constructors.</dd> 

<dt>Goroutines and Channels</dt> 

<dd>This is the most unusual feature for such a general purpose
programming language. Goroutines can be seen as extremely light-weight
threads. The Go runtime maps these to <a href="http://www.gnu.org/software/pth/">pth</a>-like cooperatively
multitasked pseudothreads or real operating system threads, the former
with low overhead, the latter with the expected non-blocking behaviour of
true threads, so we get the best of both worlds.<p></p> 

Channels are typed message queues which can be buffered or unbuffered. A
simple deal, really. Stuff objects into them at one side, fetch them
somewhere else. Most things you'd do in concurrent algorithms don't need
any more.<p></p> 

Go goes to great lengths to make Goroutines a first-class citizen, one
that can form the core of many algorithms. Segmented stacks keep
per-thread minimum stack usage low, and unless you use blocking system
calls, you get the performance benefits of pseudothreads, i.e. little more
overhead than a simple function call.<p></p> 

Combined with the fact that Go has real closures, Goroutines can be used
for lots of things that aren't classic concurrent algorithms. Pythons
generator functions can be modeled with them, or a custom memory manager
with a free object list. Read the online docs, it's amazing how versatile
cheap concurrency is.<p></p> 

By the way, by setting an environment variable, you can tell the Go
runtime how many CPU cores you want to be used so that Goroutines will be
mapped to several native threads right from the start (as opposed to the
default of not starting a second thread until a blocking system call is
about to be called)</dd> 

</dl> 

<h3>Regressions</h3> 

<p>No system is perfect. There are drawbacks to Go, here is a list of things
I have encountered so far:</p> 

<dl> 
<dt>Binary size / Runtime requirements</dt> 

<dd>A basic Go binary is statically linked and about 750k in size, if
built without debug symbols. This is about the same size as a similar C
program. I have tested this with the Tree Comparison example available on
the <a href="http://www.golang.org">Go Homepage</a>, comparing it to a
structurally similar C implementation I whipped up.<p></p> 

gccgo can compile dynamically linked executables, but libc is on every
system and not usually considered a dependency, while libgo would be an
extra 8MB package. For comparison: libstdc++ is less than 1MB, libc is
less than 2MB. To be fair, they do a lot less than the Go standard
library. Still, it's a big difference, and a dependency.<p></p> 

6g/8g, the original Go compiler, produces similar executables, but they
don't even depend on libc, these are truly standalone. No dynamic linking
of the runtime is possible, however.<p></p> 

This is also of concern for small systems. Sitting right next to me is an
ancient 16MB Pentium-100 laptop running X and a JWM desktop quite happily
and just managing to play my music collection. It even has 5MB memory left
for disk cache. Would that be possible with a system written in Go?</dd> 

<dt>No Equal Rights</dt> 

<dd>The language is privileged in several places. For example, the special
<tt>make()</tt> function does things that can't be extended from user
code. This isn't as bad as it looks at first, as you can write something
that behaves almost like <tt>make()</tt>, there is just no way to plug
into this language construct. Same goes for some other calls and keywords
that would make sense to be extensible, like <tt>range</tt>. You are more
or less forced to use goroutines and channels for extending the latter
one.<p></p> 

I'm not sure this is actually a problem. Assuming maps, slices, goroutines
and channels are implemented optimally, the impact of these restrictions
is nonexistant. It doesn't impair readability or clarity of code, but it
feels unfair if you are used to a "mimick anything" language like Perl or
Python.</dd> 

<dt>No Overloading</dt> 

<dd>Overloading is a source for many semantic ambiguities. That is a good
reason to leave it out. But at the same time overloading, especially
operator overloading, is so damn convenient and readable, so I really miss
it. Go doesn't have automatic type conversions, so things would not get
nearly as hairy as in C++.<p></p> 

As examples what overloading could be used for, imagine a BigNum library,
(numeric) vectors, matrices, or limited-range guarded data types. When
dealing with cross-platform data exchange, being able to change math
semantics for a special data type would be a big win. You could have a
1s-complement number type if dealing with data files from ancient
machines, for example, and do arithmetic that exactly mimics a target
platform instead of hoping that the current platform's semantics won't
differ.</dd> 

<dt>Limited Duck Typing</dt> 

<dd>Unfortunately, Duck Typing is incomplete. Imagine an interface like this:

<pre> 
type Arithmetic interface {
Add(other int) Arithmetic
}
</pre> 

The function parameter and return value for Add will limit the automatic
typing. An object that has a method <tt>func (this MyObj) Add(other int)
MyObj</tt> does not conform to <tt>Arithmetic</tt>. There are more
examples like this, and for some of them it's not easy to decide if Duck
Typing should cover them or the current rules are better. You can get into
a lot of non-obvious trouble, so again it's a case of "maybe it's better
we keep it simple", but again I am not totally convinced.<p></p> 

Russ Cox, one of the core authors of Go, states:

<blockquote>That doesn't work because the memory layout of a MyObj is
different from the memory layout of an Arithmetic.  Other languages
struggle mightily with this even when the memory layouts match.  Go just
says no.</blockquote> 

I guess we need to declare a <tt>func (this MyObj) Add(other int)
Arithmetic</tt> instead. A compromise which gains us simplicity of the
compiler and of the generated machine code.</dd> 


<dt>Pointers vs. Values</dt> 

<dd>I am not sure if I am happy with that pointer/value business. The
all-things-are-references semantics of Java are much simpler. The C++
reference vs. value syntax is also quite nice. On the positive side is
that you get more control about memory-layout and -usage of structs,
especially when they contain other structs, and that value-vs.-reference
semantics are more explicit at function call sites, which C++ made
unpredictable.<p></p> 

By the way, maps and slices <em>are</em> reference types. I was bothered
about them at first, but you can make your own objects that behave
similarly: structs that contain (private) pointers, which is more or less
what maps and slices are. Now if only there was a way to hook into their
<tt>[...]</tt> syntax...</dd> 

<dt>Boolean Context</dt> 

<dd>A boolean context offers lots of possibilities of simplifying
code. Unfortunately, even pointers must be compared to <tt>nil</tt>, even
though <tt>!pointer</tt> is totally obvious. Even more so since there is
no pointer arithmetic. Also see Wishlist item 10 above. This would make a
whole lot of sense and make code short and to the point.<p></p> 

Given that there is already the notion of a <a href="http://golang.org/doc/effective_go.html#data">zeroed value</a> for
every type, it's trivial to extend this to boolean context.</dd> 

</dl> 

<h3>Missing Things</h3> 

<p>Some things didn't make it into Go, and I really miss them. I hope the
community will find a way to add them, in the lightweight Go style of
course. I would like to see several less significant features, but some
things are wishes, and one thing I really want, and that's
metaprogramming.</p> 

<p>I'm not talking about Generics or Templates. Interfaces can replace these
more or less, although due to the limitations outlined above, Duck Typing is
an incomplete replacement right now.</p> 

<p>I'm talking about the real thing, the Code-that-Generates-Code
variety. Imagine the possibility to implement domain-specific languages. If
I do SQL, then SQL will be part of the source code no matter what. An SQL
package could offer an integrated solution of writing SQL, compile-time
syntax- and type-checked. It may increase compile times, but I'd rather
choose static checking of all of my code over finding out at runtime that
there is a syntax error in the SQL (plus paying the parse time at every
invocation instead of once).</p> 

<p>Such a facility also could improve the evolution of the language, making
experimental features easily implementable and testable before considering
them for inclusion into the core language. Given a well-designed
implementation, of course. No one wants the C/C++ mess all over again.</p> 

<p>Operator/method overloading would make much sense with that, so count
these in. Together Go would gain a lot of expressiveness. The Python model
with specially named methods has decent semantics and solves real-world
problems, for example.</p> 

<p>Unfortunately, this has been discussed lots and lots of times already,
and for a newcomer like me, there is simply too much discussion and too
little results to see where this is going. To quote one poster:</p> 

<blockquote> 
&gt; Feedback is very welcome.<br /> 
<br /> 
Search the list for each of those topics and find 1000s of e-mails for each.
</blockquote> 

<p>This is troubling. If certain topics have been discussed so many times,
why is nothing of this documented officially? Arguments like these
discourage potential contributore. At the same time, regulars get tired of
replying.</p> 

<p>At best, there would be a community proposal process (like Python, Java,
Perl6, XMPP, etc. have) which tracks these suggestions, outlines
requirements that must be fulfilled for a given proposal to be seriously
considered, and summarizes the current state of development. Then, ideas can
mature, can be rejected with comprehensible arguments, can be replaced by
even better ideas, and finally make it into the language without
compromising its design goals. From well-documented rejections, potential
contributors can learn what to avoid and what not to expect, and what to do
instead.</p> 

<p>This need not be democratic. There have been as many successes of the
"Bazaar" approach as for the "Benevolent Dictator" approach to project
management, and equally failures for both. More important is that there is
such a process, and that it is transparent and comprehensible.</p> 

<p>Everything said, don't get a wrong impression. The community is not of the
stubborn, arrogant kind. They do listen. It's more like another poster said
on the mailing list:</p> 

<blockquote>The problem you run into is that once you add a feature, you can
never take it back.</blockquote> 

<p>I totally subscribe to that. I want metaprogramming, it is so incredibly
convenient, but it has to be fully in-line with Go's strengths. We can leave
half-assed crap for PHP.</p> 

<h2>Open Questions</h2> 

<p>These bits are unclear to me right now. As I get or find answers to these
questions, I will update this review. Some of these are largely hypothetic,
while other are of direct interest. Judge yourself. Text in italics is my own
summary of the answers I got, they aren't quotes unless identified as such.</p> 

<ul> 
<li><p class="question">Does inlining work? How could cross-package
inlining work?</p> 

<p class="answer">Not yet, but it is on the compiler to-do. Cross-package
inlining is easy to provide, so the C++ code-in-header-files mess is
finally over.</p> 

I assume this applies to 6g/8g. No idea if gccgo will get cross-package
inlining as well.</li> 

<li><p class="question">Would it be possible to make Go 8-bit capable? C
scales down to 8-bit microcontrollers. What is the minimum hardware a
(possibly limited) Go runtime can work?</p> 

<p class="answer">Not with the current compilers. 32-bit is minimum.</p> 

I hope that someone will try to port Go to less capable machines some
day. This may take a while, the C64 didn't get C overnight either ;)</li> 

<li><p class="question">Can Go object files be loaded at runtime? Do types
implemented in these files integrate seamlessly into the hosting
application?</p> 

<p class="answer">Not yet, but there's no fundamental reason why it
couldn't be. This might even be on the roadmap already.</p></li> 

<li><p class="question">What are the limitations of the optimizer?
Assuming gccgo, does it leverage a similar level of optimization as when
compiling C++, or are significant steps unsuported?</p> 

<p class="answer">For 6g/8g, the optimizer still has a few omissions that
are standard fare in compiler technology, but it's being worked on. No
answer about gccgo yet.</p> 

This is actually good. The compilers aren't top notch yet, but still Go
performs nicely. Go was designed to make static optimizations easy, which
is why some features are intentionally limited. Looks like this will work
out as intended.</li> 

<li><p class="question">Would it be possible to detect integer overflows,
another well-known security defect? With operator overloading, I guess it
would be easy to implement a guarded data type. Still, it may be
worthwhile to have this at the core of the language, given an
implementation with decent performance is possible. C# does this
already.</p> 

<p class="answer">Not at the moment. Go explicitly defines integer
arithmetic to be wrapping. C made it undefined, so a C compiler can do
range checking if it wants to, a Go compiler can't.</p> 

Which means that the language specifications would have to change in some
way to make this feasible. IMO it is worthwhile to have a guarded integer
as default, and a wrapping integer as option for math tricks.</li> 

<li><p class="question">How high is the memory overhead of interfaces? Of
structs? Of arrays?</p> 

<p class="answer">In a nutshell, as optimal as can be. There is the type
information for every type, and interface values need up to 2 pointers,
but other than that, there is no overhead.</p> 
</li> 

<li><p class="question">How high is the CPU time overhead of an interface
method dispatch? Of creating a goroutine? Of scheduling
goroutines?</p> 

<p class="answer">Interface method dispatch is like a C++ virtual
function. Creating a goroutine only allocates some memory. Actually
scheduling goroutines is via a simple round-robin scheduler. So it's quite
fast, but it may not work well on some workloads.</p> 
</li> 

<li><p class="question">Is it possible to add methods to types not
declared in the current package? Maybe even to int64?</p> 

<p class="answer">No, and as a consequence, no. Only the package that
declares a type may add to it. So, no mixins. This is intentional to make
absolutely clear what the code does. Otherwise it would depend on the
packages imported over the lifetime of a program, and once there is
dynamic loading, non-determinism galore!</p></li> 

<li><p class="question">Is it possible to add Methods to interfaces,
i.e. a method that has the interface as receiver and thus is the same for
all objects implementing that interface?</p> 

<p class="answer">No, for a similar reason. Write package functions
instead that operate on the given types. Since functions are properly
namespaced, they are not Evil.</p></li> 

<li><p class="question">About scheduling goroutines: Can a goroutine
starve all other goroutines and the main program if it runs into an
endless loop, or is there some sort of preemption as a safeguard?</p> 

<p class="answer">There is no safeguard. Goroutines do not replace threads
for low-latency concurrency/parallel processing.</p> 

On one hand, this is unfortunate. On the other hand, cooperative
scheduling is a lot less complex and has better throughput. Just take a
look at the Linux kernel and see how many iterations its scheduler has
went through to get optimal performance. Low-latency, low-overhead,
scalable, starvation-free and fair preemption is hard, and then you have
replicated an operating system facility. Perhaps it's smarter to live with
this and add some way of spawning a goroutine in its very own thread.
</li> 

<li><p class="question">What about the garbage collector? How well does it
handle large numbers of small objects?</p> 

<p class="answer">Memory-wise about as efficient as malloc(). Speed can be
impaired if many objects are only referenced via pointers to inside them
near their end (i.e., slices with that characteristic).</p> 

Note that this is currently a simple stop-the-world collector. A
replacement will come. There is nothing fundamental preventing GC to be
pluggable, it simply isn't there yet. Priorities are at getting the
language itself as optimal as possible.
</li> 

</ul> 

<h2>Feedback</h2> 

<p>There has been quite some feedback, so I added this section. It addresses
interesting remarks that didn't fit into my main train of thought. Remarks
that clarify my original text or that made me go "Yeah, of course, how could
I miss that?" were integrated into the text above.</p> 

<h3>Ternary operators</h3> 

<p>One reader detailed the history of Pythons ternary operator <tt>a if foo
else b</tt>. Given that python already is high on the readability scale, one
may wonder why they added this construct which is usually frowned upon.</p> 

<p>Seems like Python didn't have it for a while, and people lived happily
with rich booleans alone: <tt>some_value or "default-value"</tt>. Up until
someone had the idea to do <tt>condition_to_test and value_if_true or
value_if_false</tt>. From there it went downhill. That hack has drawbacks,
so other hacks were invented. Each one uglier than the previous, until
finally, Python got a ternary condition operator.</p> 

<p>What does this mean for Go? Well, if you can't fight it, embrace
it. Please.</p> 

<h3>D</h3> 

<p>I was not going to talk about D for a good reason. I didn't miss it, but
some people didn't discover my <a href="#d">6-word summary of my opinion
about D</a>. Rather, I don't have experience in it. I didn't write
<em>about</em> D because I didn't write <em>in</em> D. So now, by popular
demand, my uninformed opinion about D:</p> 

<p>When I first encountered D (on <a href="http://www.heise.de">heise.de
news</a>, you can find select articles in english on <a href="http://www.h-online.com/">The H</a>), I was quite impressed. There was
a language that really tried to address the deficiencies of C/C++/Java. But
after skimming the docs for a while, I lost interest.</p> 

<p>Why? Simple, because it didn't scratch the real itch. Yes, it offers
everything in a single language which must be replicated in C++ with
templates and which in Java has made it into yet another abstraction layer,
but that's the point: Been there, done that. It may be less efficient in
those languages, but they map to D pretty straightly.</p> 

<p>While the all-features-in-a-unified-language approach is interesting, I
can't see where it fits into my universe. It doesn't try to be slim. It
doesn't offer any new concept worth exploring. It simply doesn't address my
needs, so why should I switch and take the hassles of not having code that I
can share with others, code that I can publish as open source and easily get
into public use, code that interfaces easily with other people's code?</p> 

<p>If I embark on a big project that needs all the bells and whistles, I
would chose Java or C++, depending on (among others) what language
interoperates best with the project requirements, especially foreign code. D
simply won't appear in there. If it's a small project, D's qualities do
nothing for me. The incentive to switch simply isn't there. Additionally, I
dislike the "heat-seeking" approach of putting everything into the
language. It's a corporate language, but corporate does Java and C#.</p> 

<p>You know what the funny thing is? D is the reason why I rejected Go at
first. I read about Go at the same place, and I thought "Oh well, there goes
another one..." and ignored the news. Only when I read about it two more
times, some fact (which I don't remember) sparked my curiosity, and two
weeks later I am writing this review.</p> 

<p>Go actually is different. It gives you a reason to switch. It addresses
my needs and gives me something no other general-purpose language has. No,
Haskell, Erlang etc. don't count. I like those from a scientific point of
view, but I'd like to still understand my code when I'm tired, and I'd like
to be able to make others easily understand my code. But most importantly,
Go interoperates well with the outside world thanks to gccgo, and I will
gain by using it, even if it should stay a relatively unknown language.</p> 

<h3>This Article on teh Internetz</h3> 

<p>Thanks to The Internets for all the discussion I didn't notice and had to
be pointed to. There has been a lot of speculation about various aspects,
and I found it entertaining. You didn't disappoint me. It's unlikely you
will read this, but just in case, some replies to things not already covered
in today's updates:</p> 

<p>One very inportant thing many people missed is that a programming
language is a tool. Once upon a time, I was deep into Perl 5, learning all
the black arts over time. But do you know what was the real reason to stay?
It wasn't the beauty of the language that fascinated me (there is
little). Partly, it was the meta-trickery you could pull off and get away
with it, that appealed to my hacker heart. But the real reason was the
package archive and the ad-hoc way you could Just Write Code. These enabled
me to solve real world problems in less time than otherwise.</p> 

<p>That's why I program AVRs in C, GUI stuff in PyQt, really old GPUs in
ARB_vertex_program assembler instead of GLSL, lightweight GUI stuff in
C++/Fltk, Emacs in Lisp, my homebuilt embedded car MP3 player in C after
prototyping it in Perl, an ARM decompiler in Python, and yes, web services
meant to be maintained independently in PHP. I hate PHP, but it's the right
tool for that job. So get over it and begin producing things with what works
for you, check the whole picture, and keep rechecking it.</p> 

<p>This is what I expect of Go, and currently it seems to fulfill that. I
have seen lots of things that can go wrong over time, no one and nothing is
safe. But I am a chronic early adopter. And the things I do, I do as hard as
I can. If you want to know how it worked out, watch this space. All I can
tell is that it's worthwhile to continue, something that few languages gave
me.</p> 

<p>One thing I am totally curious about are the hints at better language
design, often in sentences like "... ignores 20+ years of research." Vague
suggestions only work for well-known phenomena, not for specialist
topics. Anyone who can point me at results <em>relevant to Go</em> and which
fit into the constraints of the design goals, you are welcome. There is an
email link at the top!</p> 

<p><em>You may skip the rest of this section if only interested in
programming and Go. The remaining paragraphs are off-topic or meta.</em></p> 

<p>To all of you whose toes I have stepped on in my rant: It's a rant. It's
not meant to be totally objective. It uses sarcasm and exaggregation. Some
people got it, some didn't. Usually, the people who got it were the ones who
made similar experiences as I did. And boy was it good to vent publicly. I
still like C, I will still use C++ and Java when they are the correct tool
for the job, but this doesn't mean they are what make my programmer heart
happy ever after.</p> 

<p>By the way, there <em>is</em> a difference between the phrases "sth. is
not really suitable for sth." and "it is impossible to successfully do
sth. with sth.", even when they appear in the vicinity of the number
10000.</p> 

<p>Those two people who accused me of not getting OO respectively
Concurrency right, care to elaborate? I can't see where there is a
fundamental misunderstanding, save for some simplifications. Educated
readers will know the true details, while people without the theoretic
background will get the general idea of what I am referring to.</p> 

<p>To the Perl6 fan who was offended by my sidestab at the development time
that Perl6 took: Rest assured that I am still, in general, a Perl fan. I
have Rakudo Star installed on my machine, and tried to write a serious
utility with it in order to see whether it was any good. Just the way I did
with Go. And no, Perl6 is not ready for public consumption yet, although I
must admit, it's getting close.</p> 

<p>To all nitpickers about the classification and choice of languages I rant
about: Of course I can only talk about what I know. Nowhere does it say I
rant about the C family or even Algol-descendant languages. In the
introduction I explicitly mention several other languages that have been
some system's main programming language, and that I restricted myself to the
most well-known ones. Yes, JavaScript belongs there. It is one of the most
widely deployed languages, it is more than relevant.</p> 

<p>Finally, to the guy who didn't take me serious because of not getting
"it's" vs. "its" right: I am terribly sorry. You are right, there was one
instance in this 65kB text where I mistyped it. It is corrected, I hope you
are now able to read it. Feel free to point me at other typos, since I don't
use any kind of spell checking (except of you, obviously).</p> 


<h2>Conclusion</h2> 

<p>Anyone comparing the appearance of Go source code to his favourite C-oid
gets a chance to express her basic attitude: Is Go partly like what you
love, or partly like what you hate? It has a bit of annoyance for everyone,
which is what you get when trying to do better. It creates an own feeling,
so you can't claim it just mimics some other language. Well done!</p> 

<p>In the end, we get a simple and clean type system that is easy to learn,
but unusual. It might be a major obstacle in selling Go to CS beginners,
who usually get taught classic OO principles. Its simplicity and safety,
however, make it well-suited for self-taught programmers.</p> 

<p><a href="http://golang.org/doc/effective_go.html">"Effective Go"</a> is a
great document to read if you come from a classic OO background. It's
important to have such a document, but it is not complete
enough. Researching and writing this review has taught me more about Go than
writing the application which I did in parallel. There are so many things
you are used to do which Go does differently but equally well (or even
better), and I wasn't aware of them. There is a constant feeling of "Go
can't do X" while it actually can do it well, only way differently.</p> 

<p>For a fair image of what Go's potential is, note the age of Go. The first
release was less than 2 years ago and declared stable enough since this
year. Look at what Go already does today, and imagine what would be possible
if Go had the same commercial backing as Java or JavaScript have. The best
example of a successful introduction of a new language is Java, and now
compare Go's feature set to that of Java 1.0. We have a winner here.</p> 

<p>But to leverage that potential, the language needs some momentum. Either
through an open, active and growing community, or through corporate
backing. I'd prefer the community, but for real-world success, there
probably has to be some corporate involvement. Bonus points if Oracle messes
up the Java business even more :)</p> 

<p>Really, Go can be the answer to the shortcomings of all currently popular
system programming languages, it just needs adoption.</p> 

<p>And as a final note, I have seen a fair amount of criticism of Go on the
internet, which I cannot ignore, so here it goes: Most of these people
didn't actually look at it. Go is different, even though it still looks
kinda-C. It isn't. It's not C++, nor Objective C, and it doesn't try to be!
So stop saying "Who needs Go when we have C++/Objective C?" already. Check
out how Go tries to solve the same problems in a radically different
way. Accept the fact that OO can be done in different ways. You may have
opted to ignore it, but if you use JavaScript, you already use something
that isn't class-based OO. Do not just accept it, actively use the power of
that different approach. To the other ones, those who think Go isn't taking
this far enough: Remember this is a real-world language. And it is
there. And it works. What use is a beautifully constructed language that
doesn't get stable, finished or fast enough for real-world problems? It's
easy to nitpick on details, but to make it a real product, you need to
address all constraints. That's what Go does.</p> 

<p>And as a final final note: Big thanks to the community. Your feedback was
very valuable. Please continue to correct my mistakes, misconceptions and
everything, should there any remain. I hope this document will help other
potential users to get into Go. But for this, it needs to be correct. Please
be picky.</p> 

<h2>License</h2> 

<p><a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/3.0/"><img alt="Creative   Commons License" style="border-width:0" src="http://i.creativecommons.org/l/by-nc-sa/3.0/88x31.png" /></a><br /> 
<span xmlns:dct="http://purl.org/dc/terms/" href="http://purl.org/dc/dcmitype/Text" property="dct:title" rel="dct:type">The Go Programming Language, or: Why all C-like languages
except one suck.</span> by <a xmlns:cc="http://creativecommons.org/ns#" href="http://www.syntax-k.de/projekte/go-review" property="cc:attributionName" rel="cc:attributionURL">Jörg Walter</a> is
licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/3.0/">Creative Commons
Attribution-NonCommercial-ShareAlike 3.0 Unported License</a>.</p> 
