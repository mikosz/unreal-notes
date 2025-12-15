Unreal Notes
============

Triggers
--------

###  Trigger events at actor load

By default `OnComponentBeginOverlap` and similar events will not trigger during actor loading. You can see this by adding
a breakpoing in `UPrimitiveComponent::UpdateOverlapsImpl` and you'll see the initial call coming from `DispatchBeginPlay`
has `bDoNotifies` set to false. Now this is unless you set the `bGenerateOverlapEventsDuringLevelStreaming` flag
(`Generate Overlap Events During Level Streaming` in editor details) for the _actor_ containing your component.
Note that the flag suggests this has something to do with streaming, but actually this should say "level loading".
