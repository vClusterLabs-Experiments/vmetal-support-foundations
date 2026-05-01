# Learner Workspace Template

Copy this directory to start your own work. The template is read-only reference; your edits go in the copy.

## How to use

```bash
cp -r lab/vmetal/_template lab/vmetal/work-$(whoami)
cd lab/vmetal/work-$(whoami)
```

Then work milestone-by-milestone. Each `milestone-NN/` directory contains a pre-seeded `notes.md` with the conceptual questions from the brief and a checklist of artifacts to produce.

## Structure

```text
_template/
├── README.md                       (this file)
├── milestone-01/
│   └── notes.md                    (questions + checklist)
├── milestone-02/
│   └── notes.md
├── milestone-03/
│   └── notes.md
├── milestone-03-5/                 (required for M5)
│   └── notes.md
├── milestone-04/
│   └── notes.md
├── milestone-04-5/                 (required for M5)
│   └── notes.md
├── milestone-05/
│   └── notes.md
├── milestone-06/
│   └── notes.md
└── milestone-07/                   (optional)
    └── notes.md
```

Each milestone's `notes.md` is the durable artifact. Other files (configs, scripts, traces) are produced as you go and live alongside it.

## Convention

- Treat `scaffolding/` subdirectories (libvirt XML, dnsmasq.conf, etc.) as throwaway, they document reproducibility but aren't the thing you're studying.
- The conceptual question answers in `notes.md` are what transfer to real hardware.
- Don't skip the "induce a failure" success criteria. They're the most valuable exercise in each milestone.

## Working alongside the brief

For each milestone, keep two files open:

1. `lab/vmetal/milestone-NN-*.md`, the brief (read-only reference)
2. `lab/vmetal/work-<you>/milestone-NN/notes.md`, your answers (active work)

---

## License

This `_template/` directory is licensed under the [MIT License](./LICENSE). Notes and artifacts you produce by copying it into your own workspace are yours, unencumbered by the curriculum's non-commercial restriction.

---

Last Updated: 2026-05-01