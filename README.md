# Table of Content

* [Summary](#summary)
* [Motivation](#motivation)
* [Monorepo Structure](#monorepo-structure)
* [Functional Requirements](#functional-requirements)
  * [HLS Features](#hls-features)
    * [HLS VOD](#hls-vod)
    * [HLS Live](#hls-live)
    * [HLS Live DVR window](#hls-live-dvr-window)
    * [HLS Alternative Audio support (when main is muxed)](#hls-alternative-audio-support-when-main-is-muxed)
    * [HLS Single Media Playlist Load (without multivariant playlist)](#hls-single-media-playlist-load-without-multivariant-playlist)
    * [HLS Key Rotation](#hls-key-rotation)
    * [HLS Server Control](#hls-server-control)
    * [HLS Live Low Latency](#hls-live-low-latency)
    * [HLS Byte Range](#hls-byte-range)
    * [HLS In-Manifest Timed Metadata (via ext-x-dateRange)](#hls-in-manifest-timed-metadata-via-ext-x-daterange)
    * [HLS MPEG-2 Transport Stream](#hls-mpeg-2-transport-stream)
    * [HLS Common Media Application Format (CMAF)](#hls-common-media-application-format-cmaf)
    * [HLS Raw AAC](#hls-raw-aac)
    * [HLS Raw MP3](#hls-raw-mp3)
    * [HLS Raw AC3/EC3](#hls-raw-ac3ec3)
    * [HLS AES-128](#hls-aes-128)
    * [HLS SAMPLE-AES and SAMPLE-AES-CTR with KEYFORMAT="identity" (ClearKey)](#hls-sample-aes-and-sample-aes-ctr-with-keyformatidentity-clearkey)
    * [HLS WebVTT](#hls-webvtt)
    * [HLS IMSC Subtitles](#hls-imsc-subtitles)
    * [HLS Variable Substitution](#hls-variable-substitution)
    * [HLS Interstitial](#hls-interstitial)
    * [HLS Trick Play](#hls-trick-play)
    * [HLS In-manifest thumbnails (ext-x-iframes-only)](#hls-in-manifest-thumbnails-ext-x-iframes-only)
    * [HLS In-manifest thumbnails (Roku spec)](#hls-in-manifest-thumbnails-roku-spec)
  * [Dash Features](#dash-features)
    * [DASH In-manifest thumbnails](#dash-in-manifest-thumbnails)
  * [DRM](#drm)
    * [Widevine](#widevine)
    * [Fairplay](#fairplay)
    * [Fairplay (Legacy)](#fairplay-legacy)
    * [PlayReady](#playready)
    * [ClearKey](#clearkey)
  * [Container Support](#container-support)
    * [MPEG-2 TS AAC (Audio Codec)](#mpeg-2-ts-aac-audio-codec)
    * [MPEG-2 TS AC3 (Audio Codec)](#mpeg-2-ts-ac3-audio-codec)
    * [MPEG-2 TS EC3 (Audio Codec)](#mpeg-2-ts-ec3-audio-codec)
    * [MPEG-2 TS MP3 (Audio Codec)](#mpeg-2-ts-mp3-audio-codec)
    * [MPEG-2 TS Opus (Audio Codec)](#mpeg-2-ts-opus-audio-codec)
    * [MPEG-2 TS H264 (Video Codec)](#mpeg-2-ts-h264-video-codec)
    * [MPEG-2 TS H265 (Video Codec)](#mpeg-2-ts-h265-video-codec)
    * [MPEG-2 TS ID3 (timed metadata)](#mpeg-2-ts-id3-timed-metadata)
    * [MPEG-2 TS CEA-608/CEA-708 (text track)](#mpeg-2-ts-cea-608cea-708-text-track)
    * [MP4 CEA-608/CEA-708 (text-track)](#mp4-cea-608cea-708-text-track)
    * [MP4 IMSC (text-track)](#mp4-imsc-text-track)
    * [MP4 VTT (text-track)](#mp4-vtt-text-track)
    * [MP4 EMSG (timed-metadata)](#mp4-emsg-timed-metadata)
  * [Offline Viewing](#offline-viewing)
  * [Text (outside in-manifest and in-segment tracks)](#text-outside-in-manifest-and-in-segment-tracks)
    * [External WebVTT (only for VoD)](#external-webvtt-only-for-vod)
  * [Thumbnails (outside of HLS/DASH)](#thumbnails-outside-of-hlsdash)
    * [External WebVTT with images/sprites (only for VoD)](#external-webvtt-with-imagessprites-only-for-vod)
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

### HLS VOD
> ℹ️ **Priority: MUST** (*Currently supported by VHS*)

### HLS Live
> ℹ️ **Priority: MUST** (*Currently supported by VHS*)

### HLS Live DVR window
> ℹ️ **Priority: MUST** (*Currently supported by VHS*)

> ⚠️ **Note**
>
> Must support proper seekable API

### HLS Alternative Audio support (when main is muxed)
> ℹ️ **Priority: MUST** (*Currently NOT supported by VHS*)
> 
> If alternative audio is enabled, in-segment audio should be ignored

### HLS Single Media Playlist Load (without multivariant playlist)
> ℹ️ **Priority: MUST** (*Currently supported by VHS*)

### HLS Key Rotation
> ℹ️ **Priority: MUST** (*Currently supported by VHS*)

|      | URLS                                                                              |
|------|-----------------------------------------------------------------------------------|
| Spec | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-4.4.4.4 |

### HLS Server Control
> ℹ️ **Priority: MUST** (*Currently supported by VHS*)

|      | URLS                                                                              |
|------|-----------------------------------------------------------------------------------|
| Spec | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-4.4.3.8 |


### HLS Live Low Latency
> ℹ️ **Priority: MUST** (*Currently supported by VHS*)

|        | URLS                                                                                |
|--------|-------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-3.2       |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_parser.js#L3592 |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/loader/m3u8-parser.ts#L595      |

### HLS Byte Range
> ℹ️ **Priority: MUST** (*Currently supported by VHS*)

|        | URLS                                                                                |
|--------|-------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-4.4.4.2   |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_parser.js#L3647 |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/loader/m3u8-parser.ts#L388      |

### HLS In-Manifest Timed Metadata (via ext-x-dateRange)
> ℹ️ **Priority: MUST** (*Currently supported by VHS*)

> ⚠️ **Note**
>
> Both ID3 format carried in MPEG-2 TS or Emsg in CMAF/Fragmented MP4 and In-Manifest Timed Metadata will be processed.
> 
> But we should report warning and point that this might be content provider miss-configuration.

|        | URLS                                                                                |
|--------|-------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-4.4.5.1   |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_parser.js#L3874 |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/loader/date-range.ts            |

### HLS MPEG-2 Transport Stream

> ℹ️ **Priority: MUST** (*Currently supported by VHS*)
> 
> While the community is moving toward HLS CMAF, mpeg2-ts is still massively presented, so we have to support it.

> ⚠️ **Note**
>
> Widevine/Playready are not supported for mpeg-2 TS
> Fairplay should be supported only natively for mpeg-2 ts

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

### HLS Raw AAC

> ℹ️ **Priority: COULD** (*Currently NOT supported by VHS*)
> 
> Shaka and hls.js transmux to AAC in MP4 container

|        | URLS                                                                                     |
|--------|------------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-3.1.3          |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/transmuxer/aac_transmuxer.js |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/demux/audio/aacdemuxer.ts            |

### HLS Raw MP3
> ℹ️ **Priority: COULD** (*Currently NOT supported by VHS*)
>
> Shaka and hls.js transmux to MP3 in MP4 container

|        | URLS                                                                                     |
|--------|------------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-3.1.3          |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/transmuxer/mp3_transmuxer.js |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/demux/audio/mp3demuxer.ts            |

### HLS Raw AC3/EC3
> ℹ️ **Priority: COULD** (*Currently NOT supported by VHS*)
>
> Shaka and hls.js transmux to AC3/EC3 in MP4 container

|        | URLS                                                                                     |
|--------|------------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-3.1.3          |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/transmuxer/ec3_transmuxer.js |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/demux/audio/ac3-demuxer.ts           |

### HLS AES-128
> ℹ️ **Priority: MUST** (*Currently supported by VHS*)

> ⚠️ **Note**
> 
> Current VHS implementation covers only AES-128 and creates additional WebWorker per each player instance.
> 
> Shaka uses native Web Crypto subtle API
> 
> hls.js uses combination of native web crypto and fallbacks to the software aes decrypt impl.

|        | URLS                                                                                |
|--------|-------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-4.4.4.4   |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_parser.js#L3154 |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/crypt/decrypter.ts              |

### HLS SAMPLE-AES and SAMPLE-AES-CTR with KEYFORMAT="identity" (ClearKey)
> ℹ️ **Priority: COULD** (*Currently NOT supported by VHS*)

|        | URLS                                                                                |
|--------|-------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-4.4.4.4   |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_parser.js#L4773 |
| hls.js | identity" format SAMPLE-AES decryption of MPEG-2 TS segments only                   |


### HLS WebVTT

> ℹ️ **Priority: MUST** (*Currently supported by VHS*)

> ⚠️ **Note**
>
> HLS WeVTT is not encapsulated in fmp4 and timestamp sync is implemented via X-TIMESTAMP-MAP
> 
> > If content has in-manifest subtitles and in-segment subtitles (cea-608/708) - all will be emitted to the user.

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
> 
> If content has in-manifest subtitles and in-segment subtitles (cea-608/708) - all will be emitted to the user.

|        | URLS                                                                                |
|--------|-------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-3.1.5     |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/text/mp4_ttml_parser.js |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/utils/imsc1-ttml-parser.ts      |

### HLS Variable Substitution

> ℹ️ **Priority: MUST** (*Currently NOT supported by VHS*)

|        | URLS                                                                               |
|--------|------------------------------------------------------------------------------------|
| Spec   | https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis#section-4.3      |
| Shaka  | https://github.com/shaka-project/shaka-player/blob/main/lib/hls/hls_utils.js#L115  |
| hls.js | https://github.com/video-dev/hls.js/blob/master/src/utils/variable-substitution.ts |

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


## DRM

### Widevine

### Fairplay

### Fairplay (Legacy)

### PlayReady

### ClearKey

## Container Support

### MPEG-2 TS AAC (Audio Codec)
> ℹ️ **Priority: MUST** (*Currently supported by VHS*)
> 
> Transmux to MP4

### MPEG-2 TS AC3 (Audio Codec)
> ℹ️ **Priority: COULD** (*Currently NOT supported by VHS*)
>
> Transmux to MP4

### MPEG-2 TS EC3 (Audio Codec)
> ℹ️ **Priority: COULD** (*Currently NOT supported by VHS*)
>
> Transmux to MP4

### MPEG-2 TS MP3 (Audio Codec)
> ℹ️ **Priority: COULD** (*Currently NOT supported by VHS*)
>
> Transmux to MP4

### MPEG-2 TS Opus (Audio Codec)
> ℹ️ **Priority: COULD** (*Currently NOT supported by VHS*)
>
> Transmux to MP4

### MPEG-2 TS H264 (Video Codec)
> ℹ️ **Priority: MUST** (*Currently supported by VHS*)
>
> Transmux to MP4

### MPEG-2 TS H265 (Video Codec)
> ℹ️ **Priority: MUST** (*Currently NOT supported by VHS*)
>
> Transmux to MP4

### MPEG-2 TS ID3 (timed metadata)
> ℹ️ **Priority: MUST** (*Currently supported by VHS*)
>
> Should be emitted alongside with in-manifest timed metadata (if any)

### MPEG-2 TS CEA-608/CEA-708 (text track)
> ℹ️ **Priority: MUST** (*Currently supported by VHS*)
> 
> Should be emitted alongside with in-manifest text tracks (if any)

### MP4 CEA-608/CEA-708 (text-track)
> ℹ️ **Priority: MUST** (*Currently supported by VHS*)
>
> Should be emitted alongside with in-manifest text tracks (if any)

### MP4 IMSC (text-track)
> ℹ️ **Priority: MUST** (*Currently NOT supported by VHS*)
>
> Should be emitted alongside with in-manifest text tracks (if any)

### MP4 VTT (text-track)
> ℹ️ **Priority: MUST** (*Currently NOT supported by VHS*)
>
> Should be emitted alongside with in-manifest text tracks (if any)

### MP4 EMSG (timed-metadata)
> ℹ️ **Priority: MUST** (*Currently supported by VHS*)
>
> Should be emitted alongside with in-manifest timed-metadata (if any)

## Offline Viewing

> ℹ️ **Priority: COULD** (*Currently NOT supported by VHS*)
> 
> Including DRM support, see: https://www.w3.org/TR/encrypted-media/#dom-mediakeysession-load

|         | URLS                                                                        |
|---------|-----------------------------------------------------------------------------|
| Shaka   | https://github.com/shaka-project/shaka-player/tree/main/lib/offline         |
| dash.js | https://github.com/Dash-Industry-Forum/dash.js/tree/development/src/offline |
| hls.js  | Not Implemented                                                             |

## Text (outside in-manifest and in-segment tracks)

### External WebVTT (only for VoD)
> ℹ️ **Priority: SHOULD** (*Currently NOT supported by VHS directly*)

## Thumbnails (outside of HLS/DASH)

### External WebVTT with images/sprites (only for VoD)
> ℹ️ **Priority: SHOULD** (*Currently NOT supported by VHS directly*)

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
