---
---
# Explicit Locking <a name="explicit-locking"/>

 

## Introduction
This package contains an implementation of an Ehcache which provides for explicit locking,
using Read and Write locks.
It is possible to get more control over Ehcache's locking behaviour to allow business logic
to apply an atomic change with guaranteed ordering across one or more keys in one or more caches. It can therefore
be used as a custom alternative to `XA` Transactions or `Local` transactions.

With that power comes a caution. It is possible to create deadlocks in your own business logic using this
API.

Note that prior to Ehcache 2.4, this API was implemented as a CacheDecorator and was available in the
`ehcache-explicitlocking` module. It is now built into the `core` module.

## The API
The following methods are available on `Cache` and `Ehcache`.

<pre><code>/**
* Acquires the proper read lock for a given cache key
*
* @param key - The key that retrieves a value that you want to protect via locking
*/
public void acquireReadLockOnKey(Object key) {
   this.acquireLockOnKey(key, LockType.READ);
"/>
/**
* Acquires the proper write lock for a given cache key
*
* @param key - The key that retrieves a value that you want to protect via locking
*/
public void acquireWriteLockOnKey(Object key) {
   this.acquireLockOnKey(key, LockType.WRITE);
"/>
/**
* Try to get a read lock on a given key. If can't get it in timeout millis then
* return a boolean telling that it didn't get the lock
*
* @param key - The key that retrieves a value that you want to protect via locking
* @param timeout - millis until giveup on getting the lock
* @return whether the lock was awarded
* @throws InterruptedException
*/
public boolean tryReadLockOnKey(Object key, long timeout) throws InterruptedException {
   Sync s = getLockForKey(key);
   return s.tryLock(LockType.READ, timeout);
"/>
/**
* Try to get a write lock on a given key. If can't get it in timeout millis then
* return a boolean telling that it didn't get the lock
*
* @param key - The key that retrieves a value that you want to protect via locking
* @param timeout - millis until giveup on getting the lock
* @return whether the lock was awarded
* @throws InterruptedException
*/
public boolean tryWriteLockOnKey(Object key, long timeout) throws InterruptedException {
   Sync s = getLockForKey(key);
   return s.tryLock(LockType.WRITE, timeout);
"/>
/**
* Release a held read lock for the passed in key
*
* @param key - The key that retrieves a value that you want to protect via locking
*/
public void releaseReadLockOnKey(Object key) {
   releaseLockOnKey(key, LockType.READ);
"/>
/**
* Release a held write lock for the passed in key
*
* @param key - The key that retrieves a value that you want to protect via locking
*/
public void releaseWriteLockOnKey(Object key) {
   releaseLockOnKey(key, LockType.WRITE);
"/>
/**
* Returns true if a read lock for the key is held by the current thread
*
* @param key
* @return true if a read lock for the key is held by the current thread
*/
boolean isReadLockedByCurrentThread(Object key);
/**
* Returns true if a write lock for the key is held by the current thread
*
* Only Terracotta clustered cache instances currently support querying a thread's read lock hold status.
*
* @param key
* @return true if a write lock for the key is held by the current thread
*/
boolean isWriteLockedByCurrentThread(Object key);
</code></pre>

## Example
Here is a brief example:

<pre><code>String key = "123";
Foo val = new Foo();
cache.acquireWriteLockOnKey(key);
try {
	cache.put(new Element(key, val));
} finally {
	cache.releaseWriteLockOnKey(key);
"/>
...sometime later
String key = "123";
cache.acquireWriteLockOnKey(key);
try {
	Object cachedVal = cache.get(key).getValue();
	cachedVal.setSomething("abc");
 	cache.put(new Element(key, cachedVal));
 } finally {
cache.releaseWriteLockOnKey(key);
 "/>
</code></pre>

## Supported Topologies
Except as noted in the Javadoc (see above), explicit locking is supported in Ehcache standalone and also in Distributed Ehcache. It is not supported in Replicated Ehcache.

## How it works
A READ lock does not prevent other READers from also acquiring a READ lock and reading. A
READ lock cannot be obtained if there is an outstanding WRITE lock - it will queue.
A WRITE lock cannot be obtained while there are outstanding READ locks - it will queue.
In each case the lock should be released after use to avoid locking problems. The lock
release should be in a `finally` block.
If before each read you acquire a READ lock and then before each write you acquire a WRITE
lock, then an isolation level akin to READ_COMMITTED is achieved.
