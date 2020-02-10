# Opaque Response Blocking (ORB, aka CORB++)

## Objective

To block as many opaque responses as possible while remaining web compatible.

## High-level idea

CSS, JavaScript, and media (audio, images, video) can be requested across origins without CORS. Except for CSS there is no MIME type enforcement. Ideally we still block as many responses as possible that are not one of these types to avoid leaking their contents through side channels.

## Processing model

A user agent has a **opaque-safelisted requesters set**. (This should be scoped similar to other network caches.)

A request has an associated **opaque media identifier** (null or an opaque identifier). Null unless explicitly stated otherwise.

\[The idea here is that the opaque media identifier is owned by the media element (audio/video only; I'm assuming we won't do range for images without at least requiring MIME types at this point). As part of the element being GC'd, it would send a message to get all the relevant entries from the user agent's opaque-safelisted requesters set removed. There might be better strategies available here and it's not clear to me to what extent we need to specify this, but it's probably good to have a model that does not leak memory forever so the set needs to be keyed to something. The fetch group might also be reasonable.]

To determine whether to allow response _response_ to a request _request_, run these steps:

1. Let _mimeType_ be the result of extracting a MIME type from _response_'s header list.
1. If _mimeType_ is failure, then return true.
1. If _mimeType_ is a JavaScript MIME type, then return true.
1. If _mimeType_'s essence is `text/css`, then return true.
1. If _mimeType_'s essence is `image/svg+xml`, then return true.
1. If the user agent's opaque-safelisted requesters set contains (_request_'s opaque media identifier, _request_'s current URL), then return true.
1. Wait for 1024 bytes of _response_ or end-of-file, whichever comes first and let _bytes_ be those bytes.
1. If the [image type pattern matching algorithm](https://mimesniff.spec.whatwg.org/#image-type-pattern-matching-algorithm) given _bytes_ does not return undefined, then return true.
1. If the [audio or video type pattern matching algorithm](https://mimesniff.spec.whatwg.org/#audio-or-video-type-pattern-matching-algorithm) given _bytes_ does not return undefined, then:
   1. Append (_request_'s opaque media identifier, _request_'s current URL) to the user agent's opaque-safelisted requesters set.
   1. Return true.
1. If determine nosniff given _response_'s header list returns true, then return false.
1. If _response_'s body parses as JavaScript and does not parse as JSON, then return true.
1. Return false.

Note: responses for which the above algorithm returns true and contain secrets are strongly encouraged to be protected using `Cross-Origin-Resource-Policy`.

## Findings

* It's unfortunate `X-Content-Type-Options` kicks in after media sniffing, but it was not web compatible for Firefox to enforce it for images back in the day.
* Due to way [style sheet fetching works](https://github.com/whatwg/fetch/issues/964) we cannot protect responses without an extractable MIME type.
