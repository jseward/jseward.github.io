---
layout: post
title: Dangers of initializing OpenSSL
---

Earlier this week I went to prepare a new patch for [Sins of a Dark Age](http://store.steampowered.com/app/251970). I integrated everything into the Live release branch. I ran my automated builds of both the game client and ICO server packages. Everything was fine, but upon testing out the game client I found it crashed on start up with a stack overflow exception (bad).

After confirming that it was only happening on the Live release branch, I went to recreate it on our development branch. It was crashing somewhere in OpenSSL due to infinite recursion. The only difference between the two branches were project level #ifdef for excluding / including certain features. Eventually I was able to recreate the crash by removing support for modal error dialogs. On Live error dialogs are hidden. Error dialogs halt execute on the main thread. This is a pretty good indication of a nasty timing crash among threads. Code shifted around just enough to cause threads to crash one another.

The chain of events was this:

1. Game starts up
2. Game finds a missing data file and reports an error
3. The error is queued onto an separate thread. All of our errors are reported to [Raygun](https://raygun.io/) using REST requests on this thread.
4. The REST request is sent using [libcurl](http://curl.haxx.se/libcurl/)
5. The game main thread continues setting up the engine, initializing my custom IO Completion Port library for connecting to Ironclad Online servers.
6. Part of IO Completion Port initialization is also to initialize OpenSSL

So the issue is that the main thread is initializing OpenSSL and mutating it's global state at the same time the worker thread is using OpenSSL.

The solution was simply to pull out OpenSSL initialization from within IOCP initialization. Instead OpenSSL is initialized first before the engine itself is initialized.
