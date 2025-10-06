# Tornado, Bokeh, Panel, and HoloViz: Interaction Map

This comparison is grounded in the Tornado 6.5.2 documentation, the Bokeh 3.8 server reference (including `bokeh.server.tornado`), Panel 1.8 guidance on serving and FastAPI integration, the HoloViz project overview, and the primary docs for Uvicorn and FastAPI. Together they show how these pieces form a visualization stack and how that stack relates to the broader ASGI universe.

## Stack Overview

Tornado provides the asynchronous HTTP/WebSocket runtime. Bokeh Server is implemented as a Tornado application that keeps Python "documents" synchronized with BokehJS in the browser. Panel wraps Bokeh's model system to build full dashboards and serves them through the same Bokeh/Tornado machinery. HoloViz sits above as the ecosystem that supplies plotting, datashading, and declarative APIs that Panel can embed.

**Runtime spine**

```
Tornado IOLoop → Bokeh Server (websocket sync) → Panel layouts & callbacks → Browser (BokehJS)
                                           ↑
                                 HoloViz plotting APIs feed Panel
```

The same applications can call into other HoloViz tools (hvPlot, HoloViews, Datashader) so that Python code remains the source of truth while Tornado keeps connections alive without WSGI constraints, as highlighted in the Tornado user guide.

## Component Roles

### Tornado: asynchronous server and networking core

The Tornado docs describe it as a non-blocking web framework and networking library intended for long-lived connections such as WebSockets and long polling. It runs directly on its event loop (now interoperable with `asyncio`) and is not centered on WSGI, which is why it can scale a single-threaded process to many open sockets.

### Bokeh Server: model synchronization on Tornado

According to the Bokeh server reference, the server's purpose is to keep Python-side Bokeh models synchronized with BokehJS so that UI events can trigger Python callbacks and server-side updates can be pushed over the websocket. The `BokehTornado` application exposes hooks like `extra_websocket_origins` and `extra_patterns` so developers can add custom endpoints or allow embedded hosts.

### Panel: high-level app layer on Bokeh Server

Panel's serving docs confirm that a Panel server is just the Bokeh server running on Tornado. Panel composes widgets, layouts, and visualizations (including from libraries beyond Bokeh) and relies on the same session-per-connection semantics: each browser session executes the Panel app code to build a fresh Bokeh document that stays synced as users interact.

### HoloViz: ecosystem umbrella

The HoloViz overview positions Panel as the dashboard/app entry point within a broader suite that includes HoloViews, hvPlot, Datashader, GeoViews, Param, and more. These libraries generate Bokeh (or other backend) objects that Panel can render, letting data workflows move up and down the stack without leaving Python.

## Interaction Model

When a client connects, the Bokeh server (running as a Tornado app) creates a new session document, executes the application code, and keeps the Python models in lockstep with the browser via the websocket protocol. Panel builds on this by defining reactive expressions, `pn.bind`, and other APIs that mutate those models safely on the server thread.

```
Browser (BokehJS) ⇄ WebSocket ⇄ Bokeh Server on Tornado ⇄ Python callbacks & data sources
                                             ↘
                                              Panel objects, HoloViz plots, external libraries
```

The server maintains keep-alive pings (`keep_alive_milliseconds`) to prevent idle disconnects and exposes scheduling utilities so long-running or background work can be coordinated with the Tornado IOLoop.

## Comparing With the ASGI Web Stack

Uvicorn is documented as an ASGI server that speaks HTTP/1.1 and WebSockets and runs ASGI apps like FastAPI. FastAPI itself is a high-performance web framework built on Starlette and Pydantic, typically deployed via Uvicorn (optionally under Gunicorn).

While both stacks deliver async IO and websockets, their layers target different problems:

| Concern | Tornado / Bokeh / Panel stack | Uvicorn / FastAPI stack |
| --- | --- | --- |
| Server interface | Tornado's own HTTP/WebSocket runtime (non-WSGI) | ASGI 3.0 servers (Uvicorn, Hypercorn, Daphne) |
| Primary role | Interactive, Python-in-the-loop visualization apps with model syncing | General-purpose APIs, microservices, real-time backends |
| Default communication | Persistent websocket sessions driven by Bokeh documents | Request/response APIs, optional WebSocket endpoints |
| App framework | Panel (or raw Bokeh/Tornado handlers) | FastAPI (or Starlette/other ASGI frameworks) |
| Extensibility | `extra_patterns`, session hooks, Panel reactivity | ASGI middleware, dependency injection, OpenAPI tooling |

In practice, teams often pair them: expose REST or GraphQL endpoints through FastAPI/Uvicorn and deliver exploratory dashboards through Panel/Bokeh, either side by side or embedded.

## Integration Patterns

### Reverse proxy separation

Panel's deployment guides recommend running the visualization server on its own port and using a reverse proxy (e.g., Nginx or Traefik) to route different paths:

```
Browser
  ↳ /api  → Uvicorn + FastAPI (ASGI REST/WebSocket services)
  ↳ /app  → Tornado + Bokeh Server (Panel sessions over websocket)
```

This keeps lifecycles isolated and lets each server scale independently while honoring websocket requirements (`extra_websocket_origins` should include the proxy host).

### Native FastAPI hosting

Panel 1.5+ introduced the `panel.io.fastapi.add_application` decorator, which registers Panel apps directly on a FastAPI instance. This lets Uvicorn serve both REST endpoints and Panel routes from the same process, useful when the deployment pipeline or platform expects a single ASGI entrypoint. Under the hood, Panel still spins up Bokeh documents and manages Tornado callbacks, but Panel bridges the event loops so developers work with one FastAPI app.

### Tornado-centric composites

For cases needing custom Tornado handlers (e.g., bespoke WebSocket streams, long-running background jobs), the Bokeh server's `extra_patterns` option can mount additional HTTP or websocket handlers alongside Bokeh apps. This keeps everything within Tornado while still allowing Panel or raw Bokeh apps to share the loop.

## When to Choose Each Layer

- Reach for Panel (and by extension Bokeh/Tornado) when you need server-driven dashboards, streaming plots, or notebook-to-web promotion with minimal JavaScript. Reactive expressions and websocket syncing keep Python code directly manipulating UI state.
- Use Bokeh Server alone when you want full control over document construction or prefer Bokeh's lower-level model APIs without Panel abstractions.
- Opt for pure Tornado if you are building custom async services with bespoke protocol handling or want to integrate non-visual websockets tightly with visualization sessions.
- Choose FastAPI + Uvicorn for REST/JSON APIs, authentication flows, and service-oriented backends; front the Panel server or embed it if you also need rich visual UIs.

## Operational Considerations

- **WebSocket origins and routing:** Configure `extra_websocket_origins` (BokehTornado) or Panel's server settings so browsers connecting through proxies are accepted. Reverse proxies must forward websocket upgrades intact.
- **Session isolation:** Each connection creates a new Bokeh `Document`; shared state should live in external stores or carefully managed module-level objects.
- **Threading and async bridges:** Tornado code is not thread-safe by default; use IOLoop callbacks or Panel's scheduling utilities to hand work back to the main thread. When combining with FastAPI, ensure blocking tasks run in executors to avoid starving the event loop.
- **Keep-alives and payload size:** Tune `keep_alive_milliseconds` and `websocket_max_message_size` for streaming dashboards; Panel's docs highlight these knobs when deploying behind managed platforms.
- **Testing and tooling:** FastAPI inherits ASGI testing via HTTPX; Panel/Bokeh apps can be smoke-tested with bokeh's integration patterns or selenium-based UI checks. Plan for both when integrating the stacks.

## Summary

Tornado anchors the visualization stack with a purpose-built async server. Bokeh Server layers model synchronization on top of it, Panel elevates the experience to full dashboards, and HoloViz supplies plotting ecosystems that feed those dashboards. Uvicorn and FastAPI occupy the ASGI world, offering complementary infrastructure for APIs and services. With the documented integration hooks—`add_application` in Panel and `extra_patterns`/`extra_websocket_origins` in BokehTornado—you can run the stacks side by side or weave them together, picking the right layer for visualization, APIs, or both.