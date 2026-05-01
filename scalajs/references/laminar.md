# Laminar Reference

Laminar is the canonical Scala.js UI library. It uses **Functional Reactive Programming (FRP)** with direct DOM updates — no virtual DOM diffing.

## Setup

```scala
// build.sbt
libraryDependencies += "com.raquo" %%% "laminar" % "17.0.0"
```

## Bootstrap

```scala
import com.raquo.laminar.api.L.{*, given}
import org.scalajs.dom

@main def main(): Unit =
  renderOnDomContentLoaded(
    dom.document.getElementById("app"),
    appElement()
  )
```

## Core Primitives

### `Var[A]` — mutable reactive state
```scala
val counter: Var[Int] = Var(0)
counter.set(5)
counter.update(_ + 1)
val doubled: Signal[Int] = counter.signal.map(_ * 2)
```

### `Signal[A]` — read-only time-varying value
```scala
val nameSignal: Signal[String] = Var("Alice").signal
val greeting = nameSignal.map(n => s"Hello, $n!")
// Combine two signals:
val combined = nameSignal.combineWith(counter.signal)
  .map { (name, count) => s"$name has clicked $count times" }
```

### `EventStream[A]` — discrete events
```scala
val clicks: EventStream[dom.MouseEvent] = someButton.events(onClick)
val mapped: EventStream[String] = clicks.mapTo("clicked!")
```

### `Observer[A]` — event sink
```scala
val logger: Observer[String] = Observer(s => println(s))
clicks.mapTo("click") --> logger  // pipe events to observer
```

## Building Elements

```scala
// Static element
div(
  cls := "container",
  h1("Hello Laminar"),
  p("A paragraph"),
)

// Dynamic text content
p(child.text <-- counter.signal.map(n => s"Count: $n"))

// Dynamic child element
div(child <-- boolSignal.map(if _ then span("Yes") else span("No")))

// Dynamic list of children
ul(
  children <-- itemsSignal.map(_.map(item => li(item.name)))
)

// Efficient keyed children (reuses DOM nodes by key)
ul(
  children <-- itemsSignal.split(_.id) { (id, initial, itemSignal) =>
    li(child.text <-- itemSignal.map(_.name))
  }
)
```

## Event Handling

```scala
button(
  "Click me",
  onClick --> { event => counter.update(_ + 1) },
)

// Map event before handling:
input(
  onInput.mapToValue --> nameVar,  // sets nameVar to input's current value
)

// Filter events:
keyboardEvents(onKeyDown).filter(_.key == "Enter") --> handleEnter
```

## Common Bindings

| Binding | Meaning |
|---------|---------|
| `cls := "foo"` | Static CSS class |
| `cls <-- signal` | Dynamic CSS class |
| `cls.toggle("active") <-- boolSignal` | Conditional class |
| `value <-- signal` | Input value (one-way) |
| `controlled(value <-- sig, onInput.mapToValue --> obs)` | Two-way controlled input |
| `href := "..."` | Static attribute |
| `href <-- signal` | Dynamic attribute |
| `styleAttr := "color:red"` | Inline style (string) |
| `display.none.when(hiddenSignal)` | Conditional display |
| `idAttr := "my-id"` | Set element ID |
| `tpe := "button"` | Set input type |

## Component Pattern

Laminar components are plain methods returning `Element`:

```scala
def counterButton(label: String): Element =
  val count = Var(0)
  button(
    label,
    " — count: ",
    child.text <-- count,
    onClick --> { _ => count.update(_ + 1) },
  )

// Usage:
div(
  counterButton("A"),
  counterButton("B"),
)
```

## Data Flow Model

```
Var <-> Signal                 EventStream
  |        |                       |
  |    .map, .flatMap           .map, .filter
  |        |                       |
  +-- <--  DOM element  --> +-- -->  Observer
```

- Left arrow `<--` : data flows **into** DOM (from signal/stream to DOM attribute/child)
- Right arrow `-->` : data flows **from** DOM event to observer/updater

## Routing (with `waypoint` library)

```scala
// build.sbt
libraryDependencies += "com.raquo" %%% "waypoint" % "8.0.0"
```

```scala
import com.raquo.waypoint._

sealed trait Page
case object HomePage extends Page
case class UserPage(id: Int) extends Page

val router = new Router[Page](
  routes = List(
    Route.static(HomePage, root / endOfSegments),
    Route[UserPage, Int](_.id, UserPage(_), root / "user" / segment[Int] / endOfSegments),
  ),
  getPageTitle = {
    case HomePage      => "Home"
    case UserPage(id)  => s"User $id"
  },
  serializePage = { case p => p.toString },
  deserializePage = _ => HomePage,
)

// Render based on current page:
div(
  child <-- router.currentPageSignal.map {
    case HomePage     => renderHome()
    case UserPage(id) => renderUser(id)
  }
)
```

## HTTP Requests (fetch API)

```scala
import org.scalajs.dom
import scala.concurrent.Future
import scala.scalajs.concurrent.JSExecutionContext.Implicits.queue

def fetchJson(url: String): EventStream[String] =
  EventStream.fromFuture(
    dom.fetch(url)
      .toFuture
      .flatMap(_.text().toFuture)
  )

// In component:
val dataStream = fetchJson("/api/data")
div(
  child <-- dataStream.map(json => pre(json))
)
```

## Pitfalls

- Do NOT call `Signal.now()` in production UI code — it's for tests only. Use `<--` bindings.
- Always own binders inside a Laminar element so they're released on unmount.
- `split` is critical for lists that mutate — without it, every item re-renders fully on any list change.
- `controlled(...)` prevents uncontrolled text input "jumping" when programmatically updating value.
