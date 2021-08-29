# Technical Details and Limitations

Link enables the use of text files as application source by mapping workspace content to directories and files. There are some types of objects that cannot be supported, and a few other limitations to be aware of.

## Supported Objects

In the following, for want of a better word, the term *object* will be used to refer to any kind of named entity that can exist in an APL workspace - not limited to classes or instances.

**Supported:** Link supports objects of name class 2.1 (array), 3.1 (traditional function), 3.2 (d-function), 4.1 (traditional operator), 4.2 (d-operator), 9.1 (namespace), 9.4 (class) and 9.5 (interface).

**Unscripted Namespaces:** Namespaces created with `⎕NS` or `)NS` have no source code of their own and are mapped to directories. In Link version 3.0, one endpoint of a link is always an unscripted namespace and the other endpoint of the link is a directory.

**Scripted Namespaces:** So-called scripted namespaces, created using the editor or `⎕FIX`, have textual source and are treated the same way as functions and other "code objects". Link 3.0 does not support scripted namespaces, or any other objects that map to a single source file, as endpoints for a link.

It is likely that this restriction will be lifted in a future version of Link.

**Variables** are ignored by default, because most of them are not part of the source code of an application. However, they may be explicitly saved to file with [Link.Add](../API/Link.Add.md), or with the `-arrays` modifier of [Link.Create](../API/Link.Create.md) and [Link.Export](../API/Link.Export.md).

**Functions and Operators:** Link is not able to represent names which refer to primitive or derived functions or operators, or trains. You will need to define such objects in the source of another function, or a scripted namespace.

**Unsupported:** Link has no support for name classes 2.2 (field), 2.3 (property), 2.6 (external/shared variable), 3.3 (primitive or derived function or train), 4.3 (primitive or derived operator), 3.6 (external function) 9.2 (instance), 9.6 (external class) and 9.7 (external interface).

## Other Limitations

- Namespaces must be named. To be precise, it must be true that `ns≡(⎕NS⍬)⍎⍕ns`. Scripted namespaces must not be anonymous. When creating an unscripted namespace, we recommend using `⎕NS` dyadically to name the created namespace (for example `'myproject' ⎕NS ⍬` rather than `myproject←⎕NS ⍬`). This allows retrieving namespace reference from its display from (for example `#.myproject` rather than `#.[namespace]`).
- Link does not support namespace-tagged functions and operators (e.g. `foo←namespace.{function}`).
- Changes made using `←`, `⎕NS`, `⎕FX`, `⎕FIX`, `⎕CY`, `)NS` and `)COPY` or the APL line `∇` editor are not currently detected. For Link to be aware of the change, a call must be made to [Link.Fix](../API/Link.Fix.md). Similarly, deletions with `⎕EX` or `)ERASE` must be replaced by a call to [Link.Expunge](../API/Link.Expunge.md).
- Link does not support source files that define multiple names, even though `2∘⎕FIX` does support this.
- The detection of external changes to files and directories is currently only supported under .NET and .NET Core. Note that the built-in APL editor *will* detect changes to source files on all platforms, but not before the editor is opened.
- Source code must not have embedded newlines within character constants. Although `⎕FX` does allow this, Link will error if this is attempted. This restriction comes because newline characters would be interpreted as a new line when saved as text file. When newline characters are needed in source code, they should be implemented by a call to `⎕UCS` e.g. `newline←⎕UCS 13 10  ⍝ carriage-return + line-feed`
- Although Link 3.0 will work with Dyalog version 18.0, Dyalog v18.1 is recommended if it is important that all source be preserved as typed. Earlier versions of APL may occasionally lose the source as typed under certain circumstances.

## How does Link work?

Some people need to know what is happening under the covers before they can relax and move on. If you are not one of those people, do not waste any further time on this section. If you do read it, understand that things may change under the covers without notice, and we will not allow a requirement to keep this document up-to-date to delay work on the code. It is reasonably accurate as of April 2021.

**Terminology:** In the following, the term *object* is used very loosely to refer to functions, operators, namespaces, classes and arrays.

### What Exactly is a Link?

A link connects a namespace in the active workspace (which can be the root namespace `#`) to a directory in the file system. 

When a link is created:

- An entry is created in the table which is stored in the workspace using an undocumented I-Beam, recording the endpoints and all options associated with the Link. [Link.Status](../API/Link.Status.md) can be used to report this information. Earlier versions used `⎕SE.Link.Links`, but version 3.0 only uses this for links with an endpoint in `⎕SE`.

- Depending on which end of the link is specified as the source, APL source files are created from workspace definitions, or objects are loaded into the workspace from such files. These processes are described in more detail in the following sections.

- By default, if .NET is available, a .NET File System Watcher is created to watch the directory for changes so that those changes can immediately be replicated in the workspace.

#### Creating APL Source Files and Directories

Link writes textual representations of APL objects to UTF-8 text files. Most of the time, it uses the system function `⎕SRC` to extract the source form writing it to file using `⎕NPUT`. There are two exceptions to this:

- So-called "unscripted" namespaces, which contain other objects but do not themselves have a textual source, are represented as sub-directories in the file system (which may contain source files for the objects within the namespace).
- Arrays are converted to source form using the function ` ⎕SE.Dyalog.Array.Serialise`. It is expected that the APL language engine will, in the future, support the "literal array notation", and that `⎕SRC`will one day be extended to perform this function, but there is as yet no schedule for this.

#### Loading APL Objects from Source

With the exception of variables stored in `.apla` files, which are processed by `⎕SE.Dyalog.Array.Deserialise`, Link loads code into the workspace using `2 ⎕FIX 'file://...'`. 

When you are watching both sides of a link, Link delegates the work of tracking the links to the interpreter. In this case, editing objects will cause the editor itself (not Link) to update the source file. You can inspect the links which are maintained by the interpreter using a family of I-Beams numbered 517x. When a *new* function, operator, namespace or class is created, a hook in the editor calls Link code which generates a new file and sets up the link.

If .NET is available, Link uses a File System Watcher to monitor linked directories and immediately react to file creation, modification or deletion.

### The Source of Link

Link consists of a set of API functions which are loaded into the namespace `⎕SE.Link`, when APL starts, from **$DYALOG/StartupSession/Link**. The user command file **$DYALOG/SALT/SPICE/Link.dyalog** provides access to the interactive user command covers that exist for most of the API functions. The code is included with installations of Dyalog version 18.1 or later. 

If you want to use Link with version 18.0 or download and install Link from GitHub, see the [installation instructions](../Usage/Installation.md).

### The Crawler

In a future version of Link, hopefully available during 2021, an optional and configurable crawler will be able to run in the background and occasionally compare linked namespaces and directories, using the same logic as [Link.Resync](../API/Link.Resync.md), and deal with anything that might have been missed by the automatic mechanisms. This will be especially useful if:

* The File System Watcher is not available on your platform
* You add functions or operators to the active workspace without using the editor, for example using ``)COPY`` or dfn assignment.

The section on [supported objects](#supported-objects) provides much more information about the type of APL objects that are supported by Link.

### Breaking Links

If [Link.Break](../API/Link.Break.md) is used to explicitly break an existing Link, the entry is removed from `⎕SE.Link.Links` and the namespace reverts to being a completely "normal" namespace in the workspace. If file system watch was active, the watcher is disabled. Any information that the interpreter was keeping about connections to files is removed using `5178⌶`. None of the definitions in the namespace are modified by the process of breaking a link.

If you delete a linked namespace using `)ERASE` or `⎕EX`, Link may not immediately detect that this has happened. However, if you call `Link.Status`, or make a change to a watched file that causes the file system watcher to attempt to update the namespace, Link will discover that something is amiss, issue a warning, and delete the link.

If you completely destroy the active workspace using `)LOAD` or `)CLEAR`, all links will be deleted.

## The Future
To summarise, the Link road map currently includes the following goals:

- Adding the [crawler](#the-crawler), which will automatically run [Link.Resync](../API/Link.Resync.md) in the background, in order to detect and help eliminate differences between the contents of linked namespaces and the corresponding directories. It may replace the File System Watcher in environments where it is not available.
- Eliminating the use of SALT with a new implementation of user commands and other mechanisms for loading source code into the interpreter based on Link instead.
- Support for linking individual source files. Link 3.0 is only able to link a namespace to a directory. There are situations where it is practical to create a link to a single source file, particularly in the case of a scripted namespace.
- Improving integration with the APL interpreter so that the editor will honour a [Pause](../API/Link.Pause.md).

Over time, it is a strategic goal for Dyalog to move more of the work done by Link into the APL interpreter, such as:

- Serialisation and deserialisation of arrays, using the literal array notation
- File system watching or other mechanisms for detecting changes to source at both ends of a link