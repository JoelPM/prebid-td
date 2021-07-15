# Prebid Fledge

An exploration of the role Prebid can play in a Fledge world.

## Fledge and Ad Selection

While Fledge proposes a number of changes to how ad tech will function, this document is focused on the changes to the ad selection process in the context of the browser.

### Chrome/Fledge Becomes the Final Selection Layer

Today GAM is the final ad selection layer because it contains the publisher's direct demand, is tightly integrated with AdX, and doesn't provide a price signal externally. However, in a Fledge world, Chrome will contain interest group demand that isn't accessible via any other means and won't provide a price signal externally. When ```navigator.runAdAuction``` returns 'true' it has identified an appropriate ad for rendering based on provided bids and scoring logic. As a result, the browser/Fledge must become the final selection layer.

SSPs are used to participating in a final selection hosted elsewhere but it raises the question of how GAM will adjust to this.

### SSP Logic Must Span Chrome/Fledge

Today SSPs are able to provide publishers with a realtime valuation of an ad slot because they have contextual and user information in one request and are able to respond with a single *best* value in response to that request.

The process of selecting a best value involves the following steps:
1) Traffic quality (TQ) - is this a legitimate request from a real user?
2) DSP qualification - which DSPs are interested in this request?
2) Bid solicitation - which DSPs will place a bid on this request?
3) Ad Quality (AQ) - which bids are eligible given the publisher's requirements?
4) Valuation - which bid is most valuable to the publisher at this moment?

This process must now be split across the SSPs and the browser (for IG bids), which means that an SSP will no longer know what an ad slot is worth in realtime (nor will the publisher).

## Prebid.js in a Fledge World

For the contextual ad selection process, Prebid.js can continue to function in the same manner as today. Here's what that looks like:

![Prebid.js Sequence Diagram](out/prebid_td_context/prebid_td_context.png)

However, the browser will also contain bids that are targeted at Interest Groups. These bids cannot escape the Fledge sandbox, which means that logic to choose between them must be sent into Fledge.

There are two parts to the logic that must be sent into TurtleDove:
1) Prebid IG controller - the script that runs the overall IG bid selection process
2) Prebig IG SSP adapters - the SSP implemented functionality to choose from among an SSP's IG bids

### Prebid IG SSP Adapter

In the same way that an SSP implements an adapter to integrate with Prebid.js, an SSP will need to implement an adapter that selects from the IG based bids that it owns. This entails implementing some of the logic noted above:

* Bid targeting - which bids are eligible to participate based on advertiser targeting?
* Context targeting - which bids are eligible to participate based on pub restrictions?
* Valuation - does the current context influence the value of these bids? Are there Deals or Private Marketplace rules in effect?

In order to do this selection process the SSP adapter needs to have the contextual information passed in, including the publisher's restrictions and any active business models. The IG bids also need to contain any advertiser restrictions so that those can be evaluated given the current context. In the past all this information stayed within the SSP but for ad selection to be completed in Fledge the information must be sent to the browser.

### Prebid IG Controller

The Prebid IG controller must receive the winning contextual bid (already chosen by Prebid.js) and all the SSP specific metadata. The controller passes the SSP metadata to the SSP IG adapter and receives a bid (or none) from the SSP adapter. The IG Controller then compares the results from all the SSP IG Adapters and chooses a winner. Finally, the IG Controller compares the contextual winner to the IG winner. If an IG winner exists and is higher than the contextual winner, the result of ```navigator.runAdAuction``` is not ```false``` Prebid.js will render the IG ad by passing the returned opaque object to a fenced frame. If the contextual winner has the higher value then ```false``` is returned and the overall Prebid.js script renders the contextual ad.

### Prebid IG Flow

Here's what that looks like in a flow diagram:

![Prebid-td.js Sequence Diagram](out/prebid_td_interest/prebid_td_interest.png)

1. Pub invokes final selection process in Prebid.js
2. Prebid selects/looks up contextual winner
3. Prebid constructs uber-metadata object (each SSP needs the contextual info and pub restrictions, etc)
4. Prebid invokes ```navigator.runAdAuction``` with winning contextual bid, uber-metadata object, and bundled Prebid IG Controller & SSP Adapters
5. Chrome/Fledge evaluates all interest groups stored client-side calling DSP provided ```generateBid``` functions
6. DSP bid functions produce bids using a combination of IG and contextual data from the auction configuration
7. Chrome/Fledge passes returned bids one at a time to Prebid IG Controller ```scoreBid``` function
8. Prebid IG Controller invokes SSP IG adapters and passes in SSP contextual metadata and bid from DSP
9. SSP IG Adapter does bid targeting, context targeting, validation (SSP logic)
10. SSP IG Adapter passes back scored SSP IG ad/bid in **standardized** prebid scoring format @TODO how does prebid choose between different SSPs scores?
11. Prebid IG Controller selects highest scoring IG ad from among those returned by SSP IG Adapters
12. Prebid IG Controller compares contextual ad to IG ad
13. Prebid IG Controller returned IG ad score or ```0``` if contextual ad is preferred
14. Chrome/Fledge sorts scored ads and chooses highest score (> 0) as winner
15. Chrome/Fledge returns opaque object for rendering if winner, or returns false (if context is winner)
16. IG Bid Winner
    1. ```navigator.runAdAuction``` returns opaque object
    2. Prebid record IG win
    3. Prebid passes opaque object to a fenced frame for rendering
17. Context Bid Winner
    1. ```navigator.runAdAuction``` returns false
    2. Prebid renders contextual ad
    3. Prebid records contextual win
18. Ad has rendered

As can be seen in the sequence diagram above, the Fledge API is invoked by Prebid which is also responsible for contstructing the arguments.

### Winning Bid

This could be as simple as a numeric value or it could be as rich as an OpenRTB bid response object. I think it depends on how complex the Prebid IG Controller is and what info it needs.

```javascript
var prebidWinningConextualBid = {
    // possibly an OpenRTB Bid Response object, though doesn't have to be
}
```

### Auction Configuration

Fledge offers a flexible configuration object which can be used to pass arbitrary information to the buyers bidding function (perBuyerSignals), the sellers scoring function (sellerSignals), or both (auctionSignals).

Ideally this would be represented as an OpenRTB bid request that is supplemented with SSP specific metadata where needed. An OpenRTB bid request is a fairly standard way of capturing contextual information that many SSPs are familiar with and is reasonably concise. It's also consistent with how many of the SSPs probably received the contextual ad request.

```javascript
const myAuctionConfig = {
  'seller': 'publisher.com',
  'decisionLogicUrl': '/publishersPrebidIGBundle.js', // link to publisher's prebidIG bundle
  'trustedScoringSignalsUrl': 'publishers-prebid-server.com/trusted-scoring-signals/',
  'interestGroupBuyers': ['www.example-dsp.com', 'buyer2.com', ...],
  'additionalBids': [ prebidWinningContextualBid ],
  'auctionSignals': ortbBidRequest,   // captures contextual information
  'sellerSignals': {
    sspMetadata: {  // metadata provided by contextual adapters
      "magnite": {...},
      "openx": {...},
      "pubmatic": {...}
    }
  },
  'perBuyerSignals': {
    'www.example-dsp.com': {...},
    'www.another-buyer.com': {...},
    ...},
};
```

The invocation would then look like:

```javascript
navigator.runAdAuction(myAuctionConfig).then((auctionResult) => {
  if (auctionResult) {
    // IG ad rendered
  } else {
    // Render the contextual ad
  }
})
```

## Open Questions

1. Is there a permissions model for access to IG Bids? What is it?
An SSP specific TD API seems impossible (see below). The browser running the auction seems bad (see Parrrot). Having bids available for rendering by anyone seems problematic as well. This proposal assumes that anyone who invokes renderInterestGroupAd has access to all bids. I suppose SSPs could encrypt their responses and then pass a key into the metadata object that would allow only them to decrypt it. This actually seems like it might be the best way, though passing an encryption key through the wild is only mildly secure so it's not real security.

2. How will GAM interact with the TD API?
How will GAM integrate into the final selection layer in the browser and will it help or hinder SSPs? If GAM were to provide an an API/adapter that integrated with Prebid.js this would be a good thing. If, on the other hand, GAM provides a tag library that directly invokes ```navigator.renderInterestGroupAd``` without allowing other SSPs to participate this would be bad.

## An alternative idea that won't work:

While the TurtleDove author has stated the goal is that TurtleDove execute a local auction over SSP demand ([#73](https://github.com/WICG/turtledove/issues/73)), I don't see how this is possible without provding an API that returns a price signal back to the SSP without rendering an ad. An example of what this might look like:

```javascript
// This can be thought of as an OpenRTB Bid Response (to the original contextual
// bid request).
let sspContextualOrtbBidResp = {
    "adm": "...creative html...",
    "price": 1.5,
    ...
}

// This could be thought of as an OpenRTR Bid Request, since the meta-data passed 
// in is likely to be very similar to the original contextual bid request.
let sspOrtbBidReq = {
    "site": {
        "ref": "https://reverb.com/",
        "publisher": {
            ...
        }
    }
    ...
}

// This could be thought of as an OpenRTB Bid Request to an SSP's bid cache on the
// browser. The browser responds with a structure that includes an opaque pointer
// (for rendering of the ad later should the winner be an IG ad) and the value
// of the browser's auction resolution, which would be max(contextual bid, IG bids).
let sspRealtimeResultPtr = navigator.resolveInterestGroupAd(sspContextualOrtbBidResp, sspOrtbBidReq)
```

At this point the SSP would know the value they provide to the publisher in realtime and could wrap this logic in their Prebid.js adapter and Prebid.js would function as it does today.

However, TurtleDove views an IG based bid price to be a privacy escape vector so this approach isn't feasible. While this model was a nice thought exercise demonstrating how an SSP might function in a TurtleDove world if an SSP scoped decision was possible, I think we need to assume that it's not.

## Where are we?

Some fraction of the demand is locked behind the TurtleDove curtain, we can't get a price signal out from behind the curtain, and publisher's need to stay in control of their business models while integrating different demand sources.

This is remarkably similar to the GAM/AdX situation discussed before. We need a solution that allows publishers to own their business model, SSPs to control the logic specific to the demand they represent, and a way to do so in a blackbox environment.
