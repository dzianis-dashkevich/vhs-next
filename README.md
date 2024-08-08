# Table of Content

* [Summary](#summary)
* [Motivation](#motivation)
* [Monorepo Structure](#monorepo-structure)
* [Functional Requirements](#functional-requirements)
  * [HLS Features](#hls-features)
    * [HLS MPEG-2 Transport Stream](#hls-mpeg-2-transport-stream)
    * [HLS Common Media Application Format (CMAF)](#hls-common-media-application-format-cmaf)
    * [HLS WebVTT](#hls-webvtt)
    * [HLS IMSC Subtitles](#hls-imsc-subtitles)
    * [HLS Interstitial](#hls-interstitial)
    * [HLS Trick Play](#hls-trick-play)
    * [HLS In-manifest thumbnails (ext-x-iframes-only)](#hls-in-manifest-thumbnails-ext-x-iframes-only)
    * [HLS In-manifest thumbnails (Roku spec)](#hls-in-manifest-thumbnails-roku-spec)
  * [Dash Features](#dash-features)
    * [DASH In-manifest thumbnails](#dash-in-manifest-thumbnails)
  * [Server-Client Signaling](#server-client-signaling)
    * [Content Steering](#content-steering)
    * [Common-Media-Client-Data (CMCD)](#common-media-client-data-cmcd)
    * [Common-Media-Server-Data (CMSD)](#common-media-server-data-cmsd)
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

## HLS Features

### HLS MPEG-2 Transport Stream

> ℹ️ **Priority: MUST** (*Currently supported by VHS*)
> 
> While the community is moving toward HLS CMAF, mpeg2-ts is still massively presented, so we have to support it.

|        | URLS                                                                                    |
|--------|-----------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-3.1.1         |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/transmuxer/ts_transmuxer.js |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/demux/tsdemuxer.ts                  |

### HLS Common Media Application Format (CMAF)

> ℹ️ **Priority: MUST** (*Currently supported by VHS*)

|        | URLS                                                                                |
|--------|-------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-3.1.2     |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_parser.js#L3431 |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/loader/m3u8-parser.ts#L540      |

### HLS WebVTT

> ℹ️ **Priority: MUST** (*Currently supported by VHS*)

> ⚠️ **Note**
>
> HLS WeVTT is not encapsulated in fmp4 and timestamp sync is implemented via X-TIMESTAMP-MAP

|        | URLS                                                                                |
|--------|-------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-3.1.4     |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/text/vtt_text_parser.js |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/utils/webvtt-parser.ts          |

### HLS IMSC Subtitles

> ℹ️ **Priority: MUST** (*Currently NOT supported by VHS*)

> ⚠️ **Note**
>
>
> HLS IMSC is encapsulated in fmp4.

|        | URLS                                                                                |
|--------|-------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-3.1.5     |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/text/mp4_ttml_parser.js |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/utils/imsc1-ttml-parser.ts      |

### HLS Interstitial

> ℹ️ **Priority: COULD** (*Currently NOT supported by VHS*)
>
> Content producers can insert separate interstitial content into their
> primary presentations in order to display advertising, branding, or
> other information to viewers. It is implemented via EXT-X-DATERANGE
> 
> Shaka has limited support

|        | URLS                                                                         |
|--------|------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#appendix-D |
| Shaka  | Check this PR: https://github.com/shaka-project/shaka-player/pull/6761       |
| hls.js | Not Implemented                                                              |

### HLS Trick Play

> ℹ️ **Priority: COULD** (*Currently NOT supported by VHS*)
>
> Trick play could be implemented via Ext-x-iframes-only playlists

|        | URLS                                                                              |
|--------|-----------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-4.4.3.6 |
| Shaka  | Not Implemented                                                                   |
| hls.js | Not Implemented                                                                   |


### HLS In-manifest thumbnails (ext-x-iframes-only)

> ℹ️ **Priority: COULD** (*Currently NOT supported by VHS*)
>
> Official In-manifest support via Ext-x-iframes-only playlists with mjpg codec

|        | URLS                                                                               |
|--------|------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-4.4.3.6  |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_parser.js#L933 |
| hls.js | Not Implemented                                                                    |

### HLS In-manifest thumbnails (Roku spec)

> ℹ️ **Priority: COULD** (*Currently NOT supported by VHS*)
> 
> It is unofficial spec, but nice to have.

|        | URLS                                                                                   |
|--------|----------------------------------------------------------------------------------------|
| Spec   | https://github.com/image-media-playlist/spec/blob/master/image_media_playlist_v0_4.pdf |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_parser.js#L933     |
| hls.js | Not Implemented                                                                        |

## Dash Features

### DASH In-manifest thumbnails

> ℹ️ **Priority: COULD** (*Currently NOT supported by VHS*)

|         | URLS                                                                                                       |
|---------|------------------------------------------------------------------------------------------------------------|
| Spec    | https://dashif.org/docs/DASH-IF-IOP-v4.3.pdf section 6.2.6                                                 |
| Shaka   | https://github.com/shaka-project/shaka-player/blob/main/lib/dash/dash_parser.js#L2224-L2239                |
| dash.js | https://github.com/Dash-Industry-Forum/dash.js/blob/development/src/streaming/thumbnail/ThumbnailTracks.js |

## Server-Client Signaling

### Content Steering

> ℹ️ **Priority: MUST** (*Currently supported by VHS*)
>
> All major open source players support this feature.

|           | URLS                                                                                                              |
|-----------|-------------------------------------------------------------------------------------------------------------------|
| HLS Spec  | https://developer.apple.com/streaming/HLSContentSteeringSpecification.pdf                                         |
| DASH Spec | https://dashif.org/docs/DASH-IF-CTS-00XX-Content-Steering-Community-Review.pdf                                    |
| Shaka     | https://github.com/shaka-project/shaka-player/blob/main/lib/util/content_steering_manager.js                      |
| dash.js   | https://github.com/Dash-Industry-Forum/dash.js/blob/development/src/dash/controllers/ContentSteeringController.js |
| hls.js    | https://github.com/video-dev/hls.js/blob/master/src/controller/content-steering-controller.ts                     |


### Common-Media-Client-Data (CMCD)

> ℹ️ **Priority: SHOULD** (*Currently NOT supported by VHS*)
> 
> All major open source players support this feature. We may consider re-using `@svta/common-media-library/cmcd/*` utils.

|         | URLS                                                                                              |
|---------|---------------------------------------------------------------------------------------------------|
| Spec    | https://cdn.cta.tech/cta/media/media/resources/standards/pdfs/cta-5004-final.pdf                  |
| Shaka   | https://github.com/shaka-project/shaka-player/blob/main/lib/util/cmcd_manager.js                  |
| dash.js | https://github.com/Dash-Industry-Forum/dash.js/blob/development/src/streaming/models/CmcdModel.js |
| hls.js  | https://github.com/video-dev/hls.js/blob/master/src/controller/cmcd-controller.ts                 |

### Common-Media-Server-Data (CMSD)

> ℹ️ **Priority: SHOULD** (*Currently NOT supported by VHS*)
>
> Limited support in shaka and dash.js. Mainly for abr. We may consider re-using `@svta/common-media-library/cmsd/*` utils.

|         | URLS                                                                                              |
|---------|---------------------------------------------------------------------------------------------------|
| Spec    | https://cdn.cta.tech/cta/media/media/resources/standards/pdfs/cta-5006-final.pdf                  |
| Shaka   | https://github.com/shaka-project/shaka-player/blob/main/lib/util/cmsd_manager.js                  |
| dash.js | https://github.com/Dash-Industry-Forum/dash.js/blob/development/src/streaming/models/CmsdModel.js |
| hls.js  | Not Implemented                                                                                   |

# Non-Functional Requirements

TBD: Describe non-functional Requirements, eg: build your own player, pipeline, size, etc...

# Entities

TBD: Describe entities and their relations

# Scenarios

TBD: Describe Scenarios with flow charts and comments (where possible use connections to FRs and NFRs)
