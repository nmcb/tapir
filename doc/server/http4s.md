# Running as an http4s server

To expose an endpoint as an [http4s](https://http4s.org) server, first add the following 
dependency:

```scala
"com.softwaremill.sttp.tapir" %% "tapir-http4s-server" % "@VERSION@"
```

and import the object:

```scala mdoc:compile-only
import sttp.tapir.server.http4s.Http4sServerInterpreter
```

The `toRoutes` and `toHttp` methods require a single, or a list of `ServerEndpoint`s, which can be created by adding
[server logic](logic.md) to an endpoint.

The server logic should use a cats-effect-support `F[_]` effect type. For example:

```scala mdoc:compile-only
import sttp.tapir._
import sttp.tapir.server.http4s.Http4sServerInterpreter
import cats.effect.IO
import org.http4s.HttpRoutes

def countCharacters(s: String): IO[Either[Unit, Int]] = 
  IO.pure(Right[Unit, Int](s.length))

val countCharactersEndpoint: PublicEndpoint[String, Unit, Int, Any] = 
  endpoint.in(stringBody).out(plainBody[Int])
val countCharactersRoutes: HttpRoutes[IO] = 
  Http4sServerInterpreter[IO]().toRoutes(countCharactersEndpoint.serverLogic(countCharacters _))
```

The created `HttpRoutes` are the usual http4s `Kleisli`-based transformation of a `Request` to a `Response`, and can 
be further composed using http4s middlewares or request-transforming functions. The tapir-generated `HttpRoutes`
captures from the request only what is described by the endpoint.

It's completely feasible that some part of the input is read using a http4s wrapper function, which is then composed
with the tapir endpoint descriptions. Moreover, "edge-case endpoints", which require some special logic not expressible 
using tapir, can be always implemented directly using http4s.

## Streaming

The http4s interpreter accepts streaming bodies of type `Stream[F, Byte]`, as described by the `Fs2Streams`
capability. Both response bodies and request bodies can be streamed. Usage: `streamBody(Fs2Streams[F])(schema, format)`.

The capability can be added to the classpath independently of the interpreter through the 
`"com.softwaremill.sttp.shared" %% "http4s"` dependency.

## Web sockets

The interpreter supports web sockets, with pipes of type `Pipe[F, REQ, RESP]`. See [web sockets](../endpoint/websockets.md) 
for more details.

However, endpoints which use web sockets need to be interpreted using the `Http4sServerInterpreter.toWebSocketRoutes`
method, which returns a function `WebSocketBuilder2[F] => HttpRoutes[F]`. This can then be added to a server builder
using `withHttpWebSocketApp`, for example:

```scala mdoc:compile-only
import sttp.capabilities.WebSockets
import sttp.capabilities.fs2.Fs2Streams
import sttp.tapir._
import sttp.tapir.server.http4s.Http4sServerInterpreter
import cats.effect.IO
import org.http4s.HttpRoutes
import org.http4s.blaze.server.BlazeServerBuilder
import org.http4s.server.Router
import org.http4s.server.websocket.WebSocketBuilder2
import fs2._
import scala.concurrent.ExecutionContext

implicit val ec: ExecutionContext = scala.concurrent.ExecutionContext.Implicits.global

val wsEndpoint: PublicEndpoint[Unit, Unit, Pipe[IO, String, String], Fs2Streams[IO] with WebSockets] =
  endpoint.get.in("count").out(webSocketBody[String, CodecFormat.TextPlain, String, CodecFormat.TextPlain](Fs2Streams[IO]))
    
val wsRoutes: WebSocketBuilder2[IO] => HttpRoutes[IO] =
  Http4sServerInterpreter[IO]().toWebSocketRoutes(wsEndpoint.serverLogicSuccess[IO](_ => ???))
    
BlazeServerBuilder[IO]
  .withExecutionContext(ec)
  .bindHttp(8080, "localhost")
  .withHttpWebSocketApp(wsb => Router("/" -> wsRoutes(wsb)).orNotFound)        
```

## Server Sent Events

The interpreter supports [SSE (Server Sent Events)](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events).

For example, to define an endpoint that returns event stream:

```scala mdoc:compile-only
import cats.effect.IO
import sttp.model.sse.ServerSentEvent
import sttp.tapir._
import sttp.tapir.server.http4s.{Http4sServerInterpreter, serverSentEventsBody}

val sseEndpoint = endpoint.get.out(serverSentEventsBody[IO])

val routes = Http4sServerInterpreter[IO]().toRoutes(sseEndpoint.serverLogicSuccess[IO](_ =>
  IO(fs2.Stream(ServerSentEvent(Some("data"), None, None, None)))
))
```

## Configuration

The interpreter can be configured by providing an `Http4sServerOptions` value, see
[server options](options.md) for details.

The http4s options also includes configuration for the blocking execution context to use, and the io chunk size.

