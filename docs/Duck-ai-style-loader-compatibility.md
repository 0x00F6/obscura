# Duck.ai style-loader compatibility

## Reproduction and cause

On 2026-10-12, Duck.ai loaded its title and initial skeleton but did not mount
the chat controls with upstream revision `2bda71a`. The same failure occurred
with the separate `fix/dom-textarea-style-compat` branch at `a9a0760`.
The script diagnostic was:

```text
Couldn't find a style target. This probably means that the value for the 'insert' parameter is invalid.
```

The observed vendor bundle uses the generic style-loader insertion logic:
it selects `head`, then redirects to `contentDocument.head` only when the target
is an `HTMLIFrameElement`. Obscura exposed that constructor as an alias of
`Element`, so `document.head instanceof HTMLIFrameElement` incorrectly returned
`true`. The redirect failed and the application bootstrap stopped.

## Fix

Expose a distinct `HTMLIFrameElement` subclass and use it for HTML iframe
wrappers in parsed-node and known-name creation paths. Preserve the existing
inherited frame APIs. Ordinary HTML elements and foreign-namespace iframe
elements do not pass this interface check. No hostname-specific behavior,
verification bypass or remote-service change is involved.

## Offline verification

The runtime regressions cover parsed, newly created, namespaced and cloned
iframes, constructor/prototype identity, non-iframe elements, and a foreign
namespace. A second test reproduces the failing style insertion branch.
Both tests fail on the base revision and pass after the fix.

```bash
cargo nextest run --release --no-default-features -p obscura-js \
  -E 'test(html_iframe_element_class_is_tag_specific) | test(style_loader_targets_head_without_frame_redirection)'
```

`render-repros/iframe-style-target.html` provides a deterministic visual fixture.
Successful style insertion is distinct from a successful Duck.ai conversation;
live service behavior and any subsequent failures must be reported separately.

## Executed validation

The complete release suite with rendering passed: 2,156 tests and five skips.
The two focused regressions also passed without rendering in release and debug.
All four CLI feature configurations built successfully, and the offline
obstacle course passed 33/33 stages. The reduced visual fixture reported
successful style insertion in both Obscura and Chrome.

A fresh real Duck.ai session with the corrected render/stealth build mounted
the textarea and buttons in 2.664 seconds, reported `headIsIframe = false`, and
recorded no page error. This fixes the original startup exception. A subsequent
input test still left the application-controlled Ask button disabled; no chat
POST or assistant answer was observed. Complete conversation compatibility is
not established by the startup fix.
A separate release test through the embedded Rust `Browser` API, without CDP,
also mounted and filled the textarea but observed the Ask control disabled after
ten seconds of explicit `settle` calls. This remaining behavior is not resolved
by the iframe interface correction. No disabled control was overridden and no
assistant response is claimed.
