# Edit Video as Code Demos

Open [`index.html`](./index.html) in a browser. The page contains two standalone,
dependency-free conceptual demos:

1. an Agent workflow from project discovery to a semantic Timeline update;
2. an A/B/C comparison of early, action-point, and late cuts.

These demos explain principles. They are not StarCut product screenshots,
performance measurements, or substitutes for a real product recording.

## Planned product recordings

The article should later include three short recordings from a real StarCut
project:

1. `agent-workflow`: one request, visible `glob`/`grep`/`read`/`edit` trace, and
   the same Timeline updating in the Editor;
2. `streaming-edit`: a RESTful-like edit appearing progressively while the model
   is still producing the tool call;
3. `human-handoff`: the Agent changes a Clip, then a human immediately adjusts
   and undoes the same node in the Timeline UI.

Each recording should ship with an MP4 source, a compact preview, and a static
poster under `papers/assets/edit-video-as-code/`.
