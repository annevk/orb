# Opaque Response Blocking (ORB, aka CORB++)

## Objective

To block as many opaque responses as possible while remaining web compatible.

## High-level idea

CSS, JavaScript, and media (audio, images, video) can be requested across origins without CORS. Except for CSS there is no MIME type enforcement. Ideally we still block as many responses as possible that are not one of these types to avoid leaking their contents through side channels.

## Processing model

An **opaque-safelisted MIME type** is a [JavaScript MIME type](https://mimesniff.spec.whatwg.org/#javascript-mime-type) or a MIME type whose essence is "`text/css`" or "`image/svg+xml`".

An **opaque-blocklisted MIME type** is an [HTML MIME type](https://mimesniff.spec.whatwg.org/#html-mime-type), [JSON MIME type](https://mimesniff.spec.whatwg.org/#json-mime-type), or [XML MIME type](https://mimesniff.spec.whatwg.org/#xml-mime-type).

An **opaque-blocklisted-never-sniffed MIME type** is a MIME type whose essence is
"`application/gzip`",
"`application/msexcel`",
"`application/mspowerpoint`",
"`application/msword`",
"`application/msword-template`",
"`application/pdf`",
"`application/vnd.ces-quickpoint`",
"`application/vnd.ces-quicksheet`",
"`application/vnd.ces-quickword`",
"`application/vnd.ms-excel`",
"`application/vnd.ms-excel.sheet.macroenabled.12`",
"`application/vnd.ms-powerpoint`",
"`application/vnd.ms-powerpoint.presentation.macroenabled.12`",
"`application/vnd.ms-word`",
"`application/vnd.ms-word.document.12`",
"`application/vnd.ms-word.document.macroenabled.12`",
"`application/vnd.msword`",
"`application/vnd.openxmlformats-officedocument.presentationml.presentation`",
"`application/vnd.openxmlformats-officedocument.presentationml.template`",
"`application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`",
"`application/vnd.openxmlformats-officedocument.spreadsheetml.template`",
"`application/vnd.openxmlformats-officedocument.wordprocessingml.document`",
"`application/vnd.openxmlformats-officedocument.wordprocessingml.template`",
"`application/vnd.presentation-openxml`",
"`application/vnd.presentation-openxmlm`",
"`application/vnd.spreadsheet-openxml`",
"`application/vnd.wordprocessing-openxml`",
"`application/x-gzip`",
"`application/x-protobuf`",
"`application/zip`",
"`multipart/byteranges`"
"`multipart/signed`",
"`text/event-stream`", or
"`text/csv`".

A user agent has an **opaque-safelisted requesters set**. (This should be scoped similar to other network caches.)

A request has an associated **opaque media identifier** (null or an opaque identifier). Null unless explicitly stated otherwise.

\[The idea here is that the opaque media identifier is owned by the media element (audio/video only; I'm assuming we won't do range requests for images without at least requiring MIME types at this point). As part of the element being GC'd, it would send a message to get all the relevant entries from the user agent's opaque-safelisted requesters set removed. There might be better strategies available here and it's not clear to me to what extent we need to specify this, but it's probably good to have a model that does not leak memory forever so the set needs to be keyed to something. The fetch group might also be reasonable.]

To determine whether to allow response _response_ to a request _request_, run these steps:

1. Let _mimeType_ be the result of [extracting a MIME type](https://fetch.spec.whatwg.org/#concept-header-extract-mime-type) from _response_'s header list.
1. Let _nosniff_ be the result of [determining nosniff](https://fetch.spec.whatwg.org/#determine-nosniff) given _response_'s header list.
1. If _mimeType_ is not failure, then:
   1. If _mimeType_ is an opaque-safelisted MIME type, then return true.
   1. If _mimeType_ is an opaque-blocklisted-never-sniffed MIME type, then return false.
   1. If _response_'s status is `206` and _mimeType_ is an opaque-blocklisted MIME type, then return false. TODO: is this needed with the requesters set?
   1. If _nosniff_ is true and _mimeType_ is an opaque-blocklisted MIME type or its essence is "`text/plain`", then return false.
1. If the user agent's opaque-safelisted requesters set contains (_request_'s opaque media identifier, _request_'s current URL), then return true.
1. Wait for 1024 bytes of _response_ or end-of-file, whichever comes first and let _bytes_ be those bytes.
1. If the [image type pattern matching algorithm](https://mimesniff.spec.whatwg.org/#image-type-pattern-matching-algorithm) given _bytes_ does not return undefined, then return true.
1. If the [audio or video type pattern matching algorithm](https://mimesniff.spec.whatwg.org/#audio-or-video-type-pattern-matching-algorithm) given _bytes_ does not return undefined, then:
   1. Append (_request_'s opaque media identifier, _request_'s current URL) to the user agent's opaque-safelisted requesters set.
   1. Return true.
1. If _nosniff_ is true, then return false.
1. If _response_'s status is not an [ok status](https://fetch.spec.whatwg.org/#ok-status), then return false.
1. If _mimeType_ is failure, then return true.
1. If _mimeType_'s essence starts with "`audio/`", "`image/`", or "`video/`", then return false.
1. If _response_'s body parses as JavaScript and does not parse as JSON, then return true.
1. Return false.

Note: responses for which the above algorithm returns true and contain secrets are strongly encouraged to be protected using `Cross-Origin-Resource-Policy`.

## Findings

* It's unfortunate `X-Content-Type-Options` mostly kicks in after media sniffing, but it was not web compatible for Firefox to enforce it for images back in the day. (`X-Content-Type-Options` enforcement for specific callers, such as style sheets and scripts will remain and complement the above algorithm.)
* Due to the way [style sheet fetching works](https://github.com/whatwg/fetch/issues/964) we cannot protect responses without an extractable MIME type.

## Acknowledgments

Many thanks to Jake Archibald, Lukasz Anforowicz, Nathan Froyd, and those involved in Chromium's CORB project.
