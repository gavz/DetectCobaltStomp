# DetectCobaltStomp
[![](https://img.shields.io/badge/Category-Defence%20Evasion%20Detection-E5A505?style=flat-square)]() [![](https://img.shields.io/badge/Language-C-E5A505?style=flat-square)]()

A quick(and perhaps dirty!) PoC tool to detect Module Stomping as implemented by Cobalt Strike with moderate to high confidence

## Prologue (a.k.a. Who even reads this?)
It all started with a [tweet](https://twitter.com/ilove2pwn_/status/1270619595087454210) from Austin a.k.a. [Mumbai](https://twitter.com/ilove2pwn_)/[SecIdiot](https://twitter.com/SecIdiot) whose Twitter account I was stalking yesterday for ideas(Hey in my defence, I was quite bored and his Twitter rants are rather enlightening!)

He described a quick'n'neat technique to detect a variant of Module Stomping and not just any, the one used by Cobalt Strike itself and promised more details about it soon but never dropped any code or a tool :( I assume he just moved on to other interesting stuff.

Oddly enough, at the time of writing, the Twitter account seems to have been borked but with a simple Google search of a keyword, I recovered for the readers what is pretty much the original tweet. It reads something like this:

"side affect of using LoadLibraryEx() & DONT_RESOLVE_DLL_REFERNCES(Cobalt Method to start Module Stomping) : Treated as EXE so LdrEnt->EntryPoint == NULL && (LdrEnt->Flags & LPDR_IMAGE_DLL == FALSE)"

Luckily, his tweet provided me with all the information I needed to build my own and that's exactly what I did after a little bit of digging around :)

## Usage
Grab the binaries from [here](https://github.com/slaeryan/DetectCobaltStomp/releases) or compile yourself using the batch script provided in the repo.

TL;DR - Just point the tool to the suspected process and let it do its thing :)

If it detects a module, the name and the image base address is going to be printed to the console providing a **hint** to the Blue teams after which further investigation can be conducted by dumping and comparing the contents of the two etc. 

Since v3.11 Cobalt Strike supports module stomping via `module_x86/x64` flag in malleable profile so be sure to test by enabling it. 

And as for the readers that aren't into adversary emulation using Cobalt Strike or do not currently possess a Cobalt Strike license, I have developed a small tool(called `artifactkit`) to replicate the same in-memory artefacts as generated by Cobalt Strike's module stomping.

It hollows `Chakra.dll` (Microsoft Chakra DLL) with your choice of payload(PIC blob) provided as a command-line argument and executes it.

Here's how it looks in-action:

![Replicating Cobalt Stomp](https://github.com/slaeryan/DetectCobaltStomp/blob/master/Screenshots/replicating-cobalt-stomp.PNG "Replicating Cobalt Stomp")

Just for reference, this tool can also detect [Module Stomping in C#](https://offensivedefence.co.uk/posts/module-stomping/) as implemented by [RastaMouse](https://twitter.com/_RastaMouse) and [CCob](https://twitter.com/_EthicalChaos_)(It is a well-written post that I can wholly recommend reading before continuing further).

Now, let's move on to what I think is the actual interesting part of this post :) 

## DetectCobaltStomp Internals
First of all, let's try to define Module Stomping/DLL Hollowing.

Quoting from [Cobalt Strike's blogpost](https://blog.cobaltstrike.com/2018/04/09/cobalt-strike-3-11-the-snake-that-eats-its-tail/):
```
The above does raise another problem. What about the permissions of that memory? We still have pages with execute permissions that are not tied to a loaded module. These permissions exist in legitimate applications, but these properties are a warm flame that attracts the hunters from their cyber blinds.

Cobalt Strike 3.11 also adds module stomping to Beacon’s Reflective Loader. When enabled, Beacon’s loader will shun VirtualAlloc and instead load a DLL into the current process and overwrite its memory.
```

**In essence, this technique is all about ditching dynamic allocation of private/mapped executable memory in the name of OPSEC and instead relying on code execution from an implanted/hollowed _RX_ image memory or `.text` of a mapped view of section object created from an image file handle using SEC_IMAGE flag**.

Now that is out of the way, let's talk about what(on earth!) presents this detection opportunity to blue-teams/defenders?

It should be pretty evident by now that to implement Module Stomping, we need to load a DLL in-memory using one way or the other either by utilizing the Windows loader or by implementing our own loader.

The author of Cobalt Strike decided to go for the latter option because it's a sensible choice, operationally-reliable, avoids a missing PEB loaded image IoC out-of-the-box and saves from having to implement a full-blown LDR subsystem among other things. However, it doesn't simply call `LoadLibrary` but rather the `Ex` variant with a special flag(due to stability issues).

So how exactly does it accomplish this? (in case you haven't already figured it out)
```
LoadLibraryExA(moduleName, NULL, DONT_RESOLVE_DLL_REFERENCES);
```

`DONT_RESOLVE_DLL_REFERENCES` is a flag that tells the loader to not call `DllMain` after load and also instructs the loader to not process the image's IAT and load dependent modules(as the name suggests).

But what really happens under the hood? Let's take a look and what better place to look than ReactOS documentation?

From the [ReactOS docs](https://doxygen.reactos.org/de/de3/dll_2win32_2kernel32_2client_2loader_8c.html#a79bcc196e9fe011383ce28a525692252):

**LoadLibraryExA(..., `DONT_RESOLVE_DLL_REFERENCES`) which internally calls LoadLibraryExW(..., `DONT_RESOLVE_DLL_REFERENCES`) which sets the `IMAGE_FILE_EXECUTABLE_IMAGE` flag before finally calling its underlying NTAPI - LdrLoadDll(..., `IMAGE_FILE_EXECUTABLE_IMAGE`). This in-turn calls LdrpLoadDll(...,`IMAGE_FILE_EXECUTABLE_IMAGE`). It is here that the two things are set in the LDR_DATA_TABLE_ENTRY structure**:
1) `EntryPoint = NULL;`
2) `LDRP_IMAGE_DLL = FALSE OR in newer versions(>Windows 8) ImageDll = FALSE;`

**Essentially, it is internally marked and treated as an EXE rather than a DLL even if it isn't**

Here are two highlighted screenshots from ReactOS docs which will make this clearer:

![LoadLibraryExW](https://github.com/slaeryan/DetectCobaltStomp/blob/master/Screenshots/load-library-ex-w.png "LoadLibraryExW")

![LdrpLoadDll](https://github.com/slaeryan/DetectCobaltStomp/blob/master/Screenshots/ldrp-load-dll.png "LdrpLoadDll")

**The way `DetectCobaltStomp`(named in honour of Mumbai's tool as shared in the tweet) detects Module Stomping is by walking the `PEB` loaded module list of the suspected remote process for every module and checking for the two values. If set, it indicates to the defenders that the module is marked as EXE and ergo loaded in a fashion that is atypical of a DLL**.

But does it work? What about the greatest enemy of any detection tech - false positives?

Honestly, I don't know. I'd say this is a pretty damning IoC of Module Stomping in some cases especially if combined with other in-memory artefacts something which I'd definitely recommend threat-hunters to look out for(ergo, moderate to high confidence).

Aside from all this, the actual code is quite simple and small. I have also taken the liberty to comment as much as I can in-between lines so it should be easy enough to understand :)

## Screenshots
Because screenshots or it didn't happen ;)

![Detect Cobalt Stomp](https://github.com/slaeryan/DetectCobaltStomp/blob/master/Screenshots/detect-cobalt-stomp.PNG "Detect Cobalt Stomp")

![Cobalt Strike](https://github.com/slaeryan/DetectCobaltStomp/blob/master/Screenshots/cobalt-strike.png "Cobalt Strike")

## Detection Remediation
If it is not evident enough already, this is **NOT** a flaw in the implementation of the Cobalt Strike product itself in any way but rather an interesting "side-effect" of this variant of Module Stomping.

In short, this is kinda unavoidable :( Or is it really? ;)

Well, there's the other type of Module Stomping(with its own pros and cons) where we create a section from a file handle to a DLL on-disk and map a view of the section in the local process, in essence implementing our own partial loader, and then we can move onto the execution phase as we would after writing the payload.

[Here](https://gist.github.com/slaeryan/cd5edfac7a99547a14e808bd1541256e) is a PoC to map an image(Ntdll.dll) from disk.

One thing to point out here is that the mapped image will be missing entries from the PEB loaded module list altogether which may or may not be an IoC in itself and I would recommend manually linking the module to PEB.

## Conclusion
Although I'd consider myself from the other side, I like to keep a close eye on the Blue team world including new detection tech because I believe only by understanding defences in-depth can we as (wannabe)adversaries evolve our tradecraft game to stay ahead of modern defences.

That was my very intention behind releasing this and I hope that this provides some insights to Team Red as much as does it does to the Blue Team :)

## Credits
1. [Mumbai](https://twitter.com/ilove2pwn_)/[SecIdiot](https://twitter.com/SecIdiot) I'm afraid all the credit for this goes to him for discovering this curious little behaviour
2. [am0nsec](https://twitter.com/am0nsec) for [wspe](https://github.com/am0nsec/wspe/blob/master/AMSI/amsi_module_patch.c) from where I adapted some code
3. [Geoff Chappell](https://twitter.com/geoffchappell) for [https://www.geoffchappell.com/studies/windows/win32/ntdll/structs/ldr_data_table_entry.htm](https://www.geoffchappell.com/studies/windows/win32/ntdll/structs/ldr_data_table_entry.htm) On a side-note, this is one of the best resources to find more about undocumented stuff
4. [Alex Ionescu](https://twitter.com/aionescu) and all the other people who worked/are working on [ReactOS project and its docs](https://doxygen.reactos.org/) Seriously, this is such an excellent resource
5. Finally, thanks to good 'ole George for letting me test in his lab. I owe you one!

## Author
Upayan ([@slaeryan](https://twitter.com/slaeryan)) [[slaeryan.github.io](https://slaeryan.github.io)]

## License
All the code included in this project is licensed under the terms of the GNU GPLv2 license.

#

[![](https://img.shields.io/badge/slaeryan.github.io-E5A505?style=flat-square)](https://slaeryan.github.io) [![](https://img.shields.io/badge/twitter-@slaeryan-00aced?style=flat-square&logo=twitter&logoColor=white)](https://twitter.com/slaeryan) [![](https://img.shields.io/badge/linkedin-@UpayanSaha-0084b4?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/upayan-saha-404881192/)