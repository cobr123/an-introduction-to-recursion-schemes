[English version (origin)](https://nrinaudo.github.io/recschemes/cata.html)

[Назад](./fold.md) | [Оглавление](./README.md) | [Дальше](./tree_height.md)

# Обобщённая свёртка

Мы обобщили структурную рекурсию, но только для List. Давайте теперь займемся задачей обобщения `fold` - как бы обобщения обобщенной структурной рекурсии.

Это текущая реализация `fold`:

```scala
def fold[A](
  base: A,
  step: (Int, A) => A
): List => A = {

  def loop(state: List): A =
    state match {
      case Cons(head, tail) => step(head, loop(tail))
      case Nil              => base
    }

  loop
}
```

Наша задача, таким образом, - попытаться удалить из этого кода все, что напрямую связано с `List`.

## Абстрагирование структуры

Давайте сначала посмотрим на ту часть, которая работает конкретно со структурой списка:

![Hard-coded structure](./img/cata-1-hl-1.svg)

Берем `List` и следуя его структуре получаем `Cons` или `Nil`, но на это можно посмотреть иначе: мы получаем `head` и `tail`... или ничего. И в Scala для этого есть специальный тип подразумевающий возможность отсутствия данных: `Option`.

Использование `Option` вместо `List` не решит нашу задачу - опциональный `head` и `tail` все еще очень похож на `List`. Но это шаг в правильном направлении: мы двигаемся от `List` к более обобщенному типу.

Конечно, чтобы работать с опциональным `head` и `tail`, мы должны превратить List в Option. Это широко известно как проекция, что в нашем случае является прямым сопоставлением с образцом: `Cons` превращается в `Some`, а `Nil` в `None`.

```scala
val project: List => Option[(Int, List)] = {
  case Cons(head, tail) => Some((head, tail))
  case Nil              => None
}
```

Это позволяет нам передать в `fold` функцию проекции и удалить прямые ссылки на структуру списка:

```scala
def fold[A](
  base   : A,
  step   : (Int, A) => A,
  project: List => Option[(Int, List)]
): List => A = {

  def loop(state: List): A =
    project(state) match {
      case Some((head, tail)) => step(head, loop(tail))
      case None               => base
    }

  loop
}
```

Однако это немного усложняет наше графическое представление поведения `fold`:

![Projection](./img/cata-2-hl-1.svg)

Предупреждение: все станет немного хуже, прежде чем станет лучше. Эта диаграмма будет расти еще немного, но если вы потерпите меня, она *станет* намного лучше.

В итоге.


## Упрощаем base и step

Когда вы вводите новый тип, всегда полезно проверить, не появляется ли он где-нибудь еще. Если это так, возможно, вы на правильном пути к поиску общей структуры, с помощью которой, если повезет, вы сможете абстрагироваться.

И в нашем случае, опциональный `head` и `tail` действительно появляется снова, хотя и несколько менее очевидным образом:


![Projection](./img/cata-2-hl-2.svg)

`step` принимает параметры `head` и `tail`, а `base` не принимает параметров, поэтому мы может объединить их вместе и получить функцию ожидающую на вход опциональный `head` и `tail`.

Давайте напишем эту функцию. Назовем её `op`, потому что я умею давать вещам ужасные имена, и следую правилу здравого смысла _называй плохим именем, пока не узнаешь, что оно на самом деле делает_:


```scala
val op: Option[(Int, String)] => String = {
  case Some((head, tailResult)) => step(head, tailResult)
  case None                     => base
}
```

Если помните, мы все еще работаем с `mkString`: наш обобщенный `fold` вызывается с конкретными параметрами, которые превращают его в функцию получения строкового представления списка. Держа это в уме:
* `step` это объединение текстового представления `head` и `tail`, разделенных `" :: "`.
* `base` это просто `"nil"`.

```scala
val op: Option[(Int, String)] => String = {
  case Some((head, tailResult)) => head + " :: " + tailResult
  case None                     => "nil"
}
```

Это позволяет нам переписать `fold` используя `op` вместо `base` и `step`:

```scala
def fold[A](
  op     : Option[(Int, A)] => A,
  project: List => Option[(Int, List)]
): List => A = {

  def loop(state: List): A =
    project(state) match {
      case Some((head, tail)) => op(Some((head, loop(tail))))
      case None               => op(None)
    }

  loop
}
```

И, конечно же, все по-прежнему ведет себя точно так же, как и раньше:

```scala
fold(op, project)(ints)
// res12: String = 3 :: 2 :: 1 :: nil
```

Но получившаяся диаграмма немного разочаровывает:

![Op, step 1](./img/cata-3-hl-1.svg)

Немного неприятно, что `op` появляется дважды.

Однако мы можем легко исправить это, осознав, что `op` появляется справа от всех ветвей сопоставления с образцом, что позволяет нам переместить его за пределы сопоставления:

```scala
def fold[A](
  op     : Option[(Int, A)] => A,
  project: List => Option[(Int, List)]
): List => A = {

  def loop(state: List): A =
    op(project(state) match {
      case Some((head, tail)) => Some((head, loop(tail)))
      case None               => None
    })

  loop
}
```

Это дает следующую диаграмму, которая немного загружена, но делает явным наличие наших необязательных `head` и` tail`:

![Op, step 2](./img/cata-4-hl-1.svg)

## Промежуточное представление

Давайте немного подумаем об этих необязательных `head` и` tail`. Я говорил `tail` не заостряя внимания на важных деталях, и сейчас самое время посмотреть на них внимательнее.

![Option of head, tail](./img/cata-4-hl-2.svg)

С левой стороны у нас есть хвост списка. Но с правой стороны у нас есть `А`. Это больше не конец списка, так что же он представляет?

Это помогает думать о `fold` как о механизме поиска решения задачи, где сама задача является входным `List`.

Например, `mkString` находит решение _какое текстовое представление у этого списка?_. Он идет от `List`, задача *до* решения, к `String`, задача *после* решения (также известная как решение).

И если вы думаете об этом в таком свете, то опциональный `head` и `tail` это очень конкретное представление структурной рекурсии. Возьмите `Option[(Int, List)]`. Он содержит:
- наименьшую возможную задачу: `None`, пустой список
- бóльшую задачу `Some`, разложенную на меньшую задачу, `tail`, и дополнительную информацию, `head`

Но `Option[(Int, A)]` немного другой: в случае `Some`, `tail` больше не меньшая задача, а её решение - текстовое представление списка. И это очень удобно! Вас просят решить задачу, предоставив:
- наименьшую возможную задачу: `None`, пустой список
- бóльшую задачу, разложенную на *решение меньшей задачи* и дополнительную информацию, `head`

Этот опциональный `head` и `tail` является чрезвычайно интересным типом, поскольку он позволяет нам представлять промежуточные этапы нашей свёртки (`fold`). Настолько интересный, что мы дадим ему имя: `ListF`.

```scala
type ListF[A] = Option[(Int, A)]
```

У этого неприятного названия есть конкретная причина. Часть `List` очевидна: `ListF` имеет какое-то отношение к спискам, так что давайте оставим это там. Однако я пока не буду объяснять букву `F`, чтобы не испортить интуицию, которую я надеюсь развить.

Мы используем `ListF` для представления различных шагов нашей свёртки (`fold`).

Во-первых, декомпозируем задачу на основные компоненты структурной рекурсии, которую мы получаем через `project`:

```scala
val project: List => ListF[List] = {
  case Cons(head, tail) => Some((head, tail))
  case Nil              => None
}
```

`List`, параметр типа для `ListF`: тип задачи до ее решения.

Но тогда, `ListF` также представляет собой декомпозицию задачи на дополнительную информацию и *решение* меньшей задачи. И мы можем перейти от этого к решению всей задачи с помощью `op`:

```scala
val op: ListF[String] => String = {
  case Some((head, tailResult)) => head + " :: " + tailResult
  case None                     => "nil"
}
```

`String`, параметр типа для `ListF`: тип задачи после ее решения.

Теперь, когда у нас есть универсальный тип `ListF`, мы должны обновить `fold`, чтобы использовать его вместо опционального `head` и `tail`:

```scala
def fold[A](
  op     : ListF[A] => A,
  project: List => ListF[List]
): List => A = {

  def loop(state: List): A =
    op(project(state) match {
      case Some((head, tail)) => Some((head, loop(tail)))
      case None               => None
    })

  loop
}
```

Это первый шаг к тому, чтобы сделать нашу диаграмму немного менее зашумленной:

![ListF of tail](./img/cata-5-hl-1.svg)

## Обобщаем рекурсию

Теперь, когда мы сделали все это, я хотел бы, чтобы вы взглянули на следующую часть диаграммы:

![Does this look familiar?](./img/cata-5-hl-2.svg)

Вам это не кажется знакомым? Вы переходите от `ListF[List]` к `ListF[A]`, применяя массу кода, который по сути сводится к `loop` - функции из `List` в `A`.

Давайте посмотрим, сможем ли мы сделать интуицию более очевидной, взяв сопоставление с шаблоном из `loop` во вспомогательную функцию, которую мы назовем `go`, потому что у меня заканчиваются имена:

```scala
def fold[A](
  op     : ListF[A] => A,
  project: List => ListF[List]
): List => A = {

  def loop(state: List): A =
    op(go(project(state)))

  def go(state: ListF[List]): ListF[A] =
    state match {
      case Some((head, tail)) => Some((head, loop(tail)))
      case None               => None
    }

  loop
}
```

`go` это функция преобразующая `ListF[List]` в `ListF[A]` основном применяя `loop`, функцию преобразующую `List` в `A`.

Но давайте сделаем это еще более очевидным, исключив `loop` из уравнения и сделав его параметром для `go`:

```scala
def fold[A](
  op     : ListF[A] => A,
  project: List => ListF[List]
): List => A = {

  def loop(state: List): A =
    op(go(project(state), loop))

  def go(state: ListF[List], f: List => A): ListF[A] =
    state match {
      case Some((head, tail)) => Some((head, f(tail)))
      case None               => None
    }

  loop
}
```

`go` берёт `ListF[List]`, функцию преобразующую `List` в `A` и возвращает `ListF[A]`.

И это `map`! Это функция, которую вы найдете повсюду. Для `Option[A]` и функции `A => B`, `map` вернёт вам `Option[B]`. Для `Future[A]` и функции `A => B`, `map` вернёт вам `Future[B]`. Это работает для `List`, `Try`... практически для всего.

Это настолько повторяющийся паттерн, что он обычно абстрагируется за чем-то, что называется функтором, что не является самым дружелюбным названием, но на самом деле просто означает _что-то, что имеет разумную реализацию `map`_.

Это важный шаг вперед, потому что, если мы сможем выразить наши требования к `ListF` в терминах функтора, возможно, мы наконец сможем абстрагироваться от структуры `List`!

О, и да, именно отсюда `F` в `ListF`, потому что понятие функтора является важной частью того, что делает `ListF` полезным.

## Функтор

There are many ways we could encode functor - I toyed with using subtyping here, which would work perfectly, but decided that I didn't want to finish ruining whatever reputation I might have, so let's go with the traditional, boring approach: _type classes_.

You don't really need to know about type classes to follow, but you can learn more about them [here](../typeclasses) should you want to.

To declare a `Functor` type class, we merely need a `Functor` trait with an abstract `map` method:

```scala
trait Functor[F[_]] {
  def map[A, B](fa: F[A], f: A => B): F[B]
}
```

Now that the type class is defined, we need to provide an instance of that trait for our specific `ListF` type:

```scala
implicit val listFFunctor = new Functor[ListF] {
  override def map[A, B](list: ListF[A], f: A => B) =
    list match {
      case Some((head, tail)) => Some((head, f(tail)))
      case None               => None
    }
}
```

Note how the body of `map` is exactly the body of `go` in our current `fold` implementation:
- if we have a `head` and a `tail`, apply the specified function to the `tail`.
- otherwise, do nothing.

Next is a little bit of syntactic glue to make the rest of the code easier to read: a `map` function that, given an `F[A]`, an `A => B` and a `Functor[F]`, puts everything together and allows us to ignore tedious implementation details.

```scala
def map[F[_], A, B](
  fa     : F[A],
  f      : A => B
)(implicit
  functor: Functor[F]
): F[B] =
  functor.map(fa, f)
```

Right, with that out of the way, we can now rewrite `fold` to use `map` instead of `go`:

```scala
def fold[A](
  op     : ListF[A] => A,
  project: List => ListF[List]
): List => A = {

  def loop(state: List): A =
    op(map(project(state), loop))

  loop
}
```

Which simplifies our diagram quite a bit:

![Functor included](./img/cata-6-hl-1.svg)

Now, people that are already familiar with functors are probably also aware that the polymorphic list, `List[A]`, has a functor instance: given a `List[A]` and an `A => B`, you can get a `List[B]`.

It's important to realise that the functor instances for `List` and `ListF` are not the same thing. This is often a source of confusion, but it makes sense when you think about it: they do not work on the same things at all. The `A` of `List[A]` and in `ListF[A]` are completely different things.

In `List[A]`, `A` is the type of the values contained by the list. `List[Int]`, for example, represents a list of integers.

In `ListF[A]`, `A` is the type of the value we use to represent the tail of a list. `ListF[List]` is a direct representation of a `List`: a head and a tail. `ListF[Int]` is the representation of a list after we've turned its tail into an int, for example by computing its product.


## Абстрагируясь от `ListF`

We're not quite done yet: `fold` still relies on `ListF`, which is strongly tied to the structure of a list.

![ListF](./img/cata-6-hl-2.svg)

If we look at the code though, the only thing we actually need to know about `ListF` is that we can call `map` on it - that it has a `Functor` instance. This allows us to rewrite `fold` in a way that works for any type constructor `F` that has a `Functor` instance:

```scala
def fold[F[_]: Functor, A](
  op     : F[A] => A,
  project: List => F[List]
): List => A = {

  def loop(state: List): A =
    op(map(project(state), loop))

  loop
}
```

This gives us `ListF`-free implementation:

![Generalised ListF](./img/cata-7-hl-1.svg)

## Абстрагируясь от `List`

Finally, the last step is abstracting over `List`, which is still the input type of our generic `fold`:

![List](./img/cata-7-hl-2.svg)

That turns out to be much simpler than we might have though: we never actually use the fact that we're working with a `List`. All we need to know about that type is that we can provide a valid `project` for it. This allows us to turn `List` into a type parameter:

```scala
def fold[F[_]: Functor, A, B](
  op     : F[A] => A,
  project: B => F[B]
): B => A = {

  def loop(state: B): A =
    op(map(project(state), loop))

  loop
}
```

Which gives us a `List`-free implementation:

![Generalised List](./img/cata-8-hl-1.svg)

## Именование вещей

Now that we have a fully generic implementation that we're happy with, we need to start thinking about names. The generic fold has kind of a scary name: `catamorphism`, often simplified to `cata`:

```scala
def cata[F[_]: Functor, A, B](
  op     : F[A] => A,
  project: B => F[B]
): B => A = {

  def loop(state: B): A =
    op(map(project(state), loop))

  loop
}
```

While the name is intimidating, it's meaning is quite clear once you think about it. `cata` means _I know ancient greek_, `morphism` means _I know category theory_, which gives us `catamorphism` - _I know more than you do_.

And, of course, `op` has a proper, functional name. It's called, like just about everything else in functional programming, an _algebra_ (well, an F-Algebra, to be specific).

```scala
def cata[F[_]: Functor, A, B](
  algebra: F[A] => A,
  project: B => F[B]
): B => A = {

  def loop(state: B): A =
    algebra(map(project(state), loop))

  loop
}
```

`F` also has a more official name: _pattern functor_, or _base functor_, depending on the papers you read.

We've seen that the pattern functor could be thought of as a representation of intermediate steps in a structural recursion (or in any recursive algorithm, really): the decomposition of a problem, before and after solving its sub-problems.

Having named everything, we get this final representation of a catamorphism:

![Catamorphism](./img/cata-9.svg)

## `product` as a cata

Before finishing our study of catamorphisms: we said that they were generalised structural recursion. If they're generalised, surely we should be able to write another structural recursion problem, `product`, as a catamorphism:

```scala
val productAlgebra: ListF[Int] => Int = {
  case Some((head, tailProduct)) => head * tailProduct
  case None                      => 1
}

val product: List => Int =
  cata(productAlgebra, project)
```

And, yes, this yields the expected result:

```scala
product(ints)
// res19: Int = 6
```

## Ключевые выводы

We've seen that catamorphisms are far less complicated than their names make them out to be: a relatively straightforward refactoring away from familiar folds. And they would, in theory, allow us to write structural recursion algorithms on any type that can be projected into a pattern functor.

It's a bit of a shame that the only type we've seen that work on is `List`, then, isn't it?

[Назад](./fold.md) | [Оглавление](./README.md) | [Дальше](./tree_height.md)

This work is licensed under a <a rel="license" href="https://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 4.0 International License</a>.