---
layout: post
title:  "Lambda, the ultime glue"
date:   2007-08-06 20:59:12
categories: programming javascript
comments: true
---
Mais uns códigos continuando as idéias do post sobre [closures]({% post_url 2007-06-25-closures %}), só que em JavaScript (acho que mais gente entende JavaScript do que Ruby).

Antes de mais nada: códigos e inspirações do [SICP][sicp-link] (video-aulas [aqui][sicp-videos]).

Bom, anteriormente `cons`, `car` e `cdr` foram definidos assim:

{% highlight javascript %}
function cons(a, b) {
    return function(first) {
        return first ? a : b;
    }
}

function car(c) {
    return c(true);
}

function cdr(c) {
    return c(false);
}
{% endhighlight %}

Com isso acabamos definimos um padrão de como representar listas e operar nelas, e o mais legal, tudo construido do “nada”, apenas com [lambdas][lambda-link]. Uma maneira um pouco diferente de definir essas três funções é dada a seguir. [Gerald Jay Sussman][gerald-link] chamou de “[Alonzo Church][alonzo-link]’s hack”:

{% highlight javascript %}
function cons(x, y) {
    return function(m) {
        return m(x, y);
    }
}

function car(x) {
    return x(function(a, d) { return a; });
}

function cdr(x) {
    return x(function(a, d) { return d; });
}
{% endhighlight %}

As duas implementações são praticamente equivalentes, só que essa segunda é feita excusivamente de _lambdas_ (na primeira temos um condicional na definição de `cons`) e além disso ela traz o controle para o `car` e `cdr`, e o `cons` passa apenas a ser um _dispatcher_. Agora conseguimos introduzir operações para alterar uma lista feita a partir de _conses_:

{% highlight javascript %}
function cons(x, y) {
    return function (m) {
        return m(x, y, function (n) { x = n }, function (n) { y = n });
    }
}

function car(x) {
    return x(function (a, d, sa, sd) { return a; });
}

function cdr(x) {
    return x(function (a, d, sa, sd) { return d; });
}
{% endhighlight %}

Agora, no _dispatcher_ (`cons`) adicionamos duas outras operações, que são usadas a seguir para alterar o valor do `car` ou do `cdr` de um _cons_:

{% highlight javascript %}
function setcar(x, y) {
    return x(function (a, d, sa, sd) { sa(y); });
}

function setcdr(x, y) {
    return x(function (a, d, sa, sd) { sd(y); });
}
{% endhighlight %}

As funções `list`, `mapcar`, `reduce` e `filter` permanecem como definidas nos outros posts:

{% highlight javascript %}
function list() {
    arguments.shift = Array.prototype.shift;
    if (arguments.length == 0) {
        return null;
    } else {
        return cons(arguments.shift(), list.apply(this, arguments));
    }
}

function mapcar(fun, lst) {
    if (null == lst) {
        return null;
    } else {
        return cons(fun(car(lst)), mapcar(fun, cdr(lst)));
    }
}

function reduce(fun, init, lst) {
    if (null == lst) {
        return init;
    } else {
        return fun(car(lst), reduce(fun, init, cdr(lst)));
    }
}

function filter(fun, lst) {
    if (null == lst) {
        return null;
    } else {
        if (fun(car(lst))) {
            return cons(car(lst), filter(fun, cdr(lst)));
        } else {
            return filter(fun, cdr(lst));
        }
    }
}
{% endhighlight %}

Uma nova função que podemos escrever agora que é possível alterar uma lista é `append`:

{% highlight javascript %}
function append() {
    var lst = null;
    var i = 0;
    for (; i < arguments.length && null == lst; i++) {
        if (null != arguments[i]) {
            lst = copylist(arguments[i]);
        }
    }
    for (; i < arguments.length; i++) {
        if (null != arguments[i]) {
            setcdr(last(lst), copylist(arguments[i]));
        }
    }
    return lst;
}
{% endhighlight %}

As duas funções auxiliares que `append` usa são:

{% highlight javascript %}
function last(lst) {
    if (null == lst) {
        return null;
    } else if (null == cdr(lst)) {
        return lst;
    } else {
        return last(cdr(lst));
    }
}

function copylist(lst) {
    return mapcar(function (x) { return x; }, lst);
}
{% endhighlight %}

Mas append não precisa ser implementada necessáriamente usando funções destrutivas, e de fato, mesmo nessa implementação temos que é uma função [pura][pure-link]. Usei append nesse exemplo porque é uma função simples e nos permite fazer algumas coisas legais como:

{% highlight javascript %}
function qsort(lst) {
    if (null == lst) {
        return null;
    } else {
        var pivot = car(lst);
        var nlst = cdr(lst);
        return append(qsort(filter(function (x) { return x < pivot; }, nlst)),
                      list(pivot),
                      qsort(filter(function (x) { return x >= pivot; }, nlst)));
    }
}
{% endhighlight %}

Ou mesmo:

{% highlight javascript %}
function powerset(lst) {
    if (null == lst) {
        return null;
    } else {
        var first = car(lst);
        var rest  = powerset(cdr(lst));
        return append(list(list(first)),
                      mapcar(function (x) { return append(list(first), x); }, rest),
                      rest);
    }
}
{% endhighlight %}


[sicp-link]: http://mitpress.mit.edu/sicp/
[sicp-videos]: http://swiss.csail.mit.edu/classes/6.001/abelson-sussman-lectures/
[lambda-link]: http://en.wikipedia.org/wiki/Lambda_calculus
[gerald-link]: http://en.wikipedia.org/wiki/Gerald_Jay_Sussman
[alonzo-link]: http://en.wikipedia.org/wiki/Alonzo_Church
[pure-link]: http://en.wikipedia.org/wiki/Purely_functional
