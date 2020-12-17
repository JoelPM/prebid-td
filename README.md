# prebid-td
An exploration of the role Prebid can play in a TurtleDove world.

# Birth of Header Bidding
As DFP (now GAM) became the dominant ad server (with tight AdX integration) publishers wanted a way to have other demand sources compete in the GAM/AdX ad selection process. While GAM/AdX didn't provide an API to do this, publishers did have the ability create their own line items in DFP that were considered in the selection process. Futhermore, they could activate these line items based on key/value pairs added to the DFP tag invocation. The final step was figuring out how to solicit an independent SSP and map their bid to the right key/value. Soliciting the SSP in the header of the page allowed the bid to be retrieved and mapped to a key/value before the DFP ad slot was fetched. And thus header bidding was born.

# Birth of Prebid.js
Publishers saw a significant benefit from having a single SSP compete with DFP/AdX and the logical next step was to have multiple SSPs compete. While SSPs provided libraries that made it easy to do header bidding, they all had their own implementation and they weren't designed to work well together. Furthermore, there were different strategies for creating line items, mapping k/v pairs to line items, naming line items, and the creative that triggered the serving of a cached SSP creative. Very sophisticated publishers were capable of writing their own javascript solutions that managed these complexities, as well as paying for AdOps folks to manage the setup in DFP, but it was error prone and expensive. Prebid.js provided a standardized way of integrating with DFP, the publisher's page, and SSPs, who provided adapters that mapped their ad calls and bids to the Prebid standard.

# Why Prebid.js works
## ... for Publishers
Publishers (and adtech in general) must contend with the fact that ad selection happens at multiple levels. DSPs perform ad selection from among all eligible campaigns they represent. SSPs perform ad selection from all DSPs that responded to a bid request. A publisher does selection from among all SSPs that responded to it. Prebid.js works because SSPs are able to provide publishers with a realtime signal of what an ad slot is worth.

However, GAM does not provide this information to a publisher in realtime - it merely runs an auction and returns the winning ad. Due to this lack of price signal GAM must necessarily be the final level of ad selection.

*Header bidding/Prebid.js has been extremely successful because it provides a way to funnel SSP price signals into the final selection layer.* It enabled pubilshers to retain control of their business models without being beholden to a single technology vendor (to some extent).

**Prebid.js works for publishers because it enables greater competition and avoids vendor lock-in.**

## ... for SSPs
Any entity in competition with another entity wants to know that the playing field is level - it does not unfairly advantage a competitor. Prebid.js provides this assurance to SSPs in several ways:

First, the code that runs the on-page auction is open-source and also running in the open simply by virtue of the way javascript on the browser works (obfuscation not withstanding). This is a very high level of transparency.

Second, SSPs can develop their own adapters. The part of Prebid.js that is SSP specific is under the control of the SSP itself (though still open-source). SSPs can develop their own adapters to function optimally in the Prebid.js world.

Third, Prebid.js is run by an independent organization that has representation from various SSPs. While not a technology solution, it provides assurance that participants will be held accountable for their actions.

In short, **Prebid.js works for SSPs because there is transparency, control, and accountability.**

# Prebid.js Today
This sequence diagram attempts to illustrate how Prebid.js functions today.

![Prebid Sequence Diagram](out/prebid_statusquo/prebid_statusquo.png)

1. A user navigates to a publisher site
2. The publisher page loads and configures prebid.js
3. prebid.js invokes SSP adapter for participating SSPs
4. SSP adapter issues an ad request to the SSP
5. SSPs do traffic quality evaluation (and may short circuit if they deem the traffic fraudulent).
6. SSPs solicit bids from eligible and interested DSPs
7. DSPs respond with bids
8. SSPs filter bids based on pub controls (e.g. ad quality)
9. SSPs responds to the page/SSP adapter with a creative and a value.
10. SSP adapter responds to Prebid.js with the SSPs creative and a value.
11. Prebid.js selects the appropriate ad (may be auction, may not be)
12. Prebid.js configures the appropriate k/v pairs for the selected ad
13. Prebid.js returns control the page, where the pub initiates the loading of the GAM slot
14. GAM ad tag is invoked.
15. GAM ad tag calls GAM (and AdX, not OB due to their policies)
16. GAM runs the final selection process, which includes line items activated by Prebid.js
17. GAM creative is rendered regardless of outcome, but what is in that creative will vary:
  - [GAM/AdX winner] In this situation the rendered ad tag is a creative sent via GAM/AdX.
  - [Prebid.js winner] In this case the tag rendered from GAM was associated with a Prebid.js lineitem and it calls back into Prebid.js to render the earlier selected creative.

While it is a technical detail to show the invocation of the `prebid_[SSP_MODULE].js` in the sequence diagram above, I felt it worth making explicit as it illustrates one of the principles noted above.

Hopefully this diagram helps show how a Publisher using Prebid.js is able to integrate other SSPs into GAM which must be the final decision layer due to the reasons noted above (and others).

# TurtleDove Impact on Ad Selection Process
TurtleDove proposes a number of changes to how ad tech will function, but this document is only focused on the changes to the ad selection process.

## TurtleDove as Final Selection Layer
Today GAM insists on being the final ad selection layer. In a TurtleDove world Chrome insists on being the final ad selection layer. Like GAM, TurtleDove does not provide a price signal externally - only a binary yes/no on whether something rendered. (Aside: those that have been in adtech for a long time will remember the early ad networks and the proliferation of fallback tags and the ensuing fallback chains that were an attempt to optimize binary demand sources... and most likely you experienced an involuntary shudder.)

If GAM must cede final selection to TurtleDove it raises some questions:
* will they do so in a way that makes it easy for publishers to integrate other SSPs (by providing an API)?
* will they do so in a way that makes it harder for other SSPs to integrate (by calling navigator.renderInterestGroupAd directly from their tag without allowing the publisher to customize the arguments)?


## SSP No Longers Knows Value in Realtime
Today SSPs able to provide publishers with a realtime valuation of an ad slot because they have contextual and user information in one request and are able to respond with a single *best* value in response to that request. In a TurtleDove world SSPs will only know the contextual value of the ad slot in realtime. The user value of an ad slot will be stored in the browser (if an IG ad request has been issued) and will never be known in realtime by the SSP, publisher, or any entity that's not behind the curtain of TurtleDove.

While the TurtleDove author has said that they desire TurtleDove execute a local auction over SSP demand ([#73](https://github.com/WICG/turtledove/issues/73)), you can't do this if you're unwilling to provide a price signal back to the SSP.

A simple implementation of this would be something like:
```javascript
// This can be thought of as an OpenRTB Bid Response
let sspContextualOrtbBidResp = {
    "adm": "...creative html...",
    "price": 1.5,
    ...
}

 // This could be thought of as an OpenRTR Bid Request
let sspOrtbBidReq = {
    "site": {
        "ref": "https://reverb.com/",
        "publisher": {
            ...
        }
    }
    ...
}

// This could be thought of as an OpenRTB Bid Request to an SSPs bid cache on the browser
let sspRealtimeValue = navigator.resolveInterestGroupAd(sspContextualOrtbBidResp, sspOrtbBidReq)
```

At this point the SSP would know the value they provide to the publisher in realtime and be able to wrap this logic in their Prebid.js adapter and Prebid.js would function as it does today.

However, TurtleDove views an IG based bid price to be a privacy attach vector so this approach isn't feasible. If this approach isn't feasible, I don't think it's possible to do an SSP local decision in TurtleDove.