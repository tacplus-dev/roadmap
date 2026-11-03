# Trace propagation: the field carries it; user code copies it; tacplus covers projections; never stored (Jody, 2026-11-03)

Jody, in the trace huddle:
- "The important thing is that generated code does not know when 2 things are related. eg. a user call comes into the user
  code which makes a call on a S2S channel. They can mux any number of users over that channel."
- "I am unsure why you think we need a Trace function. We should put the trace_id field on all messages. So if a request
  comes in with a message and it has trace_id set and we split it each new message gets the same trace_id, etc. this way we
  can trace thru the entire system. we should clear the trace_id field when a type is stored in a database."
- "we also have a feature already for splitting messages called projections. the trace_id must be handled in the tacplus
  generated code for this."

## Decided
- **`trace_id` (field 200) is on every message,** with no exceptions: requests, responses, events, records, internal,
  state and redo types.
- **No ambient propagation.** Generated code doesn't guess causality: no context, thread-local or task-local trace. A shared
  S2S channel multiplexes many users' work.
- **User code propagates by copying the field.** When a request causes outgoing messages, each one gets the inbound
  `trace_id`. Every generated helper or convenience method that builds a request **accepts `trace_id`**, so user code can
  always pass it.
- **Projections are tacplus's job.** Generated projection outputs (`(spec.record)` / `(spec.event)` projections) and
  `(spec.transformation)` conversions copy the source message's `trace_id` onto each output.
- **`trace_id` is cleared whenever a type is stored.** Every generated storage path (state clients, record stores, the
  record plane, redo, kernel state) clears field 200 before persisting. Traces are never recorded.
- **tacplus#265 is split:**
  - its "propagation is generated" point is dropped;
  - its phase 2 (StartTrace, owner-bound traces, the typed event stream, WatchTrace, the CLI) is kept for a later huddle.
