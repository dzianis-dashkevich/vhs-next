# Table of Content

* [Summary](#summary)
* [Motivation](#motivation)
* [Monorepo Structure](#monorepo-structure)
* [Functional Requirements](#functional-requirements)
* [Non-Functional Requirements](#non-functional-requirements)
  * [Browsers Matrix Support](#browsers-matrix-support)
* [Entities](#entities)
* [Scenarios](#scenarios)
* [References](#references)
    * [HLS Key Rotation](#hls-key-rotation)
    * [HLS Server Control](#hls-server-control)
    * [HLS Live Low Latency](#hls-live-low-latency)
    * [HLS Byte Range](#hls-byte-range)
    * [HLS In-Manifest Timed Metadata](#hls-in-manifest-timed-metadata)
    * [HLS MPEG-2 Transport Stream](#hls-mpeg-2-transport-stream)
    * [HLS Common Media Application Format](#hls-common-media-application-format)
    * [HLS Raw AAC](#hls-raw-aac)
    * [HLS Raw MP3](#hls-raw-mp3)
    * [HLS Raw AC3/EC3](#hls-raw-ac3ec3)
    * [HLS AES-128](#hls-aes-128)
    * [HLS ClearKey](#hls-clearkey)
    * [HLS WebVTT](#hls-webvtt)
    * [HLS IMSC](#hls-imsc)
    * [HLS Variable Substitution](#hls-variable-substitution)
    * [HLS Interstitial](#hls-interstitial)
    * [HLS Trick Play](#hls-trick-play)
    * [HLS In-manifest thumbnails i-frames](#hls-in-manifest-thumbnails-i-frames)
    * [HLS In-manifest thumbnails Roku](#hls-in-manifest-thumbnails-roku)
    * [HLS Session Data](#hls-session-data)
    * [DASH Indexed Addressing](#dash-indexed-addressing)
    * [DASH Explicit Addressing](#dash-explicit-addressing)
    * [DASH Simple Addressing](#dash-simple-addressing)
    * [DASH SegmentList](#dash-segmentlist)
    * [DASH Xlink](#dash-xlink)
    * [DASH Trick Play](#dash-trick-play)
    * [DASH In-manifest Timed Metadata](#dash-in-manifest-timed-metadata)
    * [DASH MPD Chaining](#dash-mpd-chaining)
    * [DASH Low Latency](#dash-low-latency)
    * [DASH In-manifest thumbnails](#dash-in-manifest-thumbnails)
    * [DRM](#drm)
    * [DRM Fairplay Legacy](#drm-fairplay-legacy)
    * [MPEG-2 TS AAC](#mpeg-2-ts-aac)
    * [MPEG-2 TS AC3/EC3](#mpeg-2-ts-ac3ec3)
    * [MPEG-2 TS MP3](#mpeg-2-ts-mp3)
    * [MPEG-2 TS Opus](#mpeg-2-ts-opus)
    * [MPEG-2 TS H264](#mpeg-2-ts-h264)
    * [MPEG-2 TS H265](#mpeg-2-ts-h265)
    * [MPEG-2 TS ID3](#mpeg-2-ts-id3)
    * [MPEG-2 TS CEA-608/CEA-708](#mpeg-2-ts-cea-608cea-708)
    * [MP4 CEA-608/CEA-708](#mp4-cea-608cea-708)
    * [MP4 IMSC](#mp4-imsc)
    * [MP4 VTT](#mp4-vtt)
    * [MP4 EMSG](#mp4-emsg)
    * [Offline](#offline)
    * [Content Steering](#content-steering)
    * [Common-Media-Client-Data](#common-media-client-data)
    * [Common-Media-Server-Data](#common-media-server-data)

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

| Name                                                        | VHS (current) | Priority | Notes                                                                                                                                                                                                                                   | References                                                                  |
|-------------------------------------------------------------|---------------|----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| **HLS**                                                     |               |          |                                                                                                                                                                                                                                         |                                                                             |
| HLS VOD Type                                                | ✅             | MUST     | N/A                                                                                                                                                                                                                                     | N/A                                                                         |
| HLS Live Type                                               | ✅             | MUST     | N/A                                                                                                                                                                                                                                     | N/A                                                                         |
| HLS Event Type                                              | ✅             | MUST     | N/A                                                                                                                                                                                                                                     | N/A                                                                         |
| HLS Live DVR                                                | ✅             | MUST     | Must support proper `seekable` API                                                                                                                                                                                                      | N/A                                                                         |
| HLS Discontinuities                                         | ✅             | MUST     | To reset transmuxers and other in-memory pipelines (eg: cea-608/cea-708)                                                                                                                                                                | N/A                                                                         |
| HLS Alternative Audio (when main is muxed)                  | ❌             | MUST     | If alternative audio is enabled, in-segment audio should be ignored                                                                                                                                                                     | N/A                                                                         |
| HLS Standalone Media Playlist Loading                       | ✅             | MUST     | Player should support standalone media playlist load (outside of the multivariant playlist)                                                                                                                                             | N/A                                                                         |
| HLS Key Rotation                                            | ✅             | MUST     | Player should be able to rotate ext-x-keys if multiple keys are presented in the playlist                                                                                                                                               | [hls-key-rotation](#hls-key-rotation)                                       |
| HLS Server Control                                          | ✅             | MUST     | N/A                                                                                                                                                                                                                                     | [hls-server-control](#hls-server-control)                                   |
| HLS Live Low Latency                                        | ✅             | MUST     | N/A                                                                                                                                                                                                                                     | [hls-live-low-latency](#hls-live-low-latency)                               |
| HLS Byte Range                                              | ✅             | MUST     | N/A                                                                                                                                                                                                                                     | [hls-byte-range](#hls-byte-range)                                           |
| HLS In-Manifest Timed Metadata                              | ✅             | MUST     | Both in-segment (id3 or emsg) and in-manifest (ext-x-DateRange) metadata will be processed. We should log warning.                                                                                                                      | [hls-in-manifest-timed-metadata](#hls-in-manifest-timed-metadata)           |
| HLS MPEG-2 Transport Stream                                 | ✅             | MUST     | Widevine/Playready are not supported for mpeg-2 ts. Fairplay should supported only by native apple's player.                                                                                                                            | [hls-mpeg-2-transport-stream](#hls-mpeg-2-transport-stream)                 |
| HLS Common Media Application Format (CMAF)                  | ✅             | MUST     | N/A                                                                                                                                                                                                                                     | [hls-common-media-application-format](#hls-common-media-application-format) |
| HLS Raw AAC                                                 | ❌             | SHOULD   | Shaka and hls.js transmux to AAC in MP4                                                                                                                                                                                                 | [hls-raw-aac](#hls-raw-aac)                                                 |
| HLS Raw MP3                                                 | ❌             | SHOULD   | Shaka and hls.js transmux to MP3 in MP4 container                                                                                                                                                                                       | [hls-raw-mp3](#hls-raw-mp3)                                                 |
| HLS Raw AC3/EC3                                             | ❌             | SHOULD   | Shaka and hls.js transmux to AC3/EC3 in MP4 container                                                                                                                                                                                   | [hls-raw-ac3/ec3](#hls-raw-ac3ec3)                                          |
| HLS WebVTT                                                  | ✅             | MUST     | HLS WebVTT is not encapsulated in fmp4 and timestamp sync is implemented via X-TIMESTAMP-MAP. If content has in-manifest subtitles and in-segment subtitles (cea-608/708) - all will be emitted to the user.                            | [hls-webVTT](#hls-webvtt)                                                   |
| HLS IMSC                                                    | ❌             | MUST     | HLS IMSC is encapsulated in fmp4. If content has in-manifest subtitles and in-segment subtitles (cea-608/708) - all will be emitted to the user.                                                                                        | [hls-imsc](#hls-imsc)                                                       |
| HLS AES-128                                                 | ✅             | MUST     | VHS implementation covers only AES-128 and creates additional WebWorker per each player instance. Shaka uses native Web Crypto subtle API. hls.js uses combination of native web crypto and fallbacks to the software aes decrypt impl. | [hls-aes-128](#hls-aes-128)                                                 |
| HLS SAMPLE-AES and SAMPLE-AES-CTR with KEYFORMAT="identity" | ❌             | COULD    | Key format "identity" is `ClearKey`. It is nice for testing EME flow, but does not provide actual security.                                                                                                                             | [hls-clearKey](#hls-clearkey)                                               |
| HLS Variable Substitution                                   | ❌             | MUST     | N/A                                                                                                                                                                                                                                     | [hls-variable-substitution](#hls-variable-substitution)                     |
| HLS Interstitial                                            | ❌             | COULD    | Content producers can insert separate interstitial content into their primary presentations in order to display advertising, branding, or other information to viewers. It is implemented via EXT-X-DATERANGE.                          | [hls-interstitial](#hls-interstitial)                                       |
| HLS Trick Play                                              | ❌             | COULD    | Trick play could be implemented via Ext-x-iframes-only playlists.                                                                                                                                                                       | [hls-trick-play](#hls-trick-play)                                           |
| HLS In-manifest thumbnails (ext-x-iframes-only)             | ❌             | SHOULD   | Official In-manifest support via Ext-x-iframes-only playlists with mjpg codec.                                                                                                                                                          | [hls-in-manifest-thumbnails-i-frames](#hls-in-manifest-thumbnails-i-frames) |
| HLS In-manifest thumbnails (roku)                           | ❌             | COULD    | It is unofficial spec, but nice to have.                                                                                                                                                                                                | [hls-in-manifest-thumbnails-roku](#hls-in-manifest-thumbnails-roku)         |
| HLS Session Data                                            | ❌             | SHOULD   | N/A                                                                                                                                                                                                                                     | [hls-session-data](#hls-session-data)                                       |
| **DASH**                                                    |               |          |                                                                                                                                                                                                                                         |                                                                             |
| DASH VOD                                                    | ✅             | MUST     | N/A                                                                                                                                                                                                                                     | N/A                                                                         |
| DASH Live                                                   | ✅             | MUST     | N/A                                                                                                                                                                                                                                     | N/A                                                                         |
| DASH In-Progress Recording                                  | ✅             | MUST     | N/A                                                                                                                                                                                                                                     | N/A                                                                         |
| DASH Live DVR (timeShiftBufferDepth)                        | ❌             | MUST     | Must support proper `seekable` API                                                                                                                                                                                                      | N/A                                                                         |
| DASH Multi-Period                                           | ✅             | MUST     | To reset transmuxers and other in-memory pipelines (eg: cea-608/cea-708)                                                                                                                                                                | N/A                                                                         |
| DASH Xlink (onLoad)                                         | ❌             | SHOULD   | onLoad: an application should dereference the remote element entity immediately on loading the MPD.                                                                                                                                     | [dash-xlink](#dash-xlink)                                                   |
| DASH Xlink (onRequest)                                      | ❌             | COULD    | onRequest (default): according to W3C Xlink, an application should dereference the remote element entity only on a post-loading event triggered for the purpose of dereferencing.                                                       | [dash-xlink](#dash-xlink)                                                   |
| DASH Segment Index Addressing                               | ✅             | MUST     | SegmentBase                                                                                                                                                                                                                             | [dash-indexed-addressing](#dash-indexed-addressing)                         |
| DASH Segment Explicit Addressing                            | ✅             | MUST     | SegmentTemplate with SegmentTimeline                                                                                                                                                                                                    | [dash-explicit-addressing](#dash-explicit-addressing)                       |
| DASH Segment Simple Addressing                              | ✅             | MUST     | SegmentTemplate without SegmentTimeline                                                                                                                                                                                                 | [dash-simple-addressing](#dash-simple-addressing)                           |
| DASH Segment SegmentList Addressing                         | ✅             | MUST     | N/A                                                                                                                                                                                                                                     | [dash-segmentList](#dash-segmentlist)                                       |
| DASH In-manifest timed Metadata                             | ✅             | MUST     | via EventStream                                                                                                                                                                                                                         | [dash-in-manifest-timed-metadata](#dash-in-manifest-timed-metadata)         |
| DASH MPD-Chaining                                           | ❌             | COULD    | MPD chaining provides a mechanism to indicate that, at the end of one Media Presentation, a new Media Presentation starts.                                                                                                              | [dash-mpd-chaining](#dash-mpd-chaining)                                     |
| DASH Low Latency                                            | ❌             | SHOULD   | N/A                                                                                                                                                                                                                                     | [dash-low-latency](#dash-low-latency)                                       |
| DASH Trick Play                                             | ❌             | COULD    | N/A                                                                                                                                                                                                                                     | [dash-trick-play](#dash-trick-play)                                         |
| DASH In-manifest thumbnails                                 | ❌             | SHOULD   | N/A                                                                                                                                                                                                                                     | [dash-in-manifest-thumbnails](#dash-in-manifest-thumbnails)                 |
| **DRM**                                                     |               |          |                                                                                                                                                                                                                                         |                                                                             |
| Widevine                                                    | ✅             | MUST     | N/A                                                                                                                                                                                                                                     | [drm](#drm)                                                                 |
| PlayReady                                                   | ✅             | MUST     | N/A                                                                                                                                                                                                                                     | [drm](#drm)                                                                 |
| Fairplay                                                    | ✅             | MUST     | N/A                                                                                                                                                                                                                                     | [drm](#drm)                                                                 |
| Fairplay Legacy                                             | ✅             | MUST     | Legacy Fairplay is not aligned with the current version of the EME spec. we should use polyfill-based approach, check shaka's impl in the references.                                                                                   | [drm-fairplay-legacy](#drm-fairplay-legacy)                                 |
| ClearKey                                                    | ❌             | COULD    | See HLS and DASH sections about ClearKey.                                                                                                                                                                                               | [drm](#drm)                                                                 |
| **Containers**                                              |               |          |                                                                                                                                                                                                                                         |                                                                             |
| MPEG-2 TS AAC (Audio Codec)                                 | ✅             | MUST     | Transmux to AAC in MP4                                                                                                                                                                                                                  | [mpeg-2-ts-aac](#mpeg-2-ts-aac)                                             |
| MPEG-2 TS AC/EC3 (Audio Codec)                              | ❌             | SHOULD   | Transmux to AC/EC3 in MP4                                                                                                                                                                                                               | [mpeg-2-ts-ac3/ec3](#mpeg-2-ts-ac3ec3)                                      |
| MPEG-2 TS MP3 (Audio Codec)                                 | ❌             | SHOULD   | Transmux to MP3 in MP4                                                                                                                                                                                                                  | [mpeg-2-ts-mp3](#mpeg-2-ts-mp3)                                             |
| MPEG-2 TS Opus (Audio Codec)                                | ❌             | SHOULD   | Transmux to Opus in MP4                                                                                                                                                                                                                 | [mpeg-2-ts-opus](#mpeg-2-ts-opus)                                           |
| MPEG-2 TS H264 (Video Codec)                                | ✅             | MUST     | Transmux to H264 in MP4                                                                                                                                                                                                                 | [mpeg-2-ts-h264](#mpeg-2-ts-h264)                                           |
| MPEG-2 TS H265 (Video Codec)                                | ❌             | MUST     | Transmux to H265 in MP4                                                                                                                                                                                                                 | [mpeg-2-ts-h265](#mpeg-2-ts-h265)                                           |
| MPEG-2 TS ID3 (timed metadata)                              | ✅             | MUST     | Should be emitted alongside with in-manifest timed metadata (if any)                                                                                                                                                                    | [mpeg-2-ts-id3](#mpeg-2-ts-id3)                                             |
| MPEG-2 TS CEA-608/CEA-708 (text tracks)                     | ✅             | MUST     | Should be emitted alongside with in-manifest text tracks (if any)                                                                                                                                                                       | [mpeg-2-ts-cea-608/cea-708](#mpeg-2-ts-cea-608cea-708)                      |
| MP4 CEA-608/CEA-708 (text tracks)                           | ✅             | MUST     | Should be emitted alongside with in-manifest text tracks (if any)                                                                                                                                                                       | [mp4-cea-608/cea-708](#mp4-cea-608cea-708)                                  |
| MP4 IMSC (text-tracks)                                      | ❌             | MUST     | Should be emitted alongside with in-manifest text tracks (if any)                                                                                                                                                                       | [mp4-imsc](#mp4-imsc)                                                       |
| MP4 VTT (text-tracks)                                       | ❌             | MUST     | Should be emitted alongside with in-manifest text tracks (if any)                                                                                                                                                                       | [mp4-vtt](#mp4-vtt)                                                         |
| MP4 EMSG (timed-metadata)                                   | ✅             | MUST     | Should be emitted alongside with in-manifest timed-metadata (if any)                                                                                                                                                                    | [mp4-emsg](#mp4-emsg)                                                       |
| **Server-Client Signaling**                                 |               |          |                                                                                                                                                                                                                                         |                                                                             |
| Content Steering                                            | ✅             | MUST     | N/A                                                                                                                                                                                                                                     | [content-steering](#content-steering)                                       |
| Common-Media-Client-Data (CMCD)                             | ❌             | SHOULD   | We may consider re-using `@svta/common-media-library/cmcd/*` utils                                                                                                                                                                      | [common-media-client-data](#common-media-client-data)                       |
| Common-Media-Server-Data (CMSD)                             | ❌             | SHOULD   | Limited support in shaka and dash.js. Mainly for abr. We may consider re-using `@svta/common-media-library/cmsd/*` utils                                                                                                                | [common-media-server-data](#common-media-server-data)                       |
| **Offline**                                                 |               |          |                                                                                                                                                                                                                                         |                                                                             |
| IndexDB VOD persistence                                     | ❌             | SHOULD   | N/A                                                                                                                                                                                                                                     | [offline](#offline)                                                         |
| Offline DRM                                                 | ❌             | SHOULD   | support persisentState="required", sessionType="persistent-license" and mediaKeySession.load(sessionId)                                                                                                                                 | [offline](#offline)                                                         |

# Non-Functional Requirements

| Name                                        | VHS (current) | Priority | Notes                                                                                                                                                                                                                                                                                                                                                |
|---------------------------------------------|---------------|----------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Events API                                  | ✅             | MUST     | Current VHS implementation provides very limited Events System. Player must provide predictable and robust set of events with extensive event data.                                                                                                                                                                                                  |
| Errors API                                  | ✅             | MUST     | Current VHS implementation provides very limited Errors System. Player must provide predictable and robust set of errors with extensive error data. Each error must have category, code and severity info, alongside with error data (if any).                                                                                                       |
| Configurations API                          | ✅             | MUST     | Current VHS implementation provides very limited Configurations API. Users should be able to configure different aspects of player's workflow during runtime (where possible).                                                                                                                                                                       |
| ABR Strategies API                          | ❌             | MUST     | Current VHS implementation provides very limited ABR System. dash.js has an excellent example of ABR Strategies API. We should implement something similar.                                                                                                                                                                                          |
| Stream API                                  | ❌             | MUST     | Player should be able to utilize browser's fetch Readable/Writable Streams API and continuously process chunks of data instead of waiting while a whole resource is loaded.                                                                                                                                                                          |
| Managed Media Source API                    | ❌             | MUST     | https://www.w3.org/TR/media-source-2/#managedmediasource-interface currently supported only on Safari (mac, ios, ipados) 17+                                                                                                                                                                                                                         |
| Interceptors API                            | ❌             | MUST     | In contrast to Events API where player notifies clients, Interceptors API allows users to intercept and modify data during important processing steps (including network requests).                                                                                                                                                                  |
| Networking API                              | ❌             | MUST     | Users should be able to configure player networking behavior for **specific request type**: `maxAttempts`, `initialDelay`, `delayFactor`, `fuzzFactor` and `timeout`                                                                                                                                                                                 |
| Service Locator API                         | ❌             | MUST     | Users should be able to replace ANY service managed by ServiceLocator based on their needs (including "during player runtime"). Users must implement interfaces for the services they want to replace or use service implementations provided from us. This API offers "build your own player" feature and allows extremely flexible customizations. |
| Size: Opt-in features as separate bundles   | ❌             | MUST     | Most of the features should be provided via separate bundles and not included into main player bundle by default.                                                                                                                                                                                                                                    |
| Size: Popular bundle presets                | ❌             | MUST     | While users can create their own player using Service Locator API, we should provide popular features bundle presets. eg: (player + hls pipeline) or (player + dash pipeline) or (player + hls and dash pipelines), etc..                                                                                                                            |
| CLI prompt player builder                   | ❌             | SHOULD   | In case users wants to build a player with a very custom set of features or don't know exactly what features should be included they can use CLI prompt-based tool.                                                                                                                                                                                  |
| Demo page with timeline visualization       | ❌             | COULD    | hls.js has an excellent example of timeline visualisation on their demo page. We can build something similar.                                                                                                                                                                                                                                        |
| Live Range Recorder with Server Replay Tool | ❌             | COULD    | tool to record a problematic part from a live stream and replay (vod or live mode)                                                                                                                                                                                                                                                                   |
| Generated API reference                     | ❌             | MUST     | N/A                                                                                                                                                                                                                                                                                                                                                  |
| Tutorials Documentation                     | ✅             | MUST     | N/A                                                                                                                                                                                                                                                                                                                                                  |
| Contributors Guides                         | ✅             | MUST     | N/A                                                                                                                                                                                                                                                                                                                                                  |
| Generate Changelog                          | ✅             | MUST     | N/A                                                                                                                                                                                                                                                                                                                                                  |
| Public Roadmap                              | ❌             | SHOULD   | We may maintain public roadmap with expected features.                                                                                                                                                                                                                                                                                               |

## Browsers Matrix Support

|                     | Windows      | MacOS          | Linux        | Android      | iOS            | iPadOS         | Other                               |
|---------------------|--------------|----------------|--------------|--------------|----------------|----------------|-------------------------------------|
| Chrome              | >= 76 (2021) | >= 76 (2021)   | >= 76 (2021) | >= 76 (2021) | >= 76 (2021)   | >= 76 (2021)   | N/A                                 |
| Firefox             | >= 85 (2021) | >= 85 (2021)   | >= 85 (2021) | >= 85 (2021) | >= 85 (2021)   | >= 85 (2021)   | N/A                                 |
| Edge (Chromium)     | >= 76 (2021) | >= 76 (2021)   | >= 76 (2021) | >= 76 (2021) | >= 76 (2021)   | >= 76 (2021)   | N/A                                 |
| Safari              | N/A          | >= 15.x (2021) | N/A          | N/A          | >= 15.x (2021) | >= 15.x (2021) | N/A                                 |
| Tizen (Chromium)    | N/A          | N/A            | N/A          | N/A          | N/A            | N/A            | >= Tizen 6.x (2021, Chromium 76)    |
| WebOS TV (Chromium) | N/A          | N/A            | N/A          | N/A          | N/A            | N/A            | >= WebOS TV 6.x (2021, Chromium 79) |

> ⚠️ **Note**
> 
> iOS < 17.1 does not support MSE so only native playback is supported. iOS >= 17.1 supports MMSE (managed MSE)
> 
> IPadOS < 17.0 does not support MSE, so only native playback is supported. IPadOS >= 17.0 supports MMSE (managed MSE)

# Entities

```ts
// Events API

enum PlayerEventType {
  Error = 'Error'
};

interface EventTypeToEventMap {
  [PlayerEventType.Error]: PlayerErrorEvent;
}

abstract class PlayerEvent {
  public abstract readonly type: PlayerEventType;
}


class PlayerErrorEvent extends PlayerEvent {
  type = PlayerEventType.Error;
  error: PlayerError;

  constructor(playerError: PlayerError) {
    super();
    this.error = playerError;
  }
}
```

```ts
// Errors API

enum ErrorCategory {
  
}

enum ErrorCode {
  
}

abstract class PlayerError {
  public abstract readonly category: ErrorCategory;
  public abstract readonly code: ErrorCode;
  public abstract readonly isFatal: boolean;
}
```

```ts
// Interceptors API

enum InterpceptorType {
  
}

enum AsyncInterceptorType {
  
}

interface InterceptorTypeToPayloadMap {
  
}

interface AsyncInterceptorTypeToPayloadMap {
  
}
```

```ts
// Networking API

enum NetworkRequestType {
  
}

interface NetworkingConfiguration {
  /**
   * The maximum number of requests before we fail.
   * Defaults to `1`
   */
  maxAttempts: number;

  /**
   * The base delay in ms between retries.
   * Defaults to `1_000`
   */
  initialDelay: number;

  /**
   * Increase delay for each retry: delay += (delay * delayFactor).
   * Defaults to `0.2`
   */
  delayFactor: number;

  /**
   * Do not send retry requests in the exact same timing, but rather fuzz it by some range, eg: if the delay = 3000, and fuzz factor is 0.1,
   * then the requests will be made somewhere in the following time range range: [2700, 3300].
   * Defaults to `0.2`
   */
  fuzzFactor: number;

  /**
   * The timeout in ms, after which we abort.
   * Defaults to `20_000`
   */
  timeout: number;
}
```

```ts
// Service Locator API

// Service Locator API provides 3 main features:
// - Build your own player. (Replace default fallbacks with default implementations) 
// - Customize ANY service. (Replace default fallback or default implementations with your own implementations)
// - access to ANY service. (You can use service locator to access any service from any place)

// Each service must implement TrasferableStateProvider

interface TransferableStateProvider<T> {
  getTransferableState(): T;
}

/**
 * Use your own implemenration example:
 * 
 * import { serviceLocator } from '@videojs/playback';
 * 
 * const currentHlsParser = serviceLocator.getHlsParser();
 * const currentHlsParserState = currentHlsParser.getTransferableState();
 * 
 * // you must implement expected interface, you can optinally use serviceLocator and state in your HlsParser impl, or any other deps that you want
 * const hlsParser = new YourHlsParserImpl({ serviceLocator, state: currentHlsParserState, customDependency: MyBusinessLogicCustomDependency });
 * serviceLocator.replaceHlsParser(hlsParser);
 */

/**
 * Use our default implementation example:
 * import { serviceLocator } from '@videojs/playback';
 * import { HlsParser } from '@videojs/hls-parser'
 * 
 * const currentHlsParser = serviceLocator.getHlsParser();
 * const currentHlsParserState = currentHlsParser.getTransferableState();
 *
 * // implementations provided by us already imeplements expected interface
 * const hlsParser = new HlsParser({ serviceLocator, state: currentHlsParserState });
 */

interface TrasferableStateProvider<T> {
  getTransferableState(): T;
}

interface ServiceLocator {
  
}

```

```ts
interface PlayerConfiguration {
  
}
```

```ts
type EventHandler<T> = (event: T) => void;

type Interceptor<T> = (payload: T) => T;

type AsyncInterceptor<T> = (payload: T) => Promise<T>;

interface Player {
  // Configurations API
  
  // get deep copy of the current configuration
  getConfigurationSnapshot(): PlayerConfiguration;
  // update current configuration
  updateConfiguration(configurationChunk: DeepPartial<PlayerConfiguration>): void;
  // reset configuration to default
  resetConfiguration(): void;

  // Events API
  // Errors API is available via PlayerErrorEvent

  // register event handler for specific event
  addEventListener<K extends PlayerEventType>(eventType: K, eventHandler: EventHandler<EventTypeToEventMap[K]>): void;
  // register event handler for all events '*', mainly for debug purposes
  addEventListenerForAllEvents(eventHandler: EventHandler<PlayerEvent>): void;
  // register event handler for specific events once
  once<K extends PlayerEventType>(eventType: K, eventHandler: EventHandler<EventTypeToEventMap[K]>): void;
  // remove specific registered event handler for specific event
  removeEventListener<K extends PlayerEventType>(eventType: K, eventHandler: EventHandler<EventTypeToEventMap[K]>): void;
  // remove all registered event handlers for specific event
  removeAllEventListenersForType<K extends PlayerEventType>(eventType: K): void;
  // remove all registered event handlers for all events
  removeAllEventListeners(): void;

  // Interceptors API

  // register interceptor for a specific player's processing step. (sync)
  addInterceptor<K extends InterpceptorType>(interceptorType: K, interceptor: Interceptor<InterceptorTypeToPayloadMap[K]>): void;
  // register interceptor for a specific player's processing step. (async, some steps can be processed as async, so we should have a separate api)
  addAsyncInterceptor<K extends AsyncInterceptorType>(interceptorType: K, interceptor: AsyncInterceptor<AsyncInterceptorTypeToPayloadMap[K]>): void;
  // remove specific interceptor for specific step (sync)
  removeInterceptor<K extends InterpceptorType>(interceptorType: K, interceptor: Interceptor<InterceptorTypeToPayloadMap[K]>): void;
  // remove specific interceptor for specific step (async)
  removeAsyncInterceptor<K extends AsyncInterceptorType>(interceptorType: K, interceptor: AsyncInterceptor<AsyncInterceptorTypeToPayloadMap[K]>): void;
  // remove all interceptors
  removeAllInterceptors(): void;

  // Networking API

  // update networking configuration for specific request type
  updateNetworkingConfiguration(requestType: NetworkRequestType, networkingConfiguration: NetworkingConfiguration): void;
  // reset networking configuration to default values for specific request type
  resetNetworkingConfiguration(requestType: NetworkRequestType): void;
  // reset networking configurations for all reques types
  resetAllNetworkingConfigurations(): void;
}
```

TBD: Describe entities and their relations
TBD: support loading external WebVTT with subtitles for VOD
TBD: support loading external WebVTT with image/sprites for VOD

# Scenarios

TBD: Describe Scenarios with flow charts and comments (where possible use connections to FRs and NFRs)

# References


> ⚠️ **Note**
>
> DASH references may contain both spec and dash-if IP links.
> 
> DASH spec is ISO/IEC 23009-1 and can be downloaded here:
> 
> https://standards.iso.org/ittf/PubliclyAvailableStandards/ --> ISO/IEC 23009-1 (current is 5th 2022 edition)
> 
> DASH-IF (Industry Forum) Guidelines for Implementation: DASH-IF Interoperability Points (Deployed living document) is here:
> 
> https://dashif-documents.azurewebsites.net/DASH-IF-IOP/master/DASH-IF-IOP.html

### HLS Key Rotation

|      | URLS                                                                              |
|------|-----------------------------------------------------------------------------------|
| Spec | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-4.4.4.4 |

### HLS Server Control

|      | URLS                                                                              |
|------|-----------------------------------------------------------------------------------|
| Spec | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-4.4.3.8 |

### HLS Live Low Latency

|        | URLS                                                                                |
|--------|-------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-3.2       |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_parser.js#L3592 |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/loader/m3u8-parser.ts#L595      |

### HLS Byte Range

|        | URLS                                                                                |
|--------|-------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-4.4.4.2   |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_parser.js#L3647 |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/loader/m3u8-parser.ts#L388      |

### HLS In-Manifest Timed Metadata

|        | URLS                                                                                |
|--------|-------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-4.4.5.1   |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_parser.js#L3874 |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/loader/date-range.ts            |

### HLS MPEG-2 Transport Stream

|        | URLS                                                                                    |
|--------|-----------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-3.1.1         |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/transmuxer/ts_transmuxer.js |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/demux/tsdemuxer.ts                  |

### HLS Common Media Application Format

|        | URLS                                                                                |
|--------|-------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-3.1.2     |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_parser.js#L3431 |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/loader/m3u8-parser.ts#L540      |

### HLS Raw AAC

|        | URLS                                                                                     |
|--------|------------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-3.1.3          |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/transmuxer/aac_transmuxer.js |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/demux/audio/aacdemuxer.ts            |

### HLS Raw MP3

|        | URLS                                                                                     |
|--------|------------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-3.1.3          |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/transmuxer/mp3_transmuxer.js |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/demux/audio/mp3demuxer.ts            |

### HLS Raw AC3/EC3

|        | URLS                                                                                     |
|--------|------------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-3.1.3          |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/transmuxer/ec3_transmuxer.js |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/demux/audio/ac3-demuxer.ts           |

### HLS AES-128

|        | URLS                                                                                |
|--------|-------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-4.4.4.4   |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_parser.js#L3154 |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/crypt/decrypter.ts              |

### HLS ClearKey

|        | URLS                                                                                |
|--------|-------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-4.4.4.4   |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_parser.js#L4773 |
| hls.js | identity" format SAMPLE-AES decryption of MPEG-2 TS segments only                   |

### HLS WebVTT

|        | URLS                                                                                |
|--------|-------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-3.1.4     |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/text/vtt_text_parser.js |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/utils/webvtt-parser.ts          |

### HLS IMSC

|        | URLS                                                                                |
|--------|-------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-3.1.5     |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/text/mp4_ttml_parser.js |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/utils/imsc1-ttml-parser.ts      |

### HLS Variable Substitution

|        | URLS                                                                               |
|--------|------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-4.3      |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_utils.js#L115  |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/utils/variable-substitution.ts |

### HLS Interstitial

|        | URLS                                                                         |
|--------|------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#appendix-D |
| Shaka  | Check this PR: https://github.com/shaka-project/shaka-player/pull/6761       |
| hls.js | Not Implemented                                                              |

### HLS Trick Play

|        | URLS                                                                              |
|--------|-----------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-4.4.3.6 |
| Shaka  | Not Implemented                                                                   |
| hls.js | Not Implemented                                                                   |

### HLS In-manifest thumbnails i-frames

|        | URLS                                                                               |
|--------|------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-4.4.3.6  |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_parser.js#L933 |
| hls.js | Not Implemented                                                                    |

### HLS In-manifest thumbnails Roku

|        | URLS                                                                                   |
|--------|----------------------------------------------------------------------------------------|
| Spec   | https://github.com/image-media-playlist/spec/blob/master/image_media_playlist_v0_4.pdf |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_parser.js#L933     |
| hls.js | Not Implemented                                                                        |

### HLS Session Data

|        | URLS                                                                                |
|--------|-------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-4.4.6.4   |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_parser.js#L1295 |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/loader/m3u8-parser.ts#L152      |

### DASH Indexed Addressing

|       | URLS                                                                                              |
|-------|---------------------------------------------------------------------------------------------------|
| Spec  | https://dashif-documents.azurewebsites.net/DASH-IF-IOP/master/DASH-IF-IOP.html#addressing-indexed |
| Shaka | https://github.com/shaka-project/shaka-player/blob/main/lib/dash/segment_base.js                  |

### DASH Explicit Addressing

|       | URLS                                                                                               |
|-------|----------------------------------------------------------------------------------------------------|
| Spec  | https://dashif-documents.azurewebsites.net/DASH-IF-IOP/master/DASH-IF-IOP.html#addressing-explicit |
| Shaka | https://github.com/shaka-project/shaka-player/blob/main/lib/dash/segment_template.js               |

### DASH Simple Addressing

|       | URLS                                                                                             |
|-------|--------------------------------------------------------------------------------------------------|
| Spec  | https://dashif-documents.azurewebsites.net/DASH-IF-IOP/master/DASH-IF-IOP.html#addressing-simple |
| Shaka | https://github.com/shaka-project/shaka-player/blob/main/lib/dash/segment_template.js             |

### DASH SegmentList

|       | URLS                                                                             |
|-------|----------------------------------------------------------------------------------|
| Shaka | https://github.com/shaka-project/shaka-player/blob/main/lib/dash/segment_list.js |

### DASH Xlink

|         | URLS                                                                                                         |
|---------|--------------------------------------------------------------------------------------------------------------|
| Spec    | https://dashif-documents.azurewebsites.net/DASH-IF-IOP/master/DASH-IF-IOP.html#xlink-feature                 |
| Shaka   | https://github.com/shaka-project/shaka-player/blob/main/lib/dash/mpd_utils.js#L513                           |
| dash.js | https://github.com/Dash-Industry-Forum/dash.js/blob/development/src/streaming/controllers/XlinkController.js |

### DASH Trick Play

|       | URLS                                                                                     |
|-------|------------------------------------------------------------------------------------------|
| Spec  | https://dashif-documents.azurewebsites.net/DASH-IF-IOP/master/DASH-IF-IOP.html#trickmode |
| Shaka | https://github.com/shaka-project/shaka-player/blob/main/lib/dash/dash_parser.js#L1522    |

### DASH In-manifest Timed Metadata

|       | URLS                                                                                  |
|-------|---------------------------------------------------------------------------------------|
| Shaka | https://github.com/shaka-project/shaka-player/blob/main/lib/dash/dash_parser.js#L1472 |

### DASH MPD Chaining

|       | URLS                                                                                                                                                                         |
|-------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Spec  | Section 5.11 in the iso ISO/IEC 23009-1.                                                                                                                                     |
| Shaka | https://github.com/shaka-project/shaka-player/blob/main/lib/dash/dash_parser.js#L805 <br/> https://github.com/shaka-project/shaka-player/blob/main/lib/player.js#L2657-L2677 |

### DASH Low Latency

|         | URLS                                                                                              |
|---------|---------------------------------------------------------------------------------------------------|
| Shaka   | https://github.com/shaka-project/shaka-player/blob/main/lib/media/streaming_engine.js#L1573-L1630 |
| dash.js | https://github.com/Dash-Industry-Forum/dash.js/wiki/Low-Latency-streaming                         |

### DASH In-manifest thumbnails

|         | URLS                                                                                                       |
|---------|------------------------------------------------------------------------------------------------------------|
| Spec    | https://dashif-documents.azurewebsites.net/DASH-IF-IOP/master/DASH-IF-IOP.html#thumbnails                  |
| Shaka   | https://github.com/shaka-project/shaka-player/blob/main/lib/dash/dash_parser.js#L2224-L2239                |
| dash.js | https://github.com/Dash-Industry-Forum/dash.js/blob/development/src/streaming/thumbnail/ThumbnailTracks.js |

### DRM

|         | URLS                                                                                     |
|---------|------------------------------------------------------------------------------------------|
| Spec    | https://www.w3.org/TR/encrypted-media/                                                   |
| Shaka   | https://github.com/shaka-project/shaka-player/blob/main/lib/media/drm_engine.js          |
| dash.js | https://github.com/Dash-Industry-Forum/dash.js/tree/development/src/streaming/protection |
| hls.js  | https://github.com/video-dev/hls.js/blob/master/src/controller/eme-controller.ts         |

### DRM Fairplay Legacy

|              | URLS                                                                                            |
|--------------|-------------------------------------------------------------------------------------------------|
| shaka webkit | https://github.com/shaka-project/shaka-player/blob/main/lib/polyfill/patchedmediakeys_webkit.js |
| shaka apple  | https://github.com/shaka-project/shaka-player/blob/main/lib/polyfill/patchedmediakeys_apple.js  |

### MPEG-2 TS AAC

|        | URLS                                                                                         |
|--------|----------------------------------------------------------------------------------------------|
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/transmuxer/ts_transmuxer.js#L248 |

### MPEG-2 TS AC3/EC3

|       | URLS                                                                                              |
|-------|---------------------------------------------------------------------------------------------------|
| Shaka | https://github.com/shaka-project/shaka-player/blob/main/lib/transmuxer/ts_transmuxer.js#L252-L256 |

### MPEG-2 TS MP3

|       | URLS                                                                                         |
|-------|----------------------------------------------------------------------------------------------|
| Shaka | https://github.com/shaka-project/shaka-player/blob/main/lib/transmuxer/ts_transmuxer.js#L260 |

### MPEG-2 TS Opus

|       | URLS                                                                                         |
|-------|----------------------------------------------------------------------------------------------|
| Spec  | https://datatracker.ietf.org/doc/html/rfc6716                                                |
| Shaka | https://github.com/shaka-project/shaka-player/blob/main/lib/transmuxer/ts_transmuxer.js#L262 |

### MPEG-2 TS H264

|        | URLS                                                                                         |
|--------|----------------------------------------------------------------------------------------------|
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/transmuxer/ts_transmuxer.js#L858 |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/demux/video/avc-video-parser.ts          |

### MPEG-2 TS H265

|        | URLS                                                                                         |
|--------|----------------------------------------------------------------------------------------------|
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/transmuxer/ts_transmuxer.js#L236 |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/demux/video/hevc-video-parser.ts         |

### MPEG-2 TS ID3

|        | URLS                                                                                         |
|--------|----------------------------------------------------------------------------------------------|
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/transmuxer/ts_transmuxer.js#L188 |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/demux/transmuxer.ts#L748                 |

### MPEG-2 TS CEA-608/CEA-708

|        | URLS                                                                             |
|--------|----------------------------------------------------------------------------------|
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/cea/ts_cea_parser.js |

### MP4 CEA-608/CEA-708

|       | URLS                                                                              |
|-------|-----------------------------------------------------------------------------------|
| Shaka | https://github.com/shaka-project/shaka-player/blob/main/lib/cea/mp4_cea_parser.js |

### MP4 IMSC

|         | URLS                                                                                              |
|---------|---------------------------------------------------------------------------------------------------|
| Shaka   | https://github.com/shaka-project/shaka-player/blob/main/lib/text/mp4_ttml_parser.js               |
| hls.js  | https://github.com/video-dev/hls.js/blob/master/src/utils/imsc1-ttml-parser.ts                    |
| dash.js | https://github.com/Dash-Industry-Forum/dash.js/blob/development/src/streaming/utils/TTMLParser.js |

### MP4 VTT

|         | URLS                                                                                             |
|---------|--------------------------------------------------------------------------------------------------|
| Shaka   | https://github.com/shaka-project/shaka-player/blob/main/lib/text/mp4_vtt_parser.js               |
| hls.js  | https://github.com/video-dev/hls.js/blob/master/src/utils/vttparser.ts                           |
| dash.js | https://github.com/Dash-Industry-Forum/dash.js/blob/development/src/streaming/utils/VTTParser.js |

### MP4 EMSG

|         | URLS                                                                                           |
|---------|------------------------------------------------------------------------------------------------|
| Shaka   | https://github.com/shaka-project/shaka-player/blob/main/lib/media/streaming_engine.js#L2292    |
| hls.js  | https://github.com/video-dev/hls.js/blob/master/src/utils/mp4-tools.ts#L1197                   |
| dash.js | https://github.com/Dash-Industry-Forum/dash.js/blob/development/src/streaming/vo/IsoBox.js#L66 |

### Offline

|         | URLS                                                                        |
|---------|-----------------------------------------------------------------------------|
| Shaka   | https://github.com/shaka-project/shaka-player/tree/main/lib/offline         |
| dash.js | https://github.com/Dash-Industry-Forum/dash.js/tree/development/src/offline |
| hls.js  | Not Implemented                                                             |

### Content Steering

|           | URLS                                                                                                              |
|-----------|-------------------------------------------------------------------------------------------------------------------|
| HLS Spec  | https://developer.apple.com/streaming/HLSContentSteeringSpecification.pdf                                         |
| DASH Spec | https://dashif.org/docs/DASH-IF-CTS-00XX-Content-Steering-Community-Review.pdf                                    |
| Shaka     | https://github.com/shaka-project/shaka-player/blob/main/lib/util/content_steering_manager.js                      |
| dash.js   | https://github.com/Dash-Industry-Forum/dash.js/blob/development/src/dash/controllers/ContentSteeringController.js |
| hls.js    | https://github.com/video-dev/hls.js/blob/master/src/controller/content-steering-controller.ts                     |

### Common-Media-Client-Data

|         | URLS                                                                                              |
|---------|---------------------------------------------------------------------------------------------------|
| Spec    | https://cdn.cta.tech/cta/media/media/resources/standards/pdfs/cta-5004-final.pdf                  |
| Shaka   | https://github.com/shaka-project/shaka-player/blob/main/lib/util/cmcd_manager.js                  |
| dash.js | https://github.com/Dash-Industry-Forum/dash.js/blob/development/src/streaming/models/CmcdModel.js |
| hls.js  | https://github.com/video-dev/hls.js/blob/master/src/controller/cmcd-controller.ts                 |

### Common-Media-Server-Data

|         | URLS                                                                                              |
|---------|---------------------------------------------------------------------------------------------------|
| Spec    | https://cdn.cta.tech/cta/media/media/resources/standards/pdfs/cta-5006-final.pdf                  |
| Shaka   | https://github.com/shaka-project/shaka-player/blob/main/lib/util/cmsd_manager.js                  |
| dash.js | https://github.com/Dash-Industry-Forum/dash.js/blob/development/src/streaming/models/CmsdModel.js |
| hls.js  | Not Implemented                                                                                   |
