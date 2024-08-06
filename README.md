# Table of Content

* [Summary](#summary)
* [Motivation](#motivation)
* [Monorepo Structure](#monorepo-structure)
* [Functional Requirements](#functional-requirements)
* [Non-Functional Requirements](#non-functional-requirements)
* [Entities](#entities)
* [Scenarios](#scenarios)


# Summary

This documents describes a high level solution design for the VHS-Next (videojs playback engine) project.

This project is as a pre-requisite for a massive videojs 9 update.

# Motivation

The current playback engine (VHS and it's ecosystem: m3u8-parser, mpd-parser, mux.js, contrib-eme) has reached a critical point where it's issues significantly impede both maintenance and innovation.

These challenges have escalated the effort required to support and enhance the existing codebase.

Several key factors underscore the necessity for a new playback engine:

**Maintenance Complexity**: 

Over time, its core architecture has revealed several fundamental limitations, which have manifested as frequent bugs and performance inefficiencies. 

Fixing one issue often leads to the emergence of new problems, creating a cycle of reactive maintenance that is unsustainable.

**Troubleshooting Complexity**: 

Unpredictable data flow made it increasingly difficult to troubleshoot playback engine.

**Performance**: 

The current engine struggles to meet the performance demands of modern video applications, especially in the complex scenarios.

**API limitations**:

- The current playback engine is highly coupled with videojs itself and can't be used as a standalone player in case users what only playback engine abstraction without UI.
- The current playback engine does not provide a robust events api to monitor and get a clear picture of the data flow.
- The current playback engine does not provide a robust errors api, and it is very hard to diagnose and narrow issues.
- The current playback engine provides very limited interceptors api
- The current playback engine provides very limited configuration api

To address these (and many other) issues comprehensively, we propose the introduction of a new playback engine designed with modern principles and technologies.

This new engine will not only resolve the current deficiencies but also provide a scalable foundation for future enhancements.


# Monorepo Structure

The vhs-next and future videojs 9 related repositories should be encapsulated in one `@videojs` scoped monorepo.

We already have @videojs scoped monorepo under the "web-media-box" repository:
https://github.com/videojs/web-media-box

> Note:
> 
> "web-media-box" or simply "wmb" is just a fancy name for the repository. 
> 
> The actual packages are @videojs scoped.
> 
> We can rename it to the "videojs-monorepo" or keep existing name as a legacy to the 13th Hack-Week.

Here is monorepo structure example:

```shell
├── ...
├── packages
│   ├── ...
│   ├── dash-parser (@videojs/dash-parser)
│   ├── hls-parser  (@videojs/hls-parser)
│   ├── playback    (@videojs/playback)
│   ├── ui          (@videojs/ui)
│   └── videojs.dev (app)
├── package.json
├── nx.json
└── tsconfig.json



```

 TBD: add a note about focused scope and which repos we are planning to deprecate/archive and why.


# Functional Requirements

TBD: Describe Functional Requirements with links, samples from different open source players, etc...

# Non-Functional Requirements

TBD: Describe non-functional Requirements, eg: build your own player, pipeline, size, etc...

# Entities

TBD: Describe entities and their relations

# Scenarios

TBD: Describe Scenarios with flow charts and comments (where possible use connections to FRs and NFRs)
