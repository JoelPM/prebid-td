# prebid.js UML markup
```
User -> "pub.com": 1. navigates browser to pub.com
activate "pub.com"
"pub.com" -> "prebid.js": 2. load, configure, execute prebid.js
activate "prebid.js"
"prebid.js" -> SSPs: 3. make ad requests
activate SSPs
SSPs -> SSPs: 4. traffic quality controls
SSPs -> DSPs: 5. make bid requests
activate DSPs
SSPs <- DSPs: 6. respond with bids
deactivate DSPs
SSPs -> SSPs: 7. pub ad controls
"prebid.js" <- SSPs: 8. respond with ads
deactivate SSPs
"prebid.js" -> "prebid.js": 9. store bids/creatives
"pub.com" <- "prebid.js": 10. SSP bid information given to page
deactivate "prebid.js"
deactivate "pub.com"

note over "pub.com", "gam.js", "prebid.js": pub.com transfers control from prebid.js to gam.js at this point

"pub.com" <-> "gam.js": 11. configure K/V pairs based on prebid.js bids
activate "pub.com"
activate "gam.js"
"pub.com" -> "gam.js": 12. load slot
"gam.js" -> "GAM/AdX": 13. ad request
activate "GAM/AdX"
"gam.js" <- "GAM/AdX": 14. respond with winning ad, which may be prebid.js line item
deactivate "GAM/AdX"
alt GAM/AdX highest bid
"gam.js" -> "gam.js": 15. render GAM/AdX result
else Prebid.js highest bid
"gam.js" -> "prebid.js": 15. render highest stored SSP bid/ad
activate "prebid.js"
"gam.js" <- "prebid.js"
deactivate "prebid.js"
end
"pub.com" <- "gam.js"
deactivate "gam.js"
User <- "pub.com": 16. show highest value advertisement
deactivate "pub.com"
```

# prebid-ig.js UML markup
```
User -> "pub.com": 1. navigates browser to pub.com
activate "pub.com"
"pub.com" -> "prebid.js": 2. load, configure, execute prebid.js
activate "prebid.js"
"prebid.js" -> SSPs: 3. make contextual ad requests
activate SSPs
SSPs -> SSPs: 4. traffic quality controls
SSPs -> DSPs: 5. make contextual bid requests
activate DSPs
SSPs <- DSPs: 6. respond with contextual bids
deactivate DSPs
SSPs -> SSPs: 7. pub ad controls
"prebid.js" <- SSPs: 8. respond with contextual ads
deactivate SSPs
"prebid.js" -> "prebid.js": 9. store contextual bids/creatives
"pub.com" <- "prebid.js": 10. SSP contextual bid information given to page
deactivate "prebid.js"
deactivate "pub.com"

note over "pub.com", "gam.js", "prebid.js": pub.com transfers control from prebid.js to gam.js at this point.\nWe assume that gam.js provides an API for receiving contextual ad information and doing selection between their contextual ad and IG based ads.

"pub.com" <-> "gam.js": 11. configure K/V pairs based on prebid.js bids
activate "pub.com"
activate "gam.js"
"pub.com" -> "gam.js": 12. load slot
"gam.js" -> "GAM/AdX": 13. contextual ad request
activate "GAM/AdX"
"gam.js" <- "GAM/AdX": 14. respond with winning contextual ad, which may be prebid.js line item
deactivate "GAM/AdX"
"pub.com" <- "gam.js": 15. contextual ad/bid available
deactivate "gam.js"

"pub.com" -> "prebid.js": 16. store GAM/AdX contextual ad/bid
activate "prebid.js"
"pub.com" <- "prebid.js"
deactivate "prebid.js"
deactivate "pub.com"

note over "pub.com", "gam.js": All contextual ads/bids are now in the page. Browser (or DoveKey) may contain IG bids. Now the final selection process must take place.
```

# prebid-td-ig.js UML markup
User -> "pub.com"
activate "pub.com"
"pub.com" -> "prebid.js": 1. render best ad

activate "prebid.js"
"prebid.js" <-> "prebid.js": 2. select contextual winner

"prebid.js" <-> "prebid.js": 3. construct uber-metadata object (bundle all SSPs)

"prebid.js" -> turtledove: 4. navigator.renderInterestGroup

turtledove -> "prebid-ig.js": 5. run IG auction

"prebid-ig.js" -> "prebid-ig.js": 6. SSP local IG auction

"prebid-ig.js" -> "prebid-ig.js": 7. global IG auction

"prebid-ig.js" -> "prebid-ig.js": 8. context/IG auction

turtledove <- "prebid-ig.js": 9. true/false

alt IG winner
"prebid.js" <- turtledove: 10. true
"prebid.js" -> "prebid.js": 10.1. record IG win
else contextual winner
"prebid.js" <- turtledove: 10. false
"prebid.js" -> "prebid.js": 10.1. render contextual ad winner
"prebid.js" -> "prebid.js": 10.2. brecord contextual win
end

"pub.com" <- "prebid.js": 11. ad rendered
deactivate "prebid.js"
User <- "pub.com"
deactivate "pub.com"
