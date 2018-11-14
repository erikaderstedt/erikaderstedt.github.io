---
title: "Core Data with file wrappers"
date: 2012-08-14 18:44
comments: true
categories: [Cocoa, Core Data]
---
NSPersistentDocument doesn't support file wrappers out of the box. Apple provides some sample code, ["File wrappers with Core Data Documents"](http://developer.apple.com/library/mac/#samplecode/PersistentDocumentFileWrappers/Introduction/Intro.html), that shows you how. I used this design in the current version of my main app, which shipped back in december 2010.  With the new auto-save and versioning system introduced in OS X 10.7, the code got a lot hairier. I eventually managed to get autosave working well with file wrappers, but only after I had experienced quite a few edge cases (with Dropbox folders, for example). The code was… ugly.
<!-- more -->
When Apple rolled out the first preview of OS X 10.8, I wasn't surprised to find that my code was broken. A few months, a lot of debugging and a couple of filed Radars later, I was still far from working code. Finally, I took a step back and looked at the contract for NSDocument and subclasses:

* maintain document state
* upon request, read from a URL
* upon request, save to a URL

I realized that in the file wrapper case, NSPersistentDocument brings practically nothing but trouble. The solution was simple: throw it out.

Inherit from NSDocument directly
--------------------------------
Set up a persistent store coordinator, and a managed object context. Adding accessors for these ivars will make the transition smooth.

	- (NSPersistentStoreCoordinator *)persistentStoreCoordinator {
		if (_psc == nil) {
			_psc = [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:[self managedObjectModel]];
		}
		return _psc;
	}

	- (NSManagedObjectContext *)managedObjectContext {
		if (_context == nil) {
			_context = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSMainQueueConcurrencyType];
			[_context setPersistentStoreCoordinator:[self persistentStoreCoordinator]];
			[self setUndoManager:[_context undoManager]];
		}
		return _context;
	}

Use a local cache of the user's document
----------------------------------------

When asked to read a URL, copy the entire file wrapper to a unique location somewhere in your cache folder. Find your persistent store within the copy, and add it to your store coordinator. If you have large binary files within the wrapper, you can soft-link these to the original to save time and disk space. 

Whenever AppKit asks you to write the document, save to the persistent store in your local cache. Then copy the cached version onto the destination URL (which can be an autosave URL or the original document URL). Don't forget to update the file metadata, if required.

Gotchas
-------
* Make sure you disconnect from all persistent stores before reading from a URL.
* Handle the case where the user lacks write permissions on the file wrapper.
* Make sure that you copy the document to a truly unique location. If two instances of your app are running simultaneously and access the same persistent store, you can end up duplicating every object in the store.

Summary
-------
The obvious drawback with this method is that you need to keep an extra copy of the document inside your cache folder. Since the autosave system does this anyway, I'm not sure if this is a major problem.

Ironically, dealing with file wrappers in this way addresses one of the main problems with using NSPersistentDocument with normal files: sandboxing. You can just as easily write normal files instead of wrappers. The sqlite journal files will be created inside your sandbox.

Finally, a huge advantage of this method is that you can save to a persistent store as often as you like. Some advanced fetches work only on the persistent object graph, not the current one.
