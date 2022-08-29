# Opaque Response Blocking (ORB, aka CORB++)

## Status

This repository is being upstreamed to the Fetch Standard through [PR #1442](https://github.com/whatwg/fetch/pull/1442). As indicated there you can [preview a version of the Fetch Standard with the new text integrated](https://whatpr.org/fetch/1442.html).

The PR is primarily blocked on resolving the [mvp issues](https://github.com/annevk/orb/labels/mvp). Help appreciated!

The PR is in a more advanced state than the text below and should be the starting point for implementers and reviewers.

## Objective

To block as many opaque responses as possible while remaining web compatible.

## High-level idea

CSS, JavaScript, images, and media (audio and video) can be requested across origins without CORS. Except for CSS there is no MIME type enforcement. Ideally we still block as many responses as possible that are not one of these types to avoid leaking their contents through side channels.

## Processing model

### New MIME type sets

An **opaque-safelisted MIME type** is a [JavaScript MIME type](https://mimesniff.spec.whatwg.org/#javascript-mime-type)
or a MIME type whose essence starts with "`audio/`", "`image/`", or "`video/`"
or a MIME type whose essence is "`application/dash+xml`", "`application/ogg`", "`application/vnd.apple.mpegurl`", "`text/css`", or "`text/vtt`", .

An **opaque-blocklisted MIME type** is an [HTML MIME type](https://mimesniff.spec.whatwg.org/#html-mime-type), [JSON MIME type](https://mimesniff.spec.whatwg.org/#json-mime-type), or [XML MIME type](https://mimesniff.spec.whatwg.org/#xml-mime-type).

An **opaque-blocklisted-never-sniffed MIME type** is a MIME type whose essence is one of

* "`application/dash+xml`"
* "`application/gzip`"
* "`application/msexcel`"
* "`application/mspowerpoint`"
* "`application/msword`"
* "`application/msword-template`"
* "`application/pdf`"
* "`application/vnd.apple.mpegurl`"
* "`application/vnd.ces-quickpoint`"
* "`application/vnd.ces-quicksheet`"
* "`application/vnd.ces-quickword`"
* "`application/vnd.ms-excel`"
* "`application/vnd.ms-excel.sheet.macroenabled.12`"
* "`application/vnd.ms-powerpoint`"
* "`application/vnd.ms-powerpoint.presentation.macroenabled.12`"
* "`application/vnd.ms-word`"
* "`application/vnd.ms-word.document.12`"
* "`application/vnd.ms-word.document.macroenabled.12`"
* "`application/vnd.msword`"
* "`application/vnd.openxmlformats-officedocument.presentationml.presentation`"
* "`application/vnd.openxmlformats-officedocument.presentationml.template`"
* "`application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`"
* "`application/vnd.openxmlformats-officedocument.spreadsheetml.template`"
* "`application/vnd.openxmlformats-officedocument.wordprocessingml.document`"
* "`application/vnd.openxmlformats-officedocument.wordprocessingml.template`"
* "`application/vnd.presentation-openxml`"
* "`application/vnd.presentation-openxmlm`"
* "`application/vnd.spreadsheet-openxml`"
* "`application/vnd.wordprocessing-openxml`"
* "`application/x-gzip`"
* "`application/x-protobuf`"
* "`application/x-protobuffer`"
* "`application/zip`"
* "`audio/mpegurl`"
* "`multipart/byteranges`"
* "`multipart/signed`"
* "`text/event-stream`"
* "`text/csv`"
* "`text/vtt`"

### Changes to requests and media elements

A request has an associated **no-cors media request state** ("N/A", "initial", or "subsequent"). It is "N/A" unless explicitly stated otherwise.

We adjust the way media element fetching is done to more clearly separate between the initial and any subsequent range fetches:

* For its initial range request a media element sets request's no-cors media request state to "initial" and it follows redirects. That yields (after any redirects) an initial response.
* For its subsequent range requests the URL of the initial response is used as request's URL, it no longer follows redirects, and request's no-cors media request state is set to "subsequent". Note: redirects here resulted in an error in Chrome until recently. We could somewhat easily allow same-origin redirects by adjusting the check performed against request's no-cors media request state, but it's not clear that's desirable.

(These changes are not needed when CORS is used, but it might make sense to align these somewhat, to the extent they are not already.)

### ORB's algorithm

To determine whether to allow response _response_ to a request _request_, run these steps:

1. Let _mimeType_ be the result of [extracting a MIME type](https://fetch.spec.whatwg.org/#concept-header-extract-mime-type) from _response_'s header list.
1. Let _nosniff_ be the result of [determining nosniff](https://fetch.spec.whatwg.org/#determine-nosniff) given _response_'s header list.
1. If _mimeType_ is not failure, then:
   1. If _mimeType_ is an opaque-safelisted MIME type, then return true.
   1. If _mimeType_ is an opaque-blocklisted-never-sniffed MIME type, then return false.
   1. If _response_'s status is 206 and _mimeType_ is an opaque-blocklisted MIME type, then return false.
   1. If _nosniff_ is true and _mimeType_ is an opaque-blocklisted MIME type or its essence is "`text/plain`", then return false.
1. If _request_'s no-cors media request state is "subsequent", then return true.
1. If _response_'s status is 206 and [validate a partial response](https://wicg.github.io/background-fetch/#validate-a-partial-response) given 0 and _response_ returns invalid, then return false.
1. Wait for 1024 bytes of _response_ or end-of-file, whichever comes first and let _bytes_ be those bytes.
1. If the [audio or video type pattern matching algorithm](https://mimesniff.spec.whatwg.org/#audio-or-video-type-pattern-matching-algorithm) given _bytes_ does not return undefined, then:
   1. If _requests_'s no-cors media request state is not "initial", then return false.
   1. If _response_'s status is not 200 or 206, then return false.
   1. Return true.
1. If _requests_'s no-cors media request state is not "N/A", then return false.
1. If the [image type pattern matching algorithm](https://mimesniff.spec.whatwg.org/#image-type-pattern-matching-algorithm) given _bytes_ does not return undefined, then return true.
1. If _nosniff_ is true, then return false.
1. If _response_'s status is not an [ok status](https://fetch.spec.whatwg.org/#ok-status), then return false.
1. If _mimeType_ is failure, then return true.
1. Wait for end-of-file of _response_'s body. Note: as discussed in [GitHub's annevk/orb #22](https://github.com/annevk/orb/issues/22) partially parsing JavaScript is unfortunately infeasible. This might end up leaking the size of responses that hit this step.
1. If _response_'s body parses as JavaScript and does not parse as JSON, then return true.
1. Return false.

Note: responses for which the above algorithm returns true and contain secrets are strongly encouraged to be protected using `Cross-Origin-Resource-Policy`.

## Implementation considerations

Setting request's no-cors media request state to "subsequent" ideally happens in a process that is not easily compromised, because such a spoofed value can be used to bypass ORB. In particular, "subsequent" is only to be allowed and used if a trustworthy process can verify that the media element (or its node document, or its node document's origin) has previously received a response with the same URL that has sniffed as audio or video.

## Findings

* It's unfortunate `X-Content-Type-Options` mostly kicks in after image/media sniffing, but it was not web compatible for Firefox to enforce it for images back in the day.
* Due to the way [style sheet fetching works](https://github.com/whatwg/fetch/issues/964) we cannot protect responses without an extractable MIME type.
* Media elements always make range requests.

## Acknowledgments

Many thanks to Jake Archibald, Lukasz Anforowicz, Nathan Froyd, and those involved in Chromium's CORB project.
