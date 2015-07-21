---
layout: post
title:  "Algoritmos e linguagens de programação"
date:   2007-08-07 20:59:12
categories: programming javascript
comments: true
---
Tem gente que acredita que algoritmos e linguagens de programação são dois mundos distintos. Eu discordo. São raríssimas as pessoas que conseguem separar as duas coisas, incluse eu e você que está lendo isso.

Pegue qualquer livro de introdução a algoritmos e estrutura de dados, pode até mesmo ser um [clássico][CLRS]. Os algoritmos são dados em uma linguagem de programação. Pseudo-código é baseado numa linguagem de programação. Todo pseudo-código que eu conheço é pseudo-C ou pseudo-Pascal (um pseudo-Algol-like), inclusive a maioria dos que eu escrevo (pelo menos até agora).

Quando pensamos num problema e estamos desenvolvendo um algoritmo para resolvê-lo estamos limitados às ferramentas de construção de algoritmos que conhecemos, que aprendemos. Mas mais do que isso, acabamos usando as que estamos mais acostumados. Um exemplo disso são algoritmos recursivos. Eu conheço um sem número de programadores com uma dificuldade tremenda de entender recursão. Eu sei que é um exemplo simples e que todo programador deveria saber recursão. Mas pegue uma linguagem mainstream, escolha um projeto qualquer open source e veja quantas definições recursivas você encontra.

Digo isso por experiência própria. Eu aprendi e entendi recursão como todo mundo no curso de computação, mas eu quase nunca pensava recursivamente. Quando eu fui programar em [Erlang][erlang] pela primeira vez, de cara o obstáculo foi *[single assignment][single-wiki]*. “Como assim eu não posso fazer i++?”. “Não dá pra fazer quase nada se não dá pra fazer i++!”. “Que linguagem tosca!!”. Acontece que dá para fazer de tudo sem i++ e coisas assim te forçam a exercitar outras formas de resolver problemas, como usar recursão e todo pensamento indutivo que você precisa para isso.

Mas como eu disse, recursão é um exemplo tolo. O grande problema que eu vejo é que o [Blub Paradox][blub] está diretamente ligado não apenas à linguagem de programação mas também na forma como pensamos e resolvemos problemas, ou seja, como desenvolvemos algortimos.

Agora um exercício de uma dessas coisas que a maioria das pessoas acha perda de tempo.

Segundo a Wikipedia Memoização é: “*… memoization is an optimization technique used primarily to speed up computer programs by storing the results of function calls for later reuse, rather than recomputing them at each invocation of the function…*“.

Memoização é uma técnica, assim como muitas outras coisas, como *backtracking*, como programação dinânima, como divisão e conquista. E porque não como *design patterns*? São técnicas.

Fica difícil definir memoização como um algoritmo em pseudo-código, mas não porque não seja possível, mas porque falta alguma coisa no pseudo-código, no pseudo-C. Memoização poderia se resumir simplesmente ao algoritmo memoizar que recebe como entrada uma função e devolve a mesma função memoizada. Tendo esse algoritmo seria possível escrever algo assim em [JavaScript][javascript-link]:

{% highlight javascript %}
function fib(n) {
    if (n == 0) {
        return 0;
    } else if (n == 1) {
        return 1;
    }
    return fib(n - 1) + fib(n - 2);
}

function test(n) {
    var t1 = new Date();
    var v = fib(n);
    var t2 = new Date();
    print(“fib(”+ n + “) = “+ v + ” took: “ + (t2 - t1) + ” ms”);
}

function runtest() {
    test(42);
    fib = memoize(fib);
    test(42);
}
{% endhighlight %}

Dessa forma seria natural pensar que utilizando a função `memoize` a segunda chamada a `test(42)` fosse mais rápida que a primeira. De fato, é o que realmente acontece:

{% highlight bash %}
fib(42) = 267914296 took: 279094 ms
fib(42) = 267914296 took: 20 ms
{% endhighlight %}

A implementação de `memoize` é simples numa linguagem de programação com suporte a [higher-order functions][higher-order-wiki], mas eu não sei fazer algo parecido numa linguagem como Java/C# (por exemplo).

{% highlight javascript %}
function memoize(fun) {
    var cache = new Array();
    return function () {
        arguments.toString = Array.prototype.toString;
        if (cache[arguments.toString()] == undefined) {
            var value = fun(arguments);
            cache[arguments.toString()] = value;
        }
        return cache[arguments.toString()];
    }
}
{% endhighlight %}

Essa é uma idéia de memoização que o [Peter Norvig][peter] apresenta no [PAIP][paip-link].

De novo, isso não serve para nada, além claro de mudar um pouco a maneira como pensamos em resolver problemas. Alguns que concordam:

> Lisp has jokingly been called “the most intelligent way to misuse a computer”. I think that description is a great compliment because it transmits the full flavor of liberation: it has assisted a number of our most gifted fellow humans in thinking previously impossible thoughts.

Edsger Dijkstra, CACM, 15:10
{: .right}

> Lisp is worth learning for the profound enlightenment experience you will have when you finally get it; that experience will make you a better programmer for the rest of your days, even if you never actually use Lisp itself a lot.

Eric Raymond
{: .right}

> Lisp … made me aware that software could be close to executable mathematics.

L. Peter Deutsch
{: .right}

> Although my own previous enthusiasm has been for syntactically rich languages, like the Algol family, I now see clearly and concretely the force of Minsky’s 1970 Turing Lecture, in which he argued that Lisp’s uniformity of structure and power of self reference gave the programmer capabilities whose content was well worth the sacrifice of visual form.

Robert Floyd, Turing Award Lecture, 1979
{: .right}

[CLRS]:              https://mitpress.mit.edu/books/introduction-algorithms
[erlang]:            http://www.erlang.org
[single-wiki]:       http://en.wikipedia.org/wiki/Single_assignment
[blub]:              http://c2.com/cgi/wiki?BlubParadox
[javascript-link]:   http://javascript.crockford.com/javascript.html
[higher-order-wiki]: http://en.wikipedia.org/wiki/Higher-order_function
[peter]:             http://norvig.com/
[paip-link]:         http://norvig.com/paip.html
