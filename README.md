# Tennis algorithm lab

A design and specification project for improving tennis activity detection through
reviewed feedback, reusable signals and reproducible evaluation. Documentation only;
initial delivery uses rules and evaluation. Model training is deferred.

Read in order:

1. [System design](docs/SYSTEM-DESIGN.md): architecture, ownership, contracts,
   identities, storage and retention.
2. [Algorithm specification](docs/ALGORITHM-SPEC.md): labels, signals, candidate
   models, formulas and evaluation process.
3. [Milestones](docs/MILESTONES.md): dependencies, owners and acceptance criteria.

[Roadmap](ROADMAP.txt) is the short entry point. The website lives in
[tennis-web](https://github.com/haocs/tennis-web), execution in
[tennis-processing](https://github.com/haocs/tennis-processing), and orchestration
in [tennis-processing-server](https://github.com/haocs/tennis-processing-server).

Keep recordings, precise locations, private configuration, and feedback datasets
outside Git. Any future shareable fixtures should be synthetic or explicitly
approved for redistribution.
