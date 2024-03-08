## [L-01] Event should be emitted for ```release()```

## Proof of Concept
For all other functions events will be emitted, like when users try to stake, withdraw or being slashed and slashed tokens to be burnt, this should also hold for ```release()``` but there is no event to be emitted. Suggest add an event for release.
