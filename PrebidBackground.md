# Prebid Background

## Birth of Header Bidding

As DFP (now GAM) became the dominant ad server (and included tight AdX integration) publishers wanted a way to have other demand sources compete in the GAM/AdX ad selection process. While GAM/AdX didn't provide an API to do this, publishers did have the ability create their own line items in DFP that were considered in the selection process. Futhermore, they could activate these line items based on key/value pairs added to the DFP tag invocation. The final step was figuring out how to solicit an independent SSP and map their bid to the right key/value. Soliciting the SSP in the header of the page allowed the bid to be retrieved and mapped to a key/value before the DFP ad slot was fetched. And thus header bidding was born.

## Birth of Prebid.js

Publishers saw a significant benefit from having a single SSP compete with DFP/AdX and the next step was to have multiple SSPs compete. While SSPs provided libraries that made it easy to do header bidding, they all had their own implementation and they weren't designed to work well together. Furthermore, there were different strategies for creating line items, mapping K/V pairs to line items, naming line items, and the creative that triggered the serving of a cached SSP creative. Very sophisticated publishers were capable of writing their own javascript solutions that managed these complexities, as well as paying for AdOps folks to manage the setup in DFP, but it was error prone and expensive. Prebid.js provided a standardized way of integrating with DFP, the publisher's page, and SSPs (who provided adapters that mapped their ad calls and bids to the Prebid standard).

## Side Note: Why Multiple SSPs?

Since most SSPs are integrated with the same demand sources, one might ask what benefit there is to integrating with multiple SSPs. The drawbacks are management overhead (for the pubs), duplicated requests (for the DSPs), and extra browser load/latency (for the end users). But publishers did see a significant benefit from having multiple SSPs and there's a good reason for that.

In a perfectly efficient exchange all demand would be submitted to a single auction where a global decision would be made. Unfortunately, most exchanges (ad or otherwise) are far from perfectly efficient. One of the big inefficiencies in programmatic ad exchanges is the way DSPs and SSPs respond with a single bid to a given request. While DSPs are representing many different campaigns, when responding to a bid request from an SSP they run an internal selection process and respond with a single bid on behalf of one of these campaigns. The SSPs receive single bids from a number of different DSPs, then select a winner and respond with that to the next level of auction (the prebid auction). The result is that at each tier of the auction process demand gets lost, which leads to inefficient markets.

Soliciting multiple SSPs had positive results because it exposed more bids. While it's possible that the DSPs are responding with the exact same bids to every SSP, it's highly unlikely. The increased bid density provided a better financial outcome for publishers.

There are also other reasons that not all SSPs represent the same DSP demand. Some of those are:
* Deals/PMPs may be transacted through a single SSP
* DSPs may have negotiated different rates with certain SSPs
* SSPs may not all handle the same type of inventory

There are many reasons why a Publisher is best served by working with multiple SSPs.

### Side-side Note: First Price Auction (1PA) vs. Second Price Auction (2PA)

At the advent of programmatic advertising the auctions all used a second price strategy. More recently the industry has transitioned to a first price auction strategy. I believe there are two reasons for this:
1. When multiple layers of auction are happening, if the first layer is a second price auction you can't reason about how your bid will perform in the final layer.

A first price auction means that the bid value essentially stays the same at all selection layers (minus fees) which makes it easier to reason about how your bid will perform regardless of what layer of auction/selection it is competing in.

2) Second price auctions are only truly effective when bid density is high enough.

With DSPs submitting a single bid from all their demand, the density is often not high enough to get a sense of true value, which means pubs must resort to different flooring tactics to avoid being cherry picked.

If the industry wants to return to a second price auction dynamic, then DSPs will need to submit more bids to SSPs, and SSPs may also need to submit more bids to the final decision layer. This will insure that the bid density is high enough for a true value to be determined.


## Why Prebid.js Works

### …for Publishers

Publishers (and adtech in general) must contend with the fact that ad selection happens at multiple levels. DSPs perform ad selection from among all eligible campaigns they represent. SSPs perform ad selection from all DSPs that responded to a bid request. A publisher does selection from among all SSPs that responded to it. Prebid.js works because SSPs are able to provide publishers with a realtime signal of what an ad slot is worth.

However, GAM does not provide this information to a publisher in realtime - it merely runs an auction and returns the winning ad. Due to this lack of price signal GAM must necessarily be the final level of ad selection.

*Header bidding/Prebid.js has been extremely successful because it provides a way to funnel SSP price signals into the final selection layer.* It enabled publishers to retain control of their business models without being beholden to a single technology vendor (to some extent).

**Prebid.js works for publishers because it enables greater competition and avoids vendor lock-in.**

### …for SSPs

Any entity in competition with another entity wants to know that the playing field is level - it does not unfairly advantage a competitor. Prebid.js provides this assurance to SSPs in several ways:

First, the code that runs the on-page auction is open-source and also running in the open simply by virtue of the way javascript on the browser works (obfuscation not withstanding). This is a very high level of transparency.

Second, SSPs can develop their own adapters. The part of Prebid.js that is SSP specific is under the control of the SSP itself (though still open-source). SSPs can develop their own adapters to function optimally in the Prebid.js world.

Third, Prebid.js is run by an independent organization that has representation from various SSPs. This provides assurance that participants will be held accountable for their actions.

**In short, Prebid.js works for SSPs because there is transparency, control, and accountability.**

## Prebid.js Today

This sequence diagram attempts to illustrate how Prebid.js functions on a publisher's page today:

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

While it is a technical detail to show the invocation of the `prebid_[SSP_MODULE].js` in the sequence diagram above, it's there to illustrate the principle of SSP involvement and code control in the context of the greater auction that occurs.

Hopefully this diagram helps show how a Publisher using Prebid.js is able to integrate other SSPs into GAM.