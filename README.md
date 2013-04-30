## New public API

### Object.observe

A new function `Object.observe(O, callback)` is added, which behaves as follows:

1. If Type(O) is not Object, throw a TypeError exception.
1. If IsCallable(callback) is not true, throw a TypeError exception.
1. If IsFrozen(callback) is true, throw a TypeError exception.
1. Let notifier be the result of calling [[GetNotifier]], passing O.
1. Let changeObservers be [[ChangeObservers]] of notifier.
1. If changeObservers already contains callback, return.
1. Append callback to the end of the changeObservers list.
1. Let observerCallbacks be [[ObserverCallbacks]].
1. If observerCallbacks already contains callback, return.
1. Append callback to the end of the observerCallbacks list.
1. Return O.

### Object.unobserve

A new function `Object.unobserve(O, callback)` is added, which behaves as follows:

1. If Type(O) is not Object, throw a TypeError exception.
1. If IsCallable(callback) is not true, throw a TypeError exception.
1. Let notifier be the result of calling [[GetNotifier]], passing O.
1. Let changeObservers be [[ChangeObservers]] of notifier.
1. If changeObservers does not contain callback, return.
1. Remove callback from the changeObservers list.
1. Return O.

### Object.deliverChangeRecords

A new function `Object.deliverChangeRecords(callback)` is added, which behaves as follows:

1. If IsCallable(callback) is not true, throw a TypeError exception.
1. Call [[DeliverChangeRecords]] with arguments: callback repeatedly until it returns false (no records were pending for delivery and callback no invoked).
1. Return.

### Object.getNotifier

A new function `Object.getNotifier(O)` is added, which behaves as follows:

1. If Type(O) is not Object, throw a TypeError exception.
1. If O is frozen, return null.
1. Return the result of calling [[GetNotifier]], passing O.


## New internal properties and objects

### [[ObserverCallbacks]]

There is now an ordered list, [[ObserverCallbacks]] which is shared per event queue. It is initially empty.

Note: This list is used to provide a deterministic ordering in which callbacks are called.

### [[Notifier]]

Every object O now has a [[Notifier]] internal property which is initially undefined.

Note: This gets lazily initialized to a notifier object which is an object with the [[NotifierPrototype]] as its [[Prototype]].

### [[PendingChangeRecords]]

Every function now has a [[PendingChangeRecords]] internal property which is an ordered list of ChangeRecords. It is initially empty.

Note: This list gets populated with change records as the objects that this function is observing are mutated. It gets emptied when the change records are delivered.

### [[NotifierPrototype]]

This object is used as the [[Prototype]] of all the notifiers that are returned by Object.getNotifier(O). It has one function called notify which is defined below.

?.??.?? [[NotifierPrototype]] is defined as follows:

1. Let notifierPrototype be the result of the abstract operation ObjectCreate (15.12).
1. Call the [[DefineOwnProperty]] internal method of notifierPrototype with arguments “notify”, the PropertyDescriptor {[[Value]]: notifyFunction, [[Writable]]: true, [[Enumerable]]: false, [[Configurable]]: true}, and false.
1. Let [[NotifierPrototype]] be notifierPrototype.

### [[NotifierPrototype]].notify

The notify function of the [[NotifierPrototype]] is defined as follow:

Let notifyFunction be a function when invoked does the following:

1. Let changeRecord be the first argument to the function.
1. Let notifier be [[This]].
1. If Type(notifier) is not Object, throw a TypeError exception.
1. If notifier does not have an internal property [[Target]] return.
1. Let type be the result of calling the [[Get]] internal method of changeRecord with “type”.
1. If Type(type) is not string, throw a TypeError exception.
1. Let changeObservers be the result of getting the internal property [[ChangeObservers]] of notifier.
1. If changeObservers is empty, return.
1. Let target be [[Target]] of notifier.
1. Let newRecord be the result of the abstract operation ObjectCreate (15.12).
1. Call the [[DefineOwnProperty]] internal method of newRecord with arguments “object”, the Property Descriptor {[[Value]]: target, [[Writable]]: false, [[Enumerable]]: true, [[Configurable]]: false}, and true.
1. For each enumerable property name N in changeRecord,
1. If N is not “object”, then
1. Let value be the result of calling the [[Get]] internal method of changeRecord with N.
1. Call the [[DefineOwnProperty]] internal method of newRecord with arguments N, the Property Descriptor {[[Value]]: value, [[Writable]]: false, [[Enumerable]]: true, [[Configurable]]: false}, and true.
1. Set the [[Extensible]] internal property of newRecord to false.
1. Let changeObservers be [[ChangeObservers]] of notifier.
1. Call the [[EnqueueChangeRecord]], passing newRecord and changeObservers as arguments.


## New Internal Algorithms

### [[GetNotifier]]

There is now a [[GetNotifier]] internal algorithm:

?.??.?? [[GetNotifier]] \(O)

1. Let notifier be [[Notifier]] of O.
1. If notifier is undefined:
  1. Let notifier be the result the abstract operation ObjectCreate (15.12).
  1. Set the [[Prototype]] of notifier to [[NotifierPrototype]].
  1. Set the internal property [[Target]] of notifier to be O.
  1. Set the internal property [[ChangeObservers]] of notifier to be a new empty list.
  1. Set [[Notifier]] of O to notifier.
1. return notifier.

Note: The [[GetNotifier]] lazily sets the [[Notifier]] internal property.

### [[EnqueueChangeRecord]]

There is now an abstract [[EnqueueChangeRecord]] internal algorithm:

?.??.?? [[EnqueueChangeRecord]] \(R, T)

When the [[EnqueueChangeRecord]] internal algorithm is called with ChangeRecord R and ChangeObservers T, the following steps are taken:

1. For each observer in T, do:
  1. Let pendingRecords be the result of getting [[PendingChangeRecords]] for observer.
  1. Append R to the end of pendingRecords.

### [[DeliverChangeRecords]]

There is an abstract [[DeliverChangeRecords] internal algorithm.

?.??.?? [[DeliverChangeRecords]] \(C)

When the [[DeliverChangeRecords]] internal algorithm is called with callback C, the following steps are taken:

1. Let changeRecords be [[PendingChangeRecords]] of C.
1. Clear the [[PendingChangeRecords]] of C.
1. Let array be the result of the abstraction operation ArrayCreate (15.4) with argument 0.
1. Let n be 0.
1. For each record in changeRecords, do:
  1. Call the [[DefineOwnProperty]] internal method of array with arguments ToString(n), the PropertyDescriptor {[[Value]]: record, [[Writable]]: true, [[Enumerable]]: true, [[Configurable]]: true}, and false.
  1. Increment n by 1.
1. If array is empty, return false.
1. Call the [[Call]] internal method, (silently ignoring any thrown exception or return value) of C, passing undefined as the this parameter, and a single argument, array.
1. Return true.

Note: The user facing function Object.deliverChangeRecords returns undefined to prevent detection if anything was delivered or not.


### [[DeliverAllChangeRecords]]

There is an abstract [[DeliverAllChangeRecords]] internal algorithm:

?.??.?? [[DeliverAllChangeRecords]]

When the [[DeliverAllChangeRecords]] internal algorithm is called, the following steps are taken:

1. Let observers be the result of getting [[ObserverCallbacks]]
1. Let anyWorkDone be a boolean value, initially set to false.
1. For each observer in observers, do:
  1. Let result be the result of calling [[DeliverChangeRecords]] with observer.
  2. If result is true:
      1. Set anyWorkDone to true.
1. Return anyWorkDone.

Note: It is the intention that the embedder will call this internal algorithm when it is time to deliver the change records.

### [[CreateChangeRecord]]

There is now an abstract operation [[CreateChangeRecord]]:

?.??.?? [[CreateChangeRecord]] \(type, object, name, oldDesc, newDesc)

When the abstract operation CreateChangeRecord is called with the arguments: type, object, name and oldDesc, the following steps are taken:

1. Let changeRecord be the result of the abstraction operation ObjectCreate (15.2).
1. Call the [[DefineOwnProperty]] internal method of changeRecord with arguments “type”, Property Descriptor {[[Value]]: type, [[Writable]]: false, [[Enumerable]]: true, [[Configurable]]: false}, and false.
1. Call the [[DefineOwnProperty]] internal method of changeRecord with arguments “object”, Property Descriptor {[[Value]]: object, [[Writable]]: false, [[Enumerable]]: true, [[Configurable]]: false}, and false.
1. Call the [[DefineOwnProperty]] internal method of changeRecord with arguments “name”, Property Descriptor {[[Value]]: name, [[Writable]]: false, [[Enumerable]]: true, [[Configurable]]: false}, and false.
1. If IsDataDescriptor(oldDesc) is true:
  1. If IsDataDescritor(newDesc) is false or SameValue(oldDesc.[[Value]], newDesc.[[Value]]) is false
      1. Call the [[DefineOwnProperty]] internal method of changeRecord with arguments “oldValue”, Property Descriptor {[[Value]]: oldDesc.[Value]], [[Writable]]: false, [[Enumerable]]: true, [[Configurable]]: false}, and false.
1. Set the [[Extensible]] internal property of changeRecord to false.
1. Return changeRecord.


## Modifications to existing internal algorithms

### [[DefineOwnProperty]]

Modify the [[DefineOwnProperty]] algorithm as indicated:

8.12.9 [[DefineOwnProperty]] /(P, Desc, Throw)

In the following algorithm, the term “Reject” means “If Throw is true, then throw a TypeError exception, otherwise return false”. The algorithm contains steps that test various fields of the Property Descriptor Desc for specific values. The fields that are tested in this manner need not actually exist in Desc. If a field is absent then its value is considered to be false.

When the [[DefineOwnProperty]] internal method of O is called with property name P, property descriptor Desc, and Boolean flag Throw, the following steps are taken:

1. Let current be the result of calling the [[GetOwnProperty]] internal method of O with property name P.
1. Let extensible be the value of the [[Extensible]] internal property of O.
1. Let notifier be the result of calling [[GetNotifier]], passing O.
1. Let changeObservers be [[ChangeObservers]] of notifier.
1. Let changeType be a string, initially set to “reconfigured”.
1. If current is undefined and extensible is false, then Reject.
1. If current is undefined and extensible is true, then
  1. If IsGenericDescriptor(Desc) or IsDataDescriptor(Desc) is true, then
      1. Create an own data property named P of object O whose [[Value]], [[Writable]], [[Enumerable]] and [[Configurable]] attribute values are described by Desc. If the value of an attribute field of Desc is absent, the attribute of the newly created property is set to its default value.
  1. Else, Desc must be an accessor Property Descriptor so,
      1. Create an own accessor property named P of object O whose [[Get]], [[Set]], [[Enumerable]] and [[Configurable]] attribute values are described by Desc. If the value of an attribute field of Desc is absent, the attribute of the newly created property is set to its default value.
  1. Let R be the result of calling [[CreateChangeRecord]] with arguments: “new”, O, P, current and Desc.
  1. Call [[EnqueueChangeRecord]] passing R and changeObservers.
  1. Return true.
1. Return true, if every field in Desc is absent.
1. Return true, if every field in Desc also occurs in current and the value of every field in Desc is the same value as the corresponding field in current when compared using the SameValue algorithm (9.12).
1. If the [[Configurable]] field of current is false then
  1. Reject, if the [[Configurable]] field of Desc is true.
  1. Reject, if the [[Enumerable]] field of Desc is present and the [[Enumerable]] fields of current and Desc are the Boolean negation of each other.
1. If IsGenericDescriptor(Desc) is true, then no further validation is required.
1. Else, if IsDataDescriptor(current) and IsDataDescriptor(Desc) have different results, then
  1. Reject, if the [[Configurable]] field of current is false.
  1. If IsDataDescriptor(current) is true, then
      1. Convert the property named P of object O from a data property to an accessor property. Preserve the existing values of the converted property’s [[Configurable]] and [[Enumerable]] attributes and set the rest of the property’s attributes to their default values.
  1. Else,
      1. Convert the property named P of object O from an accessor property to a data property. Preserve the existing values of the converted property’s [[Configurable]] and [[Enumerable]] attributes and set the rest of the property’s attributes to their default values.
1. Else, if IsDataDescriptor(current) and IsDataDescriptor(Desc) are both true, then
  1. If the [[Configurable]] field of current is false, then
      1. Reject, if the [[Writable]] field of current is false and the [[Writable]] field of Desc is true.
      1. If the [[Writable]] field of current is false, then
          1. Reject, if the [[Value]] field of Desc is present and SameValue(Desc.[[Value]], current.[[Value]]) is false.
      1. else, the [[Configurable]] field of current is true, so any change is acceptable.
  1. If [[Configurable]] is not in Desc or SameValue(Desc.[[Configurable]], current.[[Configurable]]) is true and [[Enumerable]] is not in Desc or SameValue(Desc.[[Enumerable]], current.[[Enumerable]]) is true and [[Writable]] is not in Desc or SameValue(Desc.[[Writable]], current.[[Writable]]) is true, then
      1. Set changeType to be the string “updated”.
1. Else, IsAccessorDescriptor(current) and IsAccessorDescriptor(Desc) are both true so,
  1. If the [[Configurable]] field of current is false, then
      1. Reject, if the [[Set]] field of Desc is present and SameValue(Desc.[[Set]], current.[[Set]]) is false.
      1. Reject, if the [[Get]] field of Desc is present and SameValue(Desc.[[Get]], current.[[Get]]) is false.
1. For each attribute field of Desc that is present, set the correspondingly named attribute of the property named P of object O to the value of the field.
1. Let R be the result of calling [[CreateChangeRecord]] with arguments: changeType, O, P, current and Desc.
1. Call [[EnqueueChangeRecord]] passing R and changeObservers.
1. Return true.

However, if O is an Array object, it has a more elaborate [[DefineOwnProperty]] internal method defined in 15.4.5.1. NOTE Step 10.b allows any field of Desc to be different from the corresponding field of current if current’s [[Configurable]] field is true. This even permits changing the [[Value]] of a property whose [[Writable]] attribute is false. This is allowed because a true [[Configurable]] attribute would permit an equivalent sequence of calls where [[Writable]] is first set to true, a new [[Value]] is set, and then [[Writable]] is set to false.

### [[Delete]]

Modify the [[Delete]] algorithm as indicated:

8.12.7 [[Delete]] /(P, Throw)

When the [[Delete]] internal method of O is called with property name P and the Boolean flag Throw, the following steps are taken:

1. Let desc be the result of calling the [[GetOwnProperty]] internal method of O with property name P.
1. If desc is undefined, then return true.
1. Let notifier be the result of calling [[GetNotifier]], passing O.
1. Let changeObservers be [[ChangeObservers]] of notifier.
1. If desc.[[Configurable]] is true, then
  1. Remove the own property with name P from O.
  1. Let R be the result of calling [[CreateChangeRecord]] with arguments: “deleted”, O, P and desc.
  1. Call [[EnqueueChangeRecord]] on passing R and changeObservers.
  1. Return true.
1. Else if Throw, then throw a TypeError exception.
1. Return false.


## Changing the [[Prototype]]

It has been agreed that we will support changing the [[Prototype]] by setting the __proto__ property. However, the spec draft for this is currently not done.

Whenever the [[Prototype]] is set on an object O, through any means, we need to do the following:

1. Let notifier be the result of calling [[GetNotifier]], passing O.
1. Let changeObservers be [[ChangeObservers]] of notifier.
1. Let oldPrototype be the old value of [[Prototype]].
1. Let changeRecord be the result of the abstraction operation ObjectCreate (15.2).
1. Call the [[DefineOwnProperty]] internal method of changeRecord with arguments “type”, Property Descriptor {[[Value]]: “prototype”, [[Writable]]: true, [[Enumerable]]: true, [[Configurable]]: true}, and false.
1. Call the [[DefineOwnProperty]] internal method of changeRecord with arguments “object”, Property Descriptor {[[Value]]: O, [[Writable]]: true, [[Enumerable]]: true, [[Configurable]]: true}, and false.
1. Call the [[DefineOwnProperty]] internal method of changeRecord with arguments “oldValue”, Property Descriptor {[[Value]]: oldPrototype.[Value]], [[Writable]]: true, [[Enumerable]]: true, [[Configurable]]: true}, and false.
1. Call [[EnqueueChangeRecord]] on passing changeRecord and changeObservers.
