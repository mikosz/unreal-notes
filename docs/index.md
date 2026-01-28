Unreal Notes
============

Other pages
-----------

* [Unreal Garden](https://unreal-garden.com/) - originally BenUI, a useful resource in general, but the
  UPROPERTY/UFUNCTION/... specifier list is absolutely indispensable
* [Ikrima Dev](https://ikrima.dev/) - seems a bit outdated, but I've been finding some really useful notes in there every now
  and then

Logging
-------

### Setting log verbosity

To set a categories logging verbosity you have a few options:

#### DefaultEngine.ini (or other ini file)

```
[Core.Log]
CategoryName=NewVerbosity
```

#### Passing command-line argument

`-log CategoryName NewVerbosity`

#### Calling console command

`Log CategoryName NewVerbosity`

Reflection & Properties
-----------------------

### Iterating over enums

`UEnum::NumEnums()` is not reliable for iterating over possible enum values as it sometimes contains an extra entry,
the `_MAX` entry. A solution is to iterate like this:

```
const UEnum* const Enum = StaticEnum<EEnumType>();
const int32 NumEnumValues = ComparisonTypeEnum->NumEnums() - (ComparisonTypeEnum->ContainsExistingMax() ? 1 : 0);

for (int32 Index = 0; Index < NumEnumValues; ++Index)
{
	const EEnumTypeValue = static_cast<EEnumType>(Enum->GetValueByIndex(Index));
	...
```

I'm not 100% sure this covers all possible extra meta enum values, but this solution has worked for me until now.

### Changing default value

Picture this case:

```
USTRUCT()
struct FSomeStruct {
	...
	
	bool bBool = false;
};

UCLASS()
class USomeClass {
	...
	
	UPROPERTY()
	FSomeStruct S;
};
```

You have a class containing a struct field, which contains a bool field that defaults to false. You work happily like this
until some poor soul says that each time they work with objects of class `Some Class`, they need to do numerous clicks to
get to the property `Bool` and change the value to true, because 99% of the cases they want it to be true. But you already
have countless instances of `Some Class` and `Some Struct` in your game - changing the default to true or assigning it in
the constructor of `Some Class` would change all instances to true (Unreal doesn't serialize non-overwritten property
values - this is very useful in most cases, but troublesome in this case).

You could override `OnSerialize` for `Some Class` and keep a serialized version of the object, where all older versions
would default to false and all new versions would default to true, but this adds data to a struct that perhaps shows up
often and is otherwise small and it feels like a waste (not to mention that it's a bit of coding that's easy to get wrong).

There's a very neat solution for this, which is to implement `PostInitProperties` for `Some Class`:

```c++
void USomeClass::PostInitProperties()
{
	Super::PostInitProperties();

	if (!HasAnyFlags(RF_ClassDefaultObject) && !HasAnyFlags(RF_WasLoaded))
	{
		S.bBool = true;
	}
}
```

This will be called after properties are initialized and will only change the value for newly instantiated objects -
not for loaded objects or the CDO.

Triggers
--------

###  Trigger overlap events at actor load

By default `OnComponentBeginOverlap` and similar events will not trigger during actor loading. You can see this by adding
a breakpoint in `UPrimitiveComponent::UpdateOverlapsImpl` and you'll see the initial call coming from `DispatchBeginPlay`
has `bDoNotifies` set to false. Now this is unless you set the `bGenerateOverlapEventsDuringLevelStreaming` flag
(`Generate Overlap Events During Level Streaming` in editor details) for the _actor_ containing your component.
Note that the flag suggests this has something to do with streaming, but actually this should say "level loading".

### Overlap updates

Unreal optimizes how and when overlaps are calculated. This is mainly implemented in `UPrimitiveComponent::UpdateOverlapsImpl`.
The function is called once when the containing actor begins play and then only when something possibly changes - e.g.
when something moves. When a primitive component moves it calls update overlaps for itself only. It generates a new
set of overlaps, and calls `BeginOverlapComponent` / `EndOverlapComponent` for components that are now overlapping
and haven't before and vice-versa. This also updates the `OverlappingComponents` array for the moved primitive
component as well as the components for which the overlap events are generated.

A side effect of this is that if you add any "logical" overlaps, that is overlaps not based on physical bodies, these
overalps may be cleaned up by another component. This was a problem for `ZkzCompositeVolumeComponent` where I tried to
generate overlaps in `UpdateOverlapsImpl`, but whenever an overlapping player moved it would remove them (as it was
unable to reproduce them based on physical bodies).

The only way I found to work around this issue is to not derive from `PrimitiveComponent` for composite volumes.
Unfortunately this also means overlap events are only generated for the composite volume, not for the player.

Configuration files
-------------------

