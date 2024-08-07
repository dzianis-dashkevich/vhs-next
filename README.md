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

The current playback engine (VHS and it's ecosystem: m3u8-parser, mpd-parser, mux.js, contrib-eme, etc...) has reached a critical point where it's issues significantly impede both maintenance and innovation.

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

The vhs-next and future videojs 9 related repositories should live in one `@videojs` scoped monorepo.

We already have `@videojs/*` scoped monorepo under the `web-media-box` repository:
https://github.com/videojs/web-media-box

> ⚠️ **Note**
>
> `web-media-box` or simply `wmb` is just a fancy name for the repository.
>
> The actual packages are `@videojs/*` scoped.
>
> We can rename it to any other name (eg: `videojs-monorepo`) or keep existing name as a legacy to the 13th Hack-Week.

Here is monorepo structure example:

```
├── ...
├── packages
│   ├── ...
│   ├── transport-stream-parser (@videojs/transport-stream-parser)
│   ├── iso-bmff-parser         (@videojs/iso-bmff-parser)
│   ├── dash-parser             (@videojs/dash-parser)
│   ├── hls-parser              (@videojs/hls-parser)
│   ├── playback                (@videojs/playback)
│   ├── ui                      (@videojs/ui)
│   └── videojs.dev             (app, private: true)
├── package.json                (workspaces: [packages/*], private: true)
├── nx.json                     (shared NX configuration)
└── tsconfig.json               (shared TS configuration)

```

As you can see from this structure, videojs scoped monorepo will have 2 main repositories:
`@videojs/playback` and `@videojs/ui` and a bunch of additional libraries that can be used as standalone libs + videojs related apps.

`@videojs/playback` is a standalone player that handles playback and does not support UI. (Similar to hls.js and dash.js).

`@videojs/ui` is a standalone UI player that uses `@videojs/playback` under the hood and provides a lot of UI WebComponents to customize your player. (Something similar to https://www.vidstack.io/)

This way we cover both groups of developers:
- Developers who want to build their own player's UI on top of `@videojs/playback`
- Developers who want embedded player experience

This documents covers only `@videojs/playback` and other playback-related utility packages.

The following existing public packages should be deprecated and archived after `@videojs/playback` is fully developed:
- https://github.com/videojs/http-streaming
- https://github.com/videojs/videojs-contrib-eme
- https://github.com/videojs/vhs-utils
- https://github.com/videojs/m3u8-parser
- https://github.com/videojs/mux.js
- https://github.com/videojs/videojs-contrib-quality-levels
- https://github.com/videojs/mpd-parser
- https://github.com/videojs/aes-decrypter

# Functional Requirements


| Feature                         | Spec                                                                                                                                                                                                                  | Shaka                                                                                                                | hls.js                                                                                        | dash.js                                                                                                           | Notes                                                                                                                   | Priority |
|---------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|----------|
| HLS MPEG-2 Transport Stream     | [rfc-8216:section-3.1.1](https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-3.1.1)                                                                                                             | Shaka NO longer uses mux.js: https://github.com/shaka-project/shaka-player/blob/main/lib/transmuxer/ts_transmuxer.js | https://github.com/video-dev/hls.js/blob/master/src/demux/tsdemuxer.ts                        | N/A                                                                                                               | While community moving toward hls CMAF, mpeg2-ts is still massively presented, so we have to support it.                | MUST     |
| Common-Media-Client-Data (CMCD) | [cta-5004](https://cdn.cta.tech/cta/media/media/resources/standards/pdfs/cta-5004-final.pdf)                                                                                                                          | https://github.com/shaka-project/shaka-player/blob/main/lib/util/cmcd_manager.js                                     | https://github.com/video-dev/hls.js/blob/master/src/controller/cmcd-controller.ts             | https://github.com/Dash-Industry-Forum/dash.js/blob/development/src/streaming/models/CmcdModel.js                 | All major open source players support this feature. We may consider re-using `@svta/common-media-library/cmsd` utils.   | SHOULD   |
| Common-Media-Server-Data (CMSD) | [cta-5006](https://cdn.cta.tech/cta/media/media/resources/standards/pdfs/cta-5006-final.pdf)                                                                                                                          | https://github.com/shaka-project/shaka-player/blob/main/lib/util/cmsd_manager.js                                     | Not Implemented                                                                               | https://github.com/Dash-Industry-Forum/dash.js/blob/development/src/streaming/models/CmsdModel.js                 | Limited support in shaka and dash.js. Mainly for abr. We may consider re-using `@svta/common-media-library/cmsd` utils. | SHOULD   |
| Content Steering                | [hls-content-steering](https://developer.apple.com/streaming/HLSContentSteeringSpecification.pdf) <br/> <br/> [dash-content-steering](https://dashif.org/docs/DASH-IF-CTS-00XX-Content-Steering-Community-Review.pdf) | https://github.com/shaka-project/shaka-player/blob/main/lib/util/content_steering_manager.js                         | https://github.com/video-dev/hls.js/blob/master/src/controller/content-steering-controller.ts | https://github.com/Dash-Industry-Forum/dash.js/blob/development/src/dash/controllers/ContentSteeringController.js | All major open source players support this feature.                                                                     | SHOULD   |
| DASH in-manifest thumbnails     | 6.2.6 Section [dash-if Guidelines](https://dashif.org/docs/DASH-IF-IOP-v4.3.pdf)                                                                                                                                      | https://github.com/shaka-project/shaka-player/blob/main/lib/dash/dash_parser.js#L2224-L2239                          | N/A                                                                                           | https://github.com/Dash-Industry-Forum/dash.js/blob/development/src/streaming/thumbnail/ThumbnailTracks.js        | All major dash open source players support this feature.                                                                | COULD    |
| HLS in-manifest thumbnails      | [image-media-playlist spec](https://github.com/image-media-playlist/spec/blob/master/image_media_playlist_v0_4.pdf)                                                                                                   | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_parser.js#L933                                   | Not Implemented                                                                               | N/A                                                                                                               | It is unofficial spec, but nice to have.                                                                                | COULD    |


# Non-Functional Requirements

TBD: Describe non-functional Requirements, eg: build your own player, pipeline, size, etc...

# Entities

TBD: Describe entities and their relations

# Scenarios

TBD: Describe Scenarios with flow charts and comments (where possible use connections to FRs and NFRs)
