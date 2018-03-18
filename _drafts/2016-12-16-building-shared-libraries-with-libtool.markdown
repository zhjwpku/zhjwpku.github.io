---
layout: post
title: Building shared libraries with Libtool
date: 2016-12-16 11:00:00 +0800
author: John Calcote
tags:
- autoconf
- automake
- libtool
---

This is a book originally published on [Free Software magazine][ref1], I reedit it to make it more comfortable to read.

<center><h6>------ Below is the book ------</h6></center>

<h2>Chapter 5: Building shared libraries with Libtool</h2>

The person who invented the concept of shared libraries should be given a raise... and a bonus. The person who decided that shared library management and naming conventions should be left to the implementation should be flogged.

This opinion is the result of too much negative experience on my part with building shared libraries for multiple platforms without the aid of Libtool. The very existence of Libtool stands as a witness to the truth of this sentiment.

Libtool exists for one purpose only--to provide a standardized, abstract interface for developers desiring to create portable shared libraries. It abstracts both the shared library build process, and the programming interfaces used to dynamically load and access shared libraries at run time.

This chapter has [downloads](/assets/download/Autotools-Chapter-5.tgz)!

Before I get into a discussion of the proper use of Libtool, I should probably spend a few minutes on the features and functionality provided by shared libraries, so that you will understand the scope of the material I'm covering here.

**The benefits of shared libraries**

Shared libraries provide a way to ship reusable chunks of functionality in a convenient package that can be loaded into a process address space, either automatically at program load time by the operating system loader, or by code in the application itself, when it decides to load and access the library's functionality. The point at which an application binds functionality from a shared library is very flexible, and determined by the developer, based on the design of the program and the needs of the end-user.

The interfaces between the program executable and modules defined as shared libraries must be well-designed by virtue of the fact that shared library interfaces must be well-specified. This rigorous specification promotes good design practices. When you use shared libraries, you're essentially forced to be a better programmer.

Shared libraries may be (as the name implies) shared among processes. This sharing is very literal. The code segments for a shared library can be loaded once into physical memory pages. Those same memory pages can then be mapped into the process address spaces for multiple programs. The data pages must, of course, be unique per process, but global data segments are often small compared to the code segments of a shared library. This is true efficiency.

Shared libraries are easily updated during program upgrades. The base program may not have changed at all between two revisions of a software package. A new version of a shared library may be laid down on top of the old version, as long as its interfaces have not been changed. When interfaces are changed, two versions of the same shared library may co-exist side-by-side, because the versioning scheme used by shared libraries (and supported by Libtool) allows the library files to be named differently, but treated as the same library. Older programs may continue to use older versions of the library, while newer programs may use the newer versions.

If a software package specifies a well-defined "plug-in" interface, then shared libraries can be used to implement user-configurable loadable functionality. This means that additional functionality can become available to a program after it's been released, and third-parties can even add functionality to your program, if you publish a document describing your plug-in interface specification.

There are a few widely-known examples of systems such as this. Eclipse, for instance, is almost a pure plug-in framework. The base executable supports little more than a well-defined plug-in interface. Most of the functionality in an Eclipse application comes from library functions. Granted, Eclipse is written in Java, and uses Java class libraries, but the same concept can be (and has been) easily implemented in C or C++ using shared libraries.

**How shared libraries work**

As I mentioned above, the way a POSIX-based operating system implements shared libraries varies from platform to platform, but the general idea is the same for all platforms. The following discussion applies to shared library references that are resolved by the linker while the program is being built, and by the operating system loader at program load time.

**Dynamic linking at load time**

As a program executable image is being built, the linker (formally called a "link editor") maintains a table of unresolved function entry points and global data references. Each new symbol referenced by the object code being linked together, is added to this table. At the end of the linking process, all object files containing only unreferenced symbols are removed from the link list. All object files containing referenced symbols are linked together, and become part of the program executable image. If there are any outstanding references in the symbol table after all of the object files have been analyzed in this manner, the linker exits with an error message. On success, the final executable image may then be loaded and executed by a user. It is entirely self-contained, depending only upon itself.

Assuming that all undefined references are resolved during the linking process, if the list of objects to be linked contains one or more shared libraries, the linker will build the executable image from all non-shared objects specified on the linker command line. This includes all individual .o files and all static library archives. However it will add two tables to the binary image header; the first is the table of outstanding external references--those found only in shared libraries, and the second is a table of shared library names and versions in which the outstanding undefined references were found.

Later, when the operating system loader attempts to load this program, it must resolve the remaining outstanding references to symbols imported from the shared libraries named in the executable header. If the loader can't resolve all of the references, then a load error occurs, and the process is terminated with an operating system error message.

Note here that these external symbols are not tied to a specific shared library. The operating system will stop loading shared libraries as soon as it is able to resolve all of the outstanding symbol references. Usually, this happens after the last indicated shared library is loaded into the process address space, but there are exceptions.

*NOTE: This process differs a bit from the way a Windows operating system resolves symbols in Dynamic Link Libraries (DLLs). On Windows, a particular symbol is tied by the linker at program build time to a specifically named DLL.*

Using free-floating external references has both pros and cons. On some operating systems, unbound symbols can be satisfied by a library specified by the user. That is, a user can entirely replace a library (or a portion of a library) at run time by simply preloading one that contains the same symbols. On BSD and Linux based systems, for example, a user can use the "LD_PRELOAD" environment variable to inject a shared library into a process address space. Since such libraries are loaded first by the loader before any other libraries, symbols in the preloaded libraries will be located first by the loader when it tries to resolve external references.

In the following example, the "df" utility is executed in an environment containing the LD_PRELOAD variable, set to a path referring to a library that presumably contains a heap manager. This technique can be used to debug problems in your programs. By preloading your own heap manager, you can capture memory leaks in a log file, or debug memory block overruns. This sort of technique is used by such widely-known debugging aids as the valgrind package.

{% highlight shell %}
$ LD_PRELOAD=~/lib/libmymalloc.so /bin/df
{% endhighlight %}

Unfortunately, free-floating symbols can also lead to problems. For instance, two libraries can provide the same symbol name, and the dynamic loader can inadvertently bind an executable to a symbol from the wrong library. At best, this will cause a program crash when the wrong arguments are passed to the mis-matched function. At worst, it can present security risks, because the mis-matched function might be used to capture passwords and security credentials passed by the unsuspecting program.

C-language symbols do not include parameter information, so it's rather likely that symbols will clash in this manner. C++ symbols are a bit safer, in that the entire function signature (minus the return type) is encoded into the symbol name. However, even C++ is not immune to hackers purposely replacing security functions with their own versions of those functions.

**Automatic dynamic linking at run time**

The operating system loader can also use a very late form of binding, often referred to as "lazy binding". In this situation, the external reference entries in the jump table in the program header are initialized such that they refer to code in the dynamic loader itself.

When a program first calls such a "lazy" entry, the call will be routed to the loader, which will then (potentially) load the proper shared library, determine the actual address of the function, reset the entry point in the jump table, and finally redirect to the (now available) shared library function. The next time this happens, the jump table entry will have been correctly initialized, and the program will jump directly to the called function.

This lazy binding mechanism makes for very fast program startup, because shared libraries whose symbols are not bound until they're needed aren't even loaded until they're first referenced by the application program. Now, consider this--they may never be referenced. Which means they may never be loaded, saving both time and space. An example of this situation might be a word processor with a thesaurus feature, implemented in a shared library. How often do you use your thesaurus? Using automatic dynamic linking, chances are that the shared library containing the thesaurus code will never be loaded in a given execution of your word processor.

The problems with this method should be obvious, at this point. While using automatic run-time dynamic linking can give you faster load times, and better performance and space efficiency, it can also cause abrupt terminations of your application--without warning. If the loader can't find the requested symbol--perhaps the required library is missing--then it has no recourse except to abort the process.

Why not ensure that all symbols exist when the program is loaded? Well, if the loader resolved all symbols at load time, then it might as well populate the jump table entries at that point. After all, it had to load all the libraries to ensure that the symbols actually exist. This then entirely defeats the purpose of this binding method. Furthermore, even if the loader did bother to check out all external references at the point when the program was first started, there's nothing to stop someone from deleting one or more of these libraries before it's used, while the program is still running. Thus, even the pre-check is defeated.

The moral of this story is that you get what you pay for. If you don't want to pay the insurance premium for longer up-front load times, and more space consumed (even if you may never really need it), then you may have to take the hit of a missing symbol at run time, causing a program crash.

**Manual dynamic linking at run time**

One possible solution to the aforementioned problem is to take personal responsibility for the work done by the system loader. Then, when things don't go right, you have a little more control over the outcome. In the case of the thesaurus module, was it really necessary to terminate the program if the thesaurus library could not be loaded or didn't provide the correct symbols? Of course not, but the loader didn't know that. Only the programmer can make such value judgements.

When a program manages dynamic linking manually at run-time, the linker is left entirely out of the equation. The program doesn't call any shared library functions directly. Rather, shared library functions are referenced though function pointers that are populated by the application program itself at run time.

The way this works is that a program calls an operating system function to manually load a shared library into its own process address space. This system function returns a "handle", or an opaque value representing the loaded library. The program then calls another loader function to import a symbol from the library referred to by the handle. If all goes well, the operating system returns the address of the requested function or data item in the desired library. The program may then call the function, or access the global data item through this pointer.

If something goes wrong in one of these two steps--say the library could not be found, or the symbol was not found within the library, then it becomes the responsibility of the program to define the results--perhaps display an error message, indicating that the program was not configured correctly.

This is a little nicer than the way automatic dynamic run-time linking works; while the loader has no option but to abort, the application has a higher-level perspective, and can handle the problem much more gracefully. The drawback, of course, is that you as the programmer have to manage the process of loading libraries and importing symbols within your application code. However, this process is not really very difficult, as I'll explain later in this chapter.

**Using Libtool**

An entire book could be written about the details of shared libraries and their implementations on various systems. This short primer will suffice for your immediate needs; so I'll move on to how Libtool can be used to make a package maintainer's life a little easier.

The Libtool project was started in 1996 by Gordon Matzigkeit. Libtool was designed to extend Automake, but can be used independently within hand-coded makefiles, as well. The Libtool project is currently maintained by Bob Friesenhahn, Peter O'Gorman, Gary Vaughan and Ralf Wildenhues.

**Abstracting the build process**

First, I'll look at how Libtool helps during the build process. Libtool provides a script (ltmain.sh) that config.status executes in a Libtool-enabled project. The ltmain.sh script builds a custom version of the libtool script, specifically for your package. This libtool script is then used by your project's makefiles to build shared libraries specified using the LTLIBRARIES primary. The libtool script is really just a fancy wrapper around the compiler, linker and other tools. The ltmain.sh script should be shipped in a distribution tarball, as part of your end-user build system. Automake-generated rules ensure that this happens properly.

The libtool script insulates the build system author from the nuances of building shared libraries on multiple platforms. This script accepts a well-defined set of options, converting them to appropriate platform- and linker-specific options on the target platform and tool set. Thus, the maintainer need not worry about the specifics of building shared libraries on each platform. She need only understand the available libtool script options. These are well specified in the GNU Libtool manual, and I'll cover many of them in this chapter.

On systems that don't support shared libraries at all, the libtool script uses appropriate commands and options to build and link static libraries. This is all done in such a way that the maintainer is isolated from the differences between building shared libraries and static libraries.

You can emulate building your package on a static-only system by using the "--disable-shared" option on the configure command line for your project. This causes Libtool to assume that shared libraries cannot be built on the target system.

**Abstraction at run-time**

Libtool can also be used to abstract the programming interfaces supplied by the operating system for loading libraries and importing symbols. Programmers who've ever dynamically loaded a library on a Linux system are familiar with the standard Linux shared library API, including the functions, dlopen, dlsym and dlclose. These functions are provided by a system-level shared library, usually named "dl". Unfortunately, not all POSIX systems that support shared libraries provide the dl library, or functions using these names.

To address these differences, Libtool provides a shared library called "ltdl", which provides a clean, portable library management interface, very similar to the dlopen interface provided by the Linux loader. The use of this library is optional, of course, but highly recommended, because it provides more than just a common API across shared library platforms. It also provides an abstraction for manual run-time dynamic linking between shared library and non-shared library platforms.

"What!? How can that work?" You might ask. On systems that don't provide shared libraries, Libtool actually creates internal symbol tables within the executable containing all of the symbols that would otherwise be found in shared libraries on systems that support shared libraries. By using these symbol tables on these platforms, the lt_dlopen and lt_dlsym functions can make your code appear to be loading and importing symbols, when in fact, the "load" function does nothing more than return a handle to the appropriate symbol table, and the "import" function returns the address of some code that's been statically linked into the program itself.

The ltdl library is, of course, not really necessary for packages that don't use manual run-time dynamic linking. But if your package does--perhaps by providing a plug-in interface of some sort--then you'd be well-advised to use the API provided by ltdl to manage loading and linking to your plug-in modules--even if you only target systems that provide good shared library services. Otherwise, your source code will have to consider the differences in shared library management between your many target platforms. At the very least, some of your users will have to put on their "developer" hats, and attempt to modify your code so that it works on their odd-ball platforms. (They may have to do so anyway, but when they finish, their work can then be incorporated into Libtool, so that everyone else can take advantage of their efforts.)

**A word about the latest Libtool**

The most current version of Libtool is 2.2. However, many popular GNU/Linux distributions are still shipping Libtool version 1.5, so many developers don't know about the changes between these two versions. The reason for this is that certain backward-compability issues were introduced after version 1.5 that make it difficult for GNU/Linux distros to support the latest version of Libtool. The upgrade probably won't happen until all (or almost all) of the packages they provide have updated their configure.ac scripts to properly use the latest version of Libtool.

This is somewhat of a "chicken-and-egg" scenario--if distros don't ship it, how will developers ever start using it on their own packages? So it's not likely to happen any time soon. If you want to make use of the latest Libtool version while developing your packages (and I highly recommend that you do so), you'll probably have to download, build and install it manually, or look for an updated Libtool package from your distribution provider.

Downloading, building and installing Libtool manually is really trival:

{% highlight shell %}
$ wget ftp.gnu.org/gnu/libtool/libtool-2.2.tar.gz
...
$ tar xzf libtool-2.2.tar.gz
$ cd libtool-2.2
$ ./configure && make
...
$ sudo make install
...
{% endhighlight %}

Be aware that the default installation location (as with most of the GNU packages) is /usr/local. If you wish to install it into the /usr hierarchy, then you'll need to use the --prefix=/usr option on the configure command line.

You might also wish to use the --enable-ltdl-install option on the configure command line to install the ltdl libraries and header files into your lib and include directories.

**Adding shared libraries to Jupiter**

Now that I've presented that background information, I will take a look at how I might add a Libtool shared library to the Jupiter project. First, consider what I might do with a shared library in Jupiter. As mentioned above, I might wish to provide my users with some library functionality that their own applications could use. I might also have several applications in my package that need to share the same functionality. A shared library is a great tool for both of these scenarios, because I get the benefits of code reuse and memory savings, as the cost of the memory used by shared code is amortized across multiple applications--both internal and external to my project.

I'll add a shared library to Jupiter that provides the print functionality I use in the jupiter application. I'll do this by having the new shared library call into the libjupcommon.a static library. Remember that calling a routine in a static library has the same effect as linking the object code for the called routine right into the calling application (or shared library, as the case may be). The called routine ultimately becomes an integral part of the calling binary image (program or shared library).

Additionally, I'll provide a public header file from the Jupiter project that will allow external applications to call this same functionality. By doing this, I can allow other applications to "display stuff" in the same way that the jupiter program "displays stuff". (This would be significantly cooler if I was actually doing something useful in jupiter!).

**Using the LTLIBRARIES primary**

Automake has built-in support for Libtool. The LTLIBRARIES primary is provided by code in the Automake package, not the Libtool package. This really doesn't qualify as a pure extension, but rather more of an add-on package for Automake, where Automake provides the necessary infrastructure for that specific add-on package. You can't access the LTLIBRARIES primary functionality provided by Automake without Libtool, because the use of this primary obviously generates make rules that call the libtool build script.

I state all of this here because it bothers me that you can't really extend the list of primaries supported by Automake without modifying the actual Automake source code. The fact that Automake is written in perl is somewhat of a boon, because it means that it's possible to do it. But you've really got to understand Automake source code in order to do it properly. I envision a future version of Automake whereby code may be added to an Automake extension file that will allow the dynamic definition of new primaries.

It's a bit like the old FOSS addage, generally offered to someone complaining about lack of functionality in a particular package: "It's open source. Change it yourself!" This is very often easier said than done. Furthermore, what these people are actually telling you is to change your copy of the source code for your own purposes, not to change the master copy of the source code. Getting your changes accepted into the master source base often depends more on the quality of your relationship with the current project maintainers than it does on the quality of your coding skills. I'm not complaining, mind you. I'm merely stating a fact that should not be overlooked when one is considering making changes to an existing open source software package.

So why not ship Libtool as part of Automake, rather than as a separate package? Because Libtool can quite effectively be used independently of Automake. If you wish to try Libtool by itself, then please refer to the GNU Libtool manual for more information. The opening chapters in that manual describe the use of the libtool script as a stand-alone product. It's really as simple as modifying your makefile commands such that the compiler, linker and librarian are called using the libtool script, and then modifying some of your command line parameters, as required by Libtool.

**Public include directories**

Earlier in this book, I made the statement that a project sub-directory named include should only contain public header files--those that expose a public interface in your project. I'm now going to add just such a header file to the Jupiter project: so, I'll create an include directory. I'll add this directory at the top-level of the project directory structure.

If I had multiple shared libraries, I'd have a choice to make: do I create separate include directories for each library in the library source directory, or do I add a single top-level include directory? I usually use the following rule of thumb to determine the answer to this question: if the libraries are designed to work together as a group, and if consuming applications generally use the libraries as a group, then I use a single top-level include directory. If, on the other hand, the libraries can be effectively used independently, and if they offer fairly autonomous sets of functionality, then I provide individual include directories in my project's library subdirectories.

In the end, it really doesn't matter much, because the header files for these libraries will be installed in entirely different directory structures than those in which they exist within your project. In fact, make sure you don't inadvertently use the same file name for headers in two different libraries in your project, or you'll probably have problems installing these files. They generally end up all together in the "$(prefix)/include" directory, although this default can be overridden with the pkginclude prefix.

I'll also add a directory for the new Jupiter shared library, called libjupiter. These changes require adding references to these new directories to the top-level Makefile.am file's SUBDIRS variable, and then adding corresponding makefile references to the AC_CONFIG_FILES macro in the configure.ac script:

{% highlight shell %}
$ mkdir include
$ mkdir libjup
$ echo "SUBDIRS = common include libjup src" > Makefile.am
$ echo "include_HEADERS = libjupiter.h" > include/Makefile.am
$ vi configure.ac
...
AC_PREREQ([2.61])
AC_INIT([Jupiter], [1.0], [bugs@jupiter.org])
AM_INIT_AUTOMAKE
LT_PREREQ([2.2])
LT_INIT([dlopen])
...

AC_CONFIG_FILES([Makefile
        common/Makefile
        include/Makefile
        libjup/Makefile
        src/Makefile])
...
{% endhighlight %}

The `include` directory's Makefile.am file is trivial, containing only a single line, wherein the public header file, libjupiter.h is referred to in an Automake HEADERS primary. Note that I'm using the include prefix on this primary. You'll recall that the include prefix indicates that files specified in this primary are destined to be installed in the $(includedir) directory (eg., /usr/local/include). The HEADERS primary is much like the DATA primary, in that it specifies a set of files that are to be treated simply as data to be installed without modification or pre-processing. The only really tangible difference is that the HEADERS primary restricts the possible installation locations to those that make sense for header files.

The libjup/Makefile.am file is a bit more complex, containing four lines, as opposed to the usual one or two lines:

*libjup/Makefile.am*

{% highlight shell %}
lib_LTLIBRARIES = libjupiter.la
libjupiter_la_SOURCES = jup_print.c
libjupiter_la_LIBADD = ../common/libjupcommon.a
libjupiter_la_CPPFLAGS = -I$(top_srcdir)/include -I$(top_srcdir)/common
{% endhighlight %}

Let me analyze this file line by line. The first line is the primary one, and contains the usual prefix for libraries. The lib prefix indicates that the referenced products are to be installed in the $(libdir) directory. I might also have used the pkglib prefix to indicate that I wanted my libraries installed into the $(prefix)/lib/jupiter directory. Here, I'm using the LTLIBRARIES primary, rather than the older LIBRARIES primary. The use of this primary tells Automake to generate rules that use the libtool script, rather than calling the compiler and librarian (ar) directly to generate the products.

The second line lists the sources that are to be used for the first (and only) product. The third line indicates a set of linker options for this product. In this case, I'm specifying that the libjupcommon.a static library should be linked into (become part of) the libjupiter.so shared library.

There's an important concept regarding the *_LIBADD variable that you should strive to understand completely: Libraries that are consumed within, and yet built as part of the same project, should be referenced internally, using relative paths within the build directory hierarchy. Libraries that are external to a project generally need not be referenced explicitly at all, as the $(LIBS) variable should already contain the appropriate "-L" and "-l" options for those libraries. These options come from attempts made by the configure script to locate these libraries, using the appropriate AC_CHECK_LIBS, or AC_SEARCH_LIBS macros.

The fourth line indicates a set of C preprocessor flags that are to be used on the compiler command line for locating the associated shared library header files. These options indicate, of course, that the top-level include and common directories should be searched by the pre-processor for header files referenced in the source code. In fact, here's the new source file, jup_print.c:

*libjup/jup_print.c*

{% highlight shell %}
#include <libjupiter.h>
#include <jupcommon.h>

int jupiter_print(char * name)
{
   print_routine(name);
}
{% endhighlight %}

I need to include the shared library header file for access to the jupiter_print function's public prototype. This leads us to another general software engineering principle. I've heard it called by many names, but the one I tend to use the most is "The DRY Principle", which is an acronym that stands for Don't Repeat Yourself. C function prototypes are very useful, because when used correctly, they enforce the fact that the public's view of a function is identical to the package maintainer's view. So often, I've seen source code for a function where the source file doesn't include the header containing the public prototype for the function. It's easy to make a small change in the function or prototype, and then not duplicate it in the other location--unless you've included the public header file within the source file containing the function. Then, the compiler catches all such mistakes.

I need the static library header file because I call its function from within my public library function. Note also that I placed the public header file first--there's a good reason for this. Here is another general principle: by placing the public header file first in the source file, I can allow the compiler to check that the use of this header file doesn't depend on any other files in the project.

If the public header file has a hidden dependency on some construct (a typedef, structure or pre-processor definition) defined in internal headers like jupcommon.h, and if I include the public header file after jupcommon.h, then the dependency would be hidden by the fact that the required construct is already available in the translation unit when the compiler begins to process the public header file.

Next, I'll modify the jupiter application's main function so that it calls into the shared library instead of calling into the common static library:

*src/main.c*

{% highlight shell %}
#include <libjupiter.h>

int main(int argc, char * argv[])
{
    jupiter_print(argv[0]);
    return 0;
}
{% endhighlight %}

Here, I've changed the print function from print_routine, found in the static library, to jupiter_print, as provided by the new shared library. I've also changed the header file included at the top from libjupcommon.h to libjupiter.h.

My choices of names for the public function and header file were arbitrary, but based on a desire to provide a clean, rational and informational public interface. The name libjupiter.h very clearly indicates that this header file provides the public interface for the libjupiter.so shared library. I try to name library interface functions in such a way that they are clearly part of an interface. How you choose to name your public interface members--files, functions, structures, typedefs, pre-processor definitions, global data, etc--is up to you, but you should consider using a similar philosophy. Remember, the goal is to provide a great end-user experience.

Finally, the src/Makefile.am file must also be modified to use my new shared library, rather than the libjupcommon.a static library:

*src/Makefile.am*

{% highlight shell %}
bin_PROGRAMS = jupiter
jupiter_SOURCES = main.c
jupiter_CPPFLAGS = -I$(top_srcdir)/include
jupiter_LDADD = ../libjup/libjupiter.la
...
{% endhighlight %}

In this file, I've changed the jupiter_CPPFLAGS variable so that it now refers to the new include directory, rather than the common directory. I've also changed the jupiter_LDADD variable so that it refers to the new Libtool shared library object, rather than the libjupcommon.a static library. All else remains the same. Note that these changes are both obvious and simple. The syntax for referring to a Libtool library is identical to that referring to an older static library. Only the library extension is different. The Libtool library extension, .la stands for "libtool archive".

Take a step back for a moment: Do I actually need to make this change? No, of course not. The jupiter application will continue to work just fine the way it was originally set up--linking the code for the static library's print_routine directly into the application works equally well to calling the new shared library routine (which ultimately contains the same code). There is slightly more overhead in calling a shared library routine because of the extra level of indirection when calling though a jump table.

In a real project, you might actually leave it the way it was. Why? Because both public entry points, main and jupiter_print call exactly the same function (print_routine) in libjupcommon.a, so the functionality is identical. Why add the (slight) overhead of a call through the public interface? Well, you can take advantage of shared code. By using the shared library function, you're not duplicating code--either on disk, or in memory. Again, the DRY principle at work.

In this situation, you might now consider simply moving the code from the static library into the shared library, thereby removing the need for the static library entirely. Again, I'm going to beg your indulgence with my contrived example. In a more complex project, I might very well have a need for this sort of configuration, as such common code is often gathered together into static convenience libraries. Often, only a portion of this code is reused in shared libraries. I'm going to leave it the way it is for the sake of its educational value.

**Reconfigure and build**

Let me summarize where the project stands at this point. Since I've added a major new component to my project build system (Libtool), I'll add the -i option to the `autoreconf` command, just in case new files need to be installed:

{% highlight shell %}
$ autoreconf -i
$ ./configure
...
checking for ld used by gcc...
checking if the linker ... is GNU ld... yes
checking for BSD- or MS-compatible name lister...
checking the name lister ... interface...
checking whether ln -s works... yes
checking the maximum length of command line...
checking whether the shell understands some XSI...
checking whether the shell understands "+="...
checking for ...ld option to reload object files...
checking how to recognize dependent libraries...
checking for ar... ar
checking for strip... strip
checking for ranlib... ranlib
checking command to parse ...nm -B output...
...
checking for dlfcn.h... yes
checking for objdir... .libs
checking if gcc supports -fno-rtti...
checking for gcc option to produce PIC... -fPIC
checking if gcc PIC flag -fPIC -DPIC works...
checking if gcc static flag -static works...
checking if gcc supports -c -o file.o... yes
checking if gcc supports -c -o file.o... yes
checking whether ... linker ... supports shared...
checking whether -lc should be explicitly linked...
checking dynamic linker characteristics...
checking how to hardcode library paths...
checking whether stripping libraries is possible...
checking whether to build shared libraries...
checking whether to build static libraries...
...
{% endhighlight %}

Thee first noteworthy item here is that Libtool adds significant overhead to the configuration process. I've only shown the output lines here that are new since I added Libtool. All I've added to the configure.ac script is the reference to the LT_INIT macro, and I've nearly doubled my configure script output. This should give you some idea of the number of system characteristics that must be examined to create portable shared libraries. Libtool does a lot of the work for you.

*NOTE: In the following output examples, I've wrapped long output lines to fit publication formatting, and I've added blank lines between output lines for readability. I've also removed some unnecessary text, such as long directory names--both to increase readability and to shorten line lengths.*

{% highlight shell %}
$ make
...
Making all in libjup
make[2]: Entering directory `.../libjup'
/bin/sh ../libtool --tag=CC   --mode=compile gcc -DHAVE_CONFIG_H -I. -I../../libjup -I..
-I../../include -I../../common  -g -O2 -MT libjupiter_la-jup_print.lo -MD -MP -MF
.deps/libjupiter_la-jup_print.Tpo -c -o libjupiter_la-jup_print.lo `test -f 'jup_print.c' || echo '../../libjup/'`jup_print.c

libtool: compile:  gcc -DHAVE_CONFIG_H -I. -I../../libjup -I.. -I../../include
-I../../common -g -O2 -MT libjupiter_la-jup_print.lo -MD -MP -MF
.deps/libjupiter_la-jup_print.Tpo -c ../../libjup/jup_print.c  -fPIC -DPIC
-o .libs/libjupiter_la-jup_print.o

libtool: compile:  gcc -DHAVE_CONFIG_H -I. -I../../libjup -I.. -I../../include
-I../../common -g -O2 -MT libjupiter_la-jup_print.lo -MD -MP -MF
.deps/libjupiter_la-jup_print.Tpo -c ../../libjup/jup_print.c
-o libjupiter_la-jup_print.o >/dev/null 2>&1

mv -f .deps/libjupiter_la-jup_print.Tpo .deps/libjupiter_la-jup_print.Plo

/bin/sh ../libtool --tag=CC   --mode=link gcc  -g -O2 ../common/libjupcommon.a  -o libjupiter.la
-rpath /usr/local/lib libjupiter_la-jup_print.lo -lpthread 

*** Warning: Linking ... libjupiter.la against the
*** static library libjupcommon.a is not portable!



libtool: link: gcc -shared .libs/libjupiter_la-jup_print.o
../common/libjupcommon.a -lpthread -Wl,-soname -Wl,libjupiter.so.0
-o .libs/libjupiter.so.0.0.0

.../ld: ../common/libjupcommon.a(print.o):
    relocation R_X86_64_32 against 'a local symbol'
    can not be used when making a shared object;
    recompile with -fPIC

../common/libjupcommon.a: could not read symbols:
    Bad value

collect2: ld returned 1 exit status
    make[2]: *** [libjupiter.la] Error 1
    ...
{% endhighlight %}

That wasn't a very pleasant experience! It appears that I have some errors to fix. I'll take them one at a time, from top to bottom.

The first point of interest is that the libtool script is being called with a --mode=compile option, which causes libtool to act as a wrapper script around a somewhat modified version of a standard gcc command line. You can see the effects of this statement in the next two compiler command lines. Two compiler commands? That's right. It appears that libtool is causing the compile operation to occur twice.

A careful examination of the differences between these two command lines shows that the first compiler command is using two additional flags: "-fPIC" and "-DPIC". The first line also appears to be directing the output file to a ".libs" subdirectory, whereas, the second line is saving it in the current directory. Finally, both the STDOUT and STDERR output is redirected to /dev/null in the second line.

This double-compile "feature" has caused a fair amount of anxiety on the Libtool mailing list over the years. Mostly, this is due to a lack of understanding of what it is that Libtool is trying to do, and why it's necessary. Using various configure script command line options provided by Libtool, you can force a single compilation, but doing so brings with it a certain loss of functionality, which I'll explain here shortly.

The next line renames the dependency file from *.Tpo to *.Plo. Dependency files contain make rules that declare dependencies between source files and referenced header files. These are generated by the C preprocessor when the -MT compiler option is used. (And what better tool to know about such references than the one that actually processes them!) They're then included in makefiles so that the make utility can properly recompile a source file, if one or more of its include dependencies have been modified since the last build. This is not really germane to an examination of Libtool, so I'll not go into any more detail here, but check the GNU Make manual for more information. The point is that one Libtool command may (and often does) execute a group of shell commands.

The next line is another call to the libtool script, this time using the --mode=link option. This option generates a call to execute the compiler in "link" mode, passing all of the libraries and linker options specified in the Makefile.am file.

And finally, here is first problem--a portablity warning about linking a shared library against a static library. Specifically, this warning is about linking a Libtool shared library against a *non-Libtool* static library. You'll soon begin to see why this might be a problem. Notice also that this is not an error. Were it not for additional errors we'll encounter later, this library would be built in spite of this warning.

After the portability warning, libtool attempts to link the requested objects together into a shared library named "libjupiter.so.0.0.0". But here the script runs into the real problem--a linker error indicating that somewhere from within libjupcommon.a--and more specifically within print.o--an Intel object relocation cannot be performed because the original source file (print.c) was apparently not compiled correctly. The linker is kind enough to tell me exactly what I need to do to fix the problem. It indicates that I need to compile the source code using a "-fPIC" compiler option.

Now, if you were to encounter this error and didn't know anything about the "-fPIC" option, then you'd be wise at this point to open the man page for gcc and study it, before willy-nilly inserting compiler or linker options until the warning or error disappears, as many inexperienced programmers are wont to do. Software engineers should understand the meaning and nuances of every command line option used by the tools in their projects' build systems. Why? Because otherwise they don't really know what they have when their build completes. It may work the way it should--but if it does, it's simply by luck, rather than by design. Good engineers know their tools, and the best way to learn is to study error messages and their fixes until the problem is well-understood, before moving on.

**So what is "PIC" code?**

When operating systems create new process address spaces, they always load the executable images at the same memory address. This magic address is system-specific. Compilers and linkers know this, and they know what that address is on a given system. Therefore, when they generate internal references to function calls, for example, they can generate those references as absolute addresses. If you were somehow able to load the executable at a different location in memory, it would simply not work properly, because the absolute addresses within the code would be incorrect. At the very least, the program would crash when the it jumped to the wrong location during a function call.

Consider Figure 1 below for a moment. Given a system whose magic executable load address is 0x10000000, this diagram depicts two process address spaces within that system. In the process on the left, an executable image is loaded correctly at address 0x10000000. At some point in the code a "jmp" instruction tells the processor to transfer control to the absolute address 0x10001000, where it continues executing instructions in another area of the program. In the process on the right, the program is loaded incorrectly at address 0x20000000. When that same branch instruction is encountered, the processor jumps to address 0x10001000, because that address is hard-coded into the program image. This, of course, fails--often spectacularly by crashing, but sometimes with more subtle and dastardly ramifications.

<h6><center>Figure 1: Absolute addressing in executable images</center></h6>
![exe_load](/assets/201612/exe_load.png)

That's how things work for program images. However, when a shared library is built for certain types of hardware (Intel x86 and x8664 included), the address at which the library will be loaded within a process address space cannot be known by either the compiler or the linker beforehand. This is because many libraries may be loaded into any given process, and the order in which they are loaded depends on how the _executable is built, not the library. Furthermore, who's to say which library owns location "A", and which one owns location "B"? The fact is, libraries may be loaded anywhere into a process where there is space for it at the time it's loaded. Only the operating system loader knows where it will finally reside--and then only just before it's actually loaded.

As a result, shared libraries can only be built from a special class of object file called "PIC" objects. PIC is an acronym which stands for "Position-Independent Code", and implies that references in the object code are not absolute, but relative. When the "-fPIC" option is used on the compiler command line, the compiler will use somewhat less efficient relative addressing in branching instructions. Such position-independent code may be loaded anywhere.

The diagram in Figure 2 below graphically depicts the concept of relative addressing, as used when generating PIC objects. When using relative addressing, regardless of where the image is loaded, addresses work correctly because they're always encoded relative to the current instruction pointer. In Figure 2, the diagrams indicate a shared library loaded at the same addresses, 0x10000000 and 0x20000000. In both cases, the DOLLAR SIGN ($) used in the JMP instruction represents the current instruction pointer (IP), so "$ + 0xC74" tells the processor that it should jump to the instruction starting 0xC74 bytes ahead of the current instruction pointer position.

<h6><center>Figure 2: Relative addressing in shared library images.</center></h6>
![lib_load](/assets/201612/lib_load.png)

There are various nuances to generating and using position-independent code, and you should become familiar with them all before using them, so that you can choose the option that is most appropriate for your situation. For example, the GNU C compiler also supports a "-fpic" option (lowercase), which uses a slightly quicker, but more limited mechanism to accomplish relocatable object code. Wikipedia has a very informative page on position-independent code (although I find its treatment of Windows DLLs to be somewhat less than accurate).

**Fixing the jupiter "PIC" problem**

From what you now understand, one way to fix my linker error is to add the "-fPIC" option to the compiler command line for the source files that comprise the libjupcommon.a static library. Try that:

*common/Makefile.am*

{% highlight shell %}
noinst_LIBRARIES = libjupcommon.a
libjupcommon_a_SOURCES = jupcommon.h print.c
libjupcommon_a_CFLAGS = -fPIC
{% endhighlight %}

And now I'll try the build again:

{% highlight shell %}
$ autoreconf
$ make
...
gcc -DHAVE_CONFIG_H -I. -I../../common -I.. -fPIC -g -O2 -MT libjupcommon_a-print.o -MD -MP -MF
.deps/libjupcommon_a-print.Tpo -c -o libjupcommon_a-print.o `test -f 'print.c' || echo '../../common/'`print.c
...
/bin/sh ../libtool --tag=CC --mode=link gcc -g -O2 ../common/libjupcommon.a -o libjupiter.la
-rpath /usr/local/lib libjupiter_la-jup_print.lo -lpthread 

*** Warning: Linking ... libjupiter.la against the
*** static library libjupcommon.a is not portable!

libtool: link: gcc -shared .libs/libjupiter_la-jup_print.o
../common/libjupcommon.a -lpthread -Wl,-soname -Wl,libjupiter.so.0 -o .libs/libjupiter.so.0.0.0

libtool: link: (cd .libs && rm -f libjupiter.so.0 && ln -s libjupiter.so.0.0.0 libjupiter.so.0)

libtool: link: (cd .libs && rm -f libjupiter.so && ln -s libjupiter.so.0.0.0 libjupiter.so)

libtool: link: ar cru .libs/libjupiter.a ../common/libjupcommon.a libjupiter_la-jup_print.o

libtool: link: ranlib .libs/libjupiter.a

libtool: link: (cd .libs && rm -f libjupiter.la && ln -s ../libjupiter.la libjupiter.la)
...
{% endhighlight %}

I now have a shared library, built properly with position-independent code, as per system requirements. However, I still have that strange warning about the portability of linking a Libtool library against a static library. The problem here is not in what I'm doing, but rather in the way in which I'm doing it. You see, the concept of PIC does not apply to all hardware architectures. Some CPUs don't support any form of absolute addressing in their instruction sets. As a result, native compilers for these platforms don't support a -fPIC option--it has no meaning for them.

If I tried (for example) to compile my code on an IBM RS/6000 system using the native IBM compiler, it would "hiccup" when it came to the -fPIC option because it doesn't make sense to support such an option on a system where all code is automatically generated as position-independent code. One way I could get around this problem would be to make the -fPIC option conditional in my Makefile.am file, based on the type of the target system, and the tools I'm using. But that's exactly the sort of problem that Libtool was designed to address! I'd have to account for all of the different Libtool target system types and tool sets in order to handle the entire set of conditions that Libtool already handles.

The way around this portability problem then is to let Libtool generate my static library as well. Libtool makes a distinction between static libraries that are installed as part of a developer's kit, and static libraries used only internally within a project. It calls such internal static libraries "convenience" libraries, and whether or not a convenience library is generated depends on the prefix used with the LTLIBRARIES primary. If the noinst prefix is used, then Libtool assumes that I want a convenience library because there's no point in generating a shared library that will never be installed. Thus, convenience libraries are always generated as static archives.

The reason for distinguishing between convenience libraries and other forms of static library is that convenience libraries are always built, whereas non-convenience static libraries are only built if the --enable-static option is specified on the configure command line (or conversely, if the --disable-static option is *not* specified).

**Customizing Libtool with LT_INIT options**

Default values for enabling or disabling static and shared libraries can be specified in the argument list passed into the LT_INIT macro in the configure.ac script. Have a quick look at the LT_INIT macrom which may be used with or without arguments. LT_INIT accepts a single argument, which is a white-space separated list of key words. The following key words are valid:

- dlopen -- Enable checking for dlopen support. This option should be used if the package makes use of the -dlopen and -dlpreopen Libtool flags, otherwise Libtool will assume that the system does not support dl-opening. This option is actually assumed by default.
- 
- disable-fast-install -- Change the default behavior for LT_INIT to disable optimization for fast installation. The user may still override this default, depending on platform support, by specifying --enable-fast-install to configure.
- 
- shared -- Change the default behavior for LT_INIT to enable shared libraries. This is the default on all systems where Libtool knows how to create shared libraries. The user may still override this default by specifying --disable-shared to configure.
- 
- disable-shared -- Change the default behavior for LT_INIT to disable shared libraries. The user may still override this default by specifying --enable-shared to configure.
- 
- static -- Change the default behavior for LT_INIT to enable static libraries. This is the default on all systems where shared libraries have been disabled for some reason, and on most systems where shared libraries have been enabled. If shared libraries are enabled, the user may still override this default by specifying --disable-static to configure.
- 
- disable-static -- Change the default behavior for LT_INIT to disable static libraries. The user may still override this default by specifying --enable-static to configure.
- 
- pic-only -- Change the default behavior for libtool to try to use only PIC objects. The user may still override this default by specifying --without-pic to configure.
- 
- no-pic -- Change the default behavior of libtool to try to use only non-PIC objects. The user may still override this default by specifying --with-pic to configure.

*NOTE: I've omitted the description for the win32-dll option, because it doesn't apply to this book.*

Now, back to the Jupiter project. The conversion from an older static library to a new Libtool convenience library is simple enough--all I have to do is add LT to the primary name and remove the -fPIC option and the associated variable, as there were no other options being used in that variable. Note also that I've changed the library extension from .a to .la:

*common/Makefile.am*

{% highlight shell %}
noinst_LTLIBRARIES = libjupcommon.la
libjupcommon_la_SOURCES = jupcommon.h print.c
{% endhighlight %}

*libjup/Makefile.am*

{% highlight shell %}
...
libjupiter_la_LIBADD = ../common/libjupcommon.la
...
{% endhighlight %}

Now when I try to build, here's what I get:

{% highlight shell %}
$ autoreconf
$ ./configure
...
$ make
...
/bin/sh ../libtool --tag=CC --mode=compile gcc -DHAVE_CONFIG_H -I. -I../../common -I..
-g -O2 -MT print.lo -MD -MP -MF .deps/print.Tpo -c -o print.lo ../../common/print.c

libtool: compile: gcc -DHAVE_CONFIG_H -I. -I../../common -I.. -g -O2 -MT print.lo -MD -MP
-MF .deps/print.Tpo -c ../../common/print.c -fPIC -DPIC -o .libs/print.o 

...

/bin/sh ../libtool --tag=CC --mode=link gcc -g -O2 -o libjupcommon.la print.lo -lpthread

libtool: link: ar cru .libs/libjupcommon.a .libs/print.o

...

/bin/sh ../libtool --tag=CC --mode=link gcc -g -O2 ../common/libjupcommon.la -o libjupiter.la
-rpath /usr/local/lib libjupiter_la-jup_print.lo -lpthread 

libtool: link: gcc -shared .libs/libjupiter_la-jup_print.o -Wl,--whole-archive
../common/.libs/libjupcommon.a -Wl,--no-whole-archive -lpthread -Wl,-soname
-Wl,libjupiter.so.0 -o .libs/libjupiter.so.0.0.0
...
{% endhighlight %}

You can see that the common library is now built as a static convenience library because the ar utility is used to build libjupcommon.a. Libtool also seems to be building files with new and different extensions. A closer look will discover extensions such as .lo and .la. If you take a closer look at these files, you'll find that they're actually descriptive text files containing object and library meta data. Take a look at the common/libjupcommon.la file:

*common/libjupcommon.la*

{% highlight shell %}
# libjupcommon.la - a libtool library file
# Generated by ltmain.sh (GNU libtool) 2.2
#
# Please DO NOT delete this file!
# It is necessary for linking the library.

# The name that we can dlopen(3).
dlname=''

# Names of this library.
library_names=''

# The name of the static archive.
old_library='libjupcommon.a'

# Linker flags that can not go in dependency_libs.
inherited_linker_flags=''

# Libraries that this one depends upon.
dependency_libs=' -lpthread'
...
{% endhighlight %}

The various fields in these files help the linker--or rather the libtool wrapper script--to determine certain options that would otherwise have to be remembered by the developer, and then passed on the command line to the linker. For instance, the library's shared and static names are remembered here, as well as any other library dependencies required by these libraries. In this library, for example, I can see that libjupcommon.a depends on the pthread library. But, using Libtool, I don't have to pass a -lpthread option on the libtool command line because libtool can detect in this meta data file that the linker will need this, so it passes the option for me.

Making these files human-readable was a minor stroke of genius, as they can tell me a lot about my Libtool libraries, at a glance. These files are designed to be installed with their associated binaries, and in fact, the `make install` rules generated by Automake for Libtool libraries do just this.

**The Libtool library versioning scheme**
If you've spent any time at all working at the Linux command prompt, then you'll certainly recognize this series of executable and link names.

*NOTE: There's nothing special about libz--I am merely using this library as a common example:*

{% highlight shell %}
$ ls -dal /lib/libz*
... /lib/libz.so.1 -> libz.so.1.2.3
... /lib/libz.so.1.2 -> libz.so.1.2.3
... /lib/libz.so.1.2.3
{% endhighlight %}

If you've ever wondered what this means, then read on. Libtool provides a versioning scheme for shared libraries that has become prevalent in the Linux world. Other operating systems use different versioning schemes for shared libraries, but the one defined by Libtool has become so popular that people often associate it with Linux, rather than with Libtool. This is not entirely an unfair assessment because the Linux loader honors this scheme to a certain degree. But to be completely fair, it's Libtool that should be given the credit for this versioning scheme.

One interesting aspect of this scheme is that, if not understood properly, people can easily mis-use or abuse the system without intending to. People who don't understand this system tend to think of the numeric values as major, minor and revision, when in fact, these values have very specific meaning to the operating system loader, and must be updated properly for each new library version in order to keep from confusing the loader.

I remember a meeting I had at work one day several years ago with my company's corporate versioning committee. This committee's job was to come up with software versioning policy for the company as a whole. They wanted us to ensure that the version numbers incorporated into our shared library names were in alignment with the corporate software versioning standard. It took me the better part of a day to convince them that a shared library version was not related to a product version in any way, nor should such a relationship be established or enforced by them or anyone else.

Here's why. The version number on a shared library is not really a library version, but rather an interface version. The interface I'm referring to here is the application programming interface (API) presented by a library to the potential user--a programmer wishing to call functions in the interface. As the GNU Libtool manual points out, a program has a single well-defined entry point (usually called main, in the C language). But a shared library has multiple entry points that are generally not standardized in a widely understood manner. This makes it much more difficult to determine if a particular version of a library is "interface-compatible" with another version of the same library.

*NOTE: The concept of "interface" goes much deeper in shared library versioning, referring to all aspects of a shared library's connections with the outside world. These connections include files and file formats, network connections and wire data formats, IPC channels and protocols, etc. When versioning a new public release of a shared library, all aspects of the library's interactions with the world should be taken into account.*

**Microsoft DLL versioning**

Consider Microsoft Windows Dynamic Link Libraries (DLLs). These are shared libraries in every sense of the word. They provide a proper application programming interface. But unfortunately, Microsoft has in the past provided no integrated DLL interface versioning scheme. As a result, Windows developers have often refered to DLL versioning issues (tongue-in-cheek, I'm sure) as "DLL hell".

As a fix to this problem, on Windows systems, DLLs can be installed into the same directory as the program that uses them, and the Windows operating system loader will always attempt to use the local copy first before searching for a copy in the system path. This alleviates a part of the problem because a specific version of the library can be installed with the package that requires it.

While this is a fair solution it's not a really good solution, because one of the major benefits of shared libraries is that they can be shared--both on disk and in memory. If every application has its own copy of a different version of the library, then this benefit of shared libraries is lost--both on disk and in memory.

Since the introduction of this partial solution, Microsoft hasn't paid much attention to DLL sharing efficiency issues. The reasons for this include both a cavalier attitude regarding the cost of disk space and RAM, and a technical issue regarding the implementation of Windows dynamic link libraries. Instead of generating position-independent code, Microsoft system architects chose to link DLL's with a specific base address, and then list all absolute address references in a base table in the image header. When a DLL can't be loaded at the required base address (because of a conflict with another DLL), then the loader "rebases" the DLL by picking a new base address and changing all of the absolute addresses referred to in the base table. Whenever a DLL is rebased in this manner, it can only be shared with processes that happen to rebase the DLL to the same address. The odds of accidentally encountering such a scenario--especially among applications with many DLL components--are pretty slim.

Recently, Microsoft invented the concept of the "Side-by-Side Cache" (sometimes referred to as "SxS"), which allows developers to associate a unique identification value (a GUID, in fact) with a particular version of a DLL installed in a system location. This location is named by the DLL name and version identifier. Applications built against SxS-versioned libraries have meta data stored in their executable headers that indicate the particularly versioned DLLs that they require. If the right version is found (by newer OS loaders) in the SxS cache, then they load it. Based on policy in the meta data, they can then revert to the older scheme of looking for a local and then a global copy of the DLL. This is a vast improvement over earlier solutions--providing a very flexible versioning system.

Given the fact that DLLs use the rebasing technique, as opposed to PIC code, the side-by-side cache is still a fairly benign improvement with respect to applications that manage dozens of shared libraries. SxS is really intended for system libraries that many applications are likely to consume. These are generally "based" at different addresses, so that the odds of clashing (and thus rebasing) are decreased.

Regardless, the entire based approach to shared libraries has the major drawback that the program address space may become fairly fragmented, as randomly chosen base addresses are honored throughout a 32-bit address space by the system loader. 64-bit addressing helps tremendously in this area, so you may find the side-by-side cache to be much more useful on 64-bit Windows systems.

Linux and other Unix-like systems that support shared libraries manage interface versions using the Libtool versioning scheme. In this scheme, shared libraries are said to support a range of interface versions, each identified by a unique integer value. If any aspect of an interface changes in any way between public releases, then it can no longer be considered the same interface. It becomes a new interface, identified by a new integer interface value. To make the interface versioning process comprehensible to the human mind, each public release of a library wherein the interface has changed simply acquires the next consecutive interface version number. Thus, a given shared library may support versions 2-5 of an interface.

Libtool shared libraries follow a naming convention that encodes the interface range supported by a particular shared library. A shared library named libname.so.0.0.0 contains the library interface version number, 0.0.0. these three values are respectively called the library interface current, revision and age values.

The current value represents the current interface version number. This is the value that changes each time a new interface version must be declared, because the interface has changed in any way since the last public release of the library. The first interface in a library is given a version number of "0", by popular convention.

Consider a shared library wherein the developer has added a new function to the set of functions exposed by this library since the last public release. The interface can't be considered the same in this new version as it was in the previous version because there's one additional function. Thus, it's current number must be increased from "0" to "1".

The age value represents the number of back-versions supported by the shared library. In mathematical terms, the library is said to support the interface range, current - age through current. In the example I just gave, a new function was added to the library, so the interface presented in this version of the library is not the same as that presented in the previous version. However, the previous version is still fully supported because the previous interface is a proper subset of the current interface. Thus, this library could conceivably be named "libname.so.1.0.1", where the range of supported interfaces is 1 - 1 (or 0) through 1, inclusive.

The revision value merely represents a serial revision of the current interface. That is, if no changes are made to a shared library's interface between releases--perhaps an internal function was optimized--then the library name should change in some manner, but both the current and age values would be the same, as the interface has not changed. The revision value is incremented to reflect the fact that this is a new release of the same interface. If two libraries exist on a system with the same name, and the same current and age values, then the operating system loader will always select the library with the higher revision value.

To simplify the release process for shared libraries, the GNU Libtool manual provides an algorithm that should be followed step-by-step for each new version of a library that is about to be publically released. I'll reproduce the algorithm verbatim here for your information:

<p style="padding-left:15px">
1. Start with version information of 0:0:0 for each libtool library. [This is done automatically by simply omitting the -version option from the list of linker flags passed to the libtool script.]
</p>
<p style="padding-left:15px">
2. Update the version information only immediately before a public release of your software. More frequent updates are unnecessary, and only guarantee that the current interface number gets larger faster.
</p>
<p style="padding-left:15px">
3. If the library source code has changed at all since the last update, then increment revision (c:r:a becomes c:r+1:a).
</p>
<p style="padding-left:15px">
4. If any interfaces [exported functions or data] have been added, removed, or changed since the last update, increment current, and set revision to 0.
</p>
<p style="padding-left:15px">
5. If any interfaces have been added since the last public release, then increment age.
</p>
<p style="padding-left:15px">
6. If any interfaces have been removed since the last public release, then set age to 0.
</p>

Keep in mind that this is an algorithm, and as such it is designed to be followed step by step, as opposed to jumping directly to the steps that appear to apply to your case. For example, if you removed any API functions from your library since the last release, you would not simply jump to the last step and set age to zero. Rather, you would follow all of the steps properly until you reached the last step, and then set age to zero.

In greater detail: assume that this is the second release of a library, and that the first release was named libexample.so.0.0.0, and that one new function was added to the API during this development cycle, and one old function was deleted. The effect on this release of the library would be as follows:

<p style="padding-left:15px">
1. (n/a)
</p>
<p style="padding-left:15px">
2. (n/a)
</p>
<p style="padding-left:15px">
3. libexample.so.0.0.0 -> libexample.so.0.1.0 (library source was changed)
</p>
<p style="padding-left:15px">
4. libexample.so.0.1.0 -> libexample.so.1.0.0 (library interface was modified)
</p>
<p style="padding-left:15px">
5. libexample.so.1.0.0 -> libexample.so.1.0.1 (one new function was added)
</p>
<p style="padding-left:15px">
6. libexample.so.1.0.1 -> libexample.so.1.0.0 (one old function was removed)
</p>

Why all the "hoop jumping"? Because, as I alluded to earlier, the versioning scheme is honored by the linker and the operating system loader. When the linker creates the library name table in an executable image header, it writes the versions of the libraries linked to the application along side of each entry in this table. When the loader searches for a matching library, it looks for the latest version of the library required by the executable. If the application was linked with version 0.0.0 of a particular library, but the user only has version 1.0.1 installed, the system will load it and use it because it's current and age values indicate that it supports the required version (0).

Note also that libname.so.0.0.0 can coexist in the same directory as libname.so.1.0.0 without any problem. Programs that need the earlier version (which supports only the later interface because of the deleted function) will properly and automatically have it loaded into their process address space, just as will programs that require the later version properly have the "1.0.0" version loaded.

One more point regarding interface versioning. Once you fully understand Libtool versioning, you'll find that even the above algorithm does not cover all possible interface modification scenarios. Consider, for example, version 0.0.0 of a shared library that you maintain. Now, assume you add a new function to the interface for the next public release. This second release is properly named version 1.0.1, because the library supports both interfaces 0 and 1. Just before the third release of the library, you realize that you didn't really need that new function after all, and so you remove it. Assume also that this is the only change made to the library interface in this release. The above algorithm would have this release named version 2.0.0. But in fact, you've merely removed the second interface, and are now presenting the original interface once again. Technically, this library should be properly named version 0.1.0, as it presents a second release of version 0 of the shared library interface.

**Using libltdl to dlopen a shared library**

Once again, I'm going to have to add some functionality to the Jupiter project in order to illustrate the concepts of this section. The goal here is to create a plug-in interface that the jupiter application can use to enhance the output.

**Necessary infrastructure**

Currently, jupiter prints "Hello, from jupiter!". (Actually, the name printed is more likely at this point to be a long ugly path containing some Libtool directory garbage and some derivation of the name "jupiter", but just pretend it prints "jupiter" for now.) I'm going to add an additional parameter to the common static library print routine, named "salutation". This parameter will also be a character string reference, and will contain the leading word or phrase--the salutation, as it were.

Here are the changes I have to make to the files in the common directory:

*common/print.c*

{% highlight shell %}
...
static void * print_it(void * data)
{
    char ** strings = (char **)data;
    printf("%s from %s!\n", strings[0], strings[1]);
    return 0;
}

int print_routine(char * salutation, char * name)
{
    char * strings[] = {salutation, name};
#if ASYNC_EXEC
    pthread_t tid;
    pthread_create(&tid, 0, print_it, strings);
    pthread_join(tid, 0);
#else
    print_it(strings);
#endif
    return 0;
}
{% endhighlight %}

*common/jupcommon.h*

{% highlight shell %}
#ifndef JUPCOMMON_H_INCLUDED
#define JUPCOMMON_H_INCLUDED

int print_routine(char * salutation, char * name);

#endif  /* JUPCOMMON_H_INCLUDED */
{% endhighlight %}

And here are the changes I need to make to the files in the libjup and include directories:

*libjup/jup_print.c*

{% highlight shell %}
...
int jupiter_print(char * salutation, char * name)
{
    print_routine(salutation, name);
}
{% endhighlight %}

*include/libjupiter.h*

{% highlight shell %}
...
int jupiter_print(char * salutation, char * name);
...
{% endhighlight %}

And finally, here are the changes I need to make to main.c in the src directory:

*src/main.c*

{% highlight shell %}
...
#define DEFAULT_SALUTATION "Hello"

int main(int argc, char * argv[])
{
    char * salutation = DEFAULT_SALUTATION;
    jupiter_print(salutation, argv[0]);
    return 0;
}
{% endhighlight %}

To be clear, all I've really done here is parameterize the salutation in the print routines. That way, I can indicate from main what salutation I'd like to use. I've set the default salutation to "Hello", so that nothing will have changed from the user's perspective. The overall effect of these changes was benign. Note also that these are all source code changes. I've made no changes to the build system.

**Adding a plug-in interface**

Now, I can begin to discuss adding a plug-in interface to Jupiter. I'd like to make it possible to change the salutation displayed by simply changing a plug-in module. The code and build system changes required to add this functionality will be limited here to the src directory, and subdirectories thereof.

First, I need to define the actual plug-in interface. I'll do this by creating a new private header file in the src directory, called module.h:

*src/module.h*

{% highlight shell %}
#ifndef MODULE_H_INCLUDED
#define MODULE_H_INCLUDED

#define GET_SALUTATION_SYM "get_salutation"

typedef char * get_salutation_t(void);
char * get_salutation(void);

#endif  /* MODULE_H_INCLUDED */
{% endhighlight %}

There are a number of interesting points about this header file. First, the preprocessor definition, GET_SALUTATION_SYM. This string represents the name of the function you need to import from the plug-in module. I like to define these in the header file, so that all of the information that needs to be reconciled co-exists in one place. In this case, the symbol name, the function type definition, and the function prototype must all be in alignment. While I could have simply allowed the caller to specify the string, defining the symbol name here allows me to change it later if I need to. As long as the caller used the definition I provided, s/he should be unaffected by a name change (of course, s/he'll have to recompile).

Another interesting point is the type definition: why should I provide one? If I don't, the user is going to have to invent one, or else use a complex type cast on the return value of the dlsym function. I provide it here for consistency. Finally, look at the function prototype. This isn't so much for the caller, as it is for the module itself. Modules providing this function should include this header file, so that the compiler can catch potential mis-spellings of the function name. Since all of this information must be in agreement, I simply define it all here together.

**Doing it the "old-fashioned" way**

For this first attempt, I'll use the dlopen/dlsym/dlclose interface provided by the Solaris, BSD and Linux libdl.so library. Then, in the next section, I'll convert this code over to the Libtool ltdl interface. To do this right, I need to add checks to the configure.ac script to look for both the libdl library and the dlfcn.h header file:

*configure.ac*

{% highlight shell %}
...
# Checks for header files (2).
AC_CHECK_HEADERS([stdlib.h dlfcn.h])

# Checks for libraries.
# Checks for typedefs, structures, and compiler...
# Checks for library functions.
AC_SEARCH_LIBS([dlopen], [dl])
    ...
    echo \
    "-------------------------------------------------
    ${PACKAGE_NAME} Version ${PACKAGE_VERSION}

Prefix: '${prefix}'.
Compiler: '${CC} ${CFLAGS} ${CPPFLAGS}'
Libraries: '${LIBS}'
...
{% endhighlight %}

These changes consist of adding the dlfcn.h header file to the list of files passed to the AC_CHECK_HEADERS macro, and adding a check for the dlopen function in the dl library. Note here that the AC_SEARCH_LIBS macro searches a list of libraries for a function, so this call goes under the section entitled, "Checks for library functions.", rather than the one entitled, "Checks for libraries."

To help me see which libraries I'm actually linking against, I've also added a line to the echo statement at the end of the file. The "Libraries:" line displays the contents of the LIBS variable, which is modified by the AC_SEARCH_LIBS macro.

*NOTE: The LT_INIT macro actually already checks for the existence of the dlfcn.h header file, but I do it here explicitly, so it's obvious to observers that I wish to use this header file myself. This is a good rule of thumb to follow, as long as it doesn't negatively affect performance too much. In this case, I felt it was well worth the extra check. Besides that, the results of the check performed by LT_INIT is cached by autom4te, so it has little effect anyway.*

Now it's time to actually add a new module. This requires several changes, so I'll make them all here now in the following command sequence:

{% highlight shell %}
$ cd src
$ mkdir -p modules/hithere
$ vi Makefile.am
SUBDIRS = modules
...
$ echo "SUBDIRS=hithere" > modules/Makefile.am
$ cd modules/hithere
$ echo "pkglib_LTLIBRARIES = hithere.la hithere_la_SOURCES = hithere.c \
        hithere_la_LDFLAGS = -module -avoid-version" > Makefile.am
$ vi hithere.c
#include "../../module.h"

char * get_salutation(void)
{
   return "Hi there";
}
{% endhighlight %}

Okay, look for a moment at this sequence. First, I created a modules directory beneath the existing src directory, and then a hithere directory beneath the new modules directory. The hithere module will provide the salutation, "Hi there" to the caller.

Next, I added a SUBDIRS directive to the top of the src/Makefile.am file, indicating that the new modules directory should be processed by Automake. Then I created a new Makefile.am file in the new hithere directory, containing instructions on how to build the hithere.c source file. Finally, I went ahead and added the hithere.c source file, itself.

The source file includes the private module.h header file using a double quoted relative path. The make VPATH statement will handle any differences between the source and build trees with regard to this relative path. The file then defines the get_salutation function, which is prototyped in the module.h header file. It simply returns a pointer to a static string.

As long as this library is loaded, this string is available to the caller. This is important to know because the caller must know the scope of data references returned by plug-in modules, as such modules could inadvertently be unloaded before the caller is ready to stop using these references.

The last line of the hithere/Makefile.am file requires some explanation. Here, I'm using a -module option on the hithere_la_LDFLAGS variable. This is a Libtool option, that tells Libtool that you really do want to call your library "hithere", and not "libhithere". The GNU Libtool manual makes the statement that modules do not need to be prefixed with "lib". Quite frankly, I'm not sure who came up with this policy, but it seems fairly arbitrary to me. I suppose the reason for this is that since your own code will be loading the module, it should not have to be concerned with using the "lib" prefix. Oh well, there you have it--modules need not be prefixed with "lib".

If you don't care to use module versioning on your dynamically loadable (dlopen) modules, then try using the Libtool -avoid-version option, as I've also done here. This option causes Libtool to generate the shared library as libname.so, rather than libname.so.0.0.0, along with links for libname.so.0 and libname.so pointing to this binary image.

I still need to make one more change to the configure.ac file to get this new module to build. I need to add these two new makefiles to the AC_CONFIG_FILES list.

*configure.ac*

{% highlight shell %}
...
AC_CONFIG_FILES([Makefile
        common/Makefile
        include/Makefile
        libjup/Makefile
        src/Makefile
        src/modules/Makefile
        src/modules/hithere/Makefile])
...
{% endhighlight %}

These changes will allow our module to be built, but I'm still not using it. To use the module, I need to modify the src/main.c file so that it loads the module, imports the symbol, and uses it.

*src/main.c*

{% highlight shell %}
#include <libjupiter.h>
#include "module.h"

#if HAVE_CONFIG_H
# include <config.h>
#endif

#if HAVE_DLFCN_H
# include <dlfcn.h>
#endif

#define DEFAULT_SALUTATION "Hello"

int main(int argc, char * argv[])
{
    char * salutation = DEFAULT_SALUTATION;
#if HAVE_DLFCN_H
    void * module;
    get_salutation_t * get_salutation_fp = 0;
    module = dlopen("./module.so", RTLD_NOW);
    if (module != 0)
    {
        get_salutation_fp = (get_salutation_t *)
        dlsym(module, GET_SALUTATION_SYM);

        if (get_salutation_fp != 0)
            salutation = get_salutation_fp();
    }
#endif

    jupiter_print(salutation, argv[0]);

#if HAVE_DLFCN_H
    if (module != 0)
        dlclose(module);
#endif

    return 0;
}
{% endhighlight %}

In this new version of main.c, I'm including the new private module.h header file. I've also added preprocessor directives to conditionally include config.h, and then dlfcn.h. Finally, I've added two sections of code; one before and one after the original call to jupiter_print. Both are conditionally compiled, based on whether or not I have access to a dynamic loader. This conditional, of course, allows our code to build and run correctly on systems that do not provide run-time dynamic linking via the libdl library.

The general philosophy that I use here when deciding if code should be conditionally compiled is this: if I fail in the configure script because a library or header file is missing, then I don't need to conditionally compile the code that uses the item checked for by configure. If I check for a library or header file in configure, but allow it to continue if it's missing, then I'd better use conditional compilation.

There are just a few more minor points to bring up regarding the use of libdl interface functions. First, dlopen accepts two parameters, a file name or path (absolute or relative), and a flags word, which is the bitwise composite of your choice of several flag values defined in dlfcn.h. If a path is used, then dlopen honors that path verbatim. But if a file name is used, then the library search path is searched for your module. By prefixing the name with ./, I've told dlopen not to search.

But, shouldn't the file name have been "hithere.so"? Well, it's true that I built a module called "hithere.so", but I want to be able to configure which module jupiter uses. So I'm using the generic name, "module.so". In fact, the built module is actually located several directories down in the build tree from the src directory. To test this functionality, I'll need to create a link in the current directory called module.so that points to the module I wish to load.

{% highlight shell %}
$ ./configure && make
...
$ cd src
$ ./jupiter
Hello, from ...jupiter!
$ ln -s modules/hithere/.libs/hithere.so module.so
$ ./jupiter
Hi there, from ...jupiter!
{% endhighlight %}

All of this would normally be done using policy defined in some sort of configuration file in a real application, but none of this is important in this example, so I'm simply ignoring these details to simplify the code.

Check the man page for dlopen to learn more about the flag bits that may be specified. By this point in this chapter, you should have the background required to understand most of the descriptions you'll find there.

**Converting to Libtool's ltdl library**

As I mentioned earlier, Libtool provides a wrapper library called ltdl that abstracts and hides some of the portability issues surrounding the use of shared libraries across many different platforms. Most applications ignore the ltdl library because of the added complexity involved in using it. But there are really only a few issues to deal with. I'll enumerate them here, and then cover them in detail:

- The ltdl functions follow a naming convention based on the dl library. However, the names are different. Generally, the rule of thumb is that dl functions such as dlopen are prefixed in the ltdl library with lt_. Thus, dlopen is named lt_dlopen.
- Unlike the dl library, the ltdl library must be initialized and terminated at appropriate locations in a program.
- To make full use of ltdl functionality--even on platforms that don't provide shared library functionality--you need to build your consuming application (the jupiter program, in this case), using the -dlopen <modulename> option on the linker command line.
- To ensure that modules can be "opened" on non-shared library platforms, or when building static-only configurations, you need to use the LTDL_SET_PRELOADED_SYMBOLS() macro at an appropriate location in your program source code.
- Shared library modules designed to be dlopened using ldtl should use the -module option (and optionally, the -avoid-version option) on the linker command line (specifically, in the *_LDFLAGS variable).
- The ltdl library also provides extensive functionality beyond the dl library, and this can be intimidating, but all of this other functionality is optional.

Take a look specifically at what I need to do to the Jupiter project build system in order to use the ltdl library. First, I need to modify the configure.ac script to look for the ltdl.h header, and search for the lt_dlopen function. This means modifying references to dl.h and the dl library in the AC_CHECK_HEADERS and AC_SEARCH_LIBS macros:

*configure.ac*

{% highlight shell %}
...
# Checks for header files (2).
AC_CHECK_HEADERS([stdlib.h ltdl.h])

# Checks for libraries.
# Checks for typedefs, structures, and compiler...
# Checks for library functions.
AC_SEARCH_LIBS([lt_dlopen], [ltdl])

...
{% endhighlight %}

If I'm using Libtool, then why do I even need to check for ltdl.h and libltdl? Because, these are separate libraries, which must be installed on your end-user's system in order to make them available.

I'd like you to recognize that this is the first time that the Autotools have required an end-user to have an Autotools package installed on his or her machine. This is the very reason is why most people avoid the use of ltdl entirely. The GNU Libtool manual provides a detailed description of how to package the ltdl library with your project, so that it get's built and installed on the end-user's system when your package is built and installed.

In fact, this tutorial (which you'll find in Section 11.6 of that manual) is a great example of adding sub-projects into a project. Interestingly, shipping the source code for the ltdl library with your package is the only way to get your program to statically link with the ltdl library. Linking statically with ltdl has the added (and very ironic) side effect of not requiring the ltdl library to be installed on the end-user's system at all! Since it becomes part of your executable images, you no longer need it to be installed. However, there are caveats to doing this. If you happen to consume a third-party library that does link dynamically to ltdl, then you'll have a symbol conflict between the shared and static versions of the ltdl libraries. Given how little ltdl is currently used, this is an unlikely scenario these days, but all of this could change in the future, if more packages begin to use ltdl, one way or the other.

In any case, by searching for these installed resources on the end-user's system, and by failing configuration if they're not found, or by properly using preprocessor definitions in your source code, you can provide the same sort configuration experience with ltdl that I've talked about throughout this book when using other third-party resources. It's the same, really.

The next major change required is found in the source code--limited, in this case, to src/main.c:

*src/main.c*

{% highlight shell %}
#include <libjupiter.h>
#include "module.h"

#if HAVE_CONFIG_H
# include <config.h>
#endif

#if HAVE_LTDL_H
# include <ltdl.h>
#endif

#define DEFAULT_SALUTATION "Hello"

int main(int argc, char * argv[])
{
    char * salutation = DEFAULT_SALUTATION;

#if HAVE_LTDL_H
    int ltdl;
    lt_dlhandle module;
    get_salutation_t * get_salutation_fp = 0;

    LTDL_SET_PRELOADED_SYMBOLS();

    ltdl = lt_dlinit();
    if (ltdl == 0)
    {
        module = lt_dlopen("modules/.../hithere.la");
        if (module != 0)
        {
            get_salutation_fp = (get_salutation_t *)
            lt_dlsym(module, GET_SALUTATION_SYM);

            if (get_salutation_fp != 0)
                salutation = get_salutation_fp();
        }
    }
#endif

    jupiter_print(salutation, argv[0]);

#if HAVE_LTDL_H
    if (ltdl == 0)
    {
        if (module != 0)
            lt_dlclose(module);
        lt_dlexit();
    }
#endif

    return 0;
}
{% endhighlight %}

The changes here are very symmetrical with respect to the original code. Mostly, items that previously referred to dl now refer to ltdl or lt_dl. For example, #if HAVE_DL_H now becomes #if HAVE_LTDL_H, and so forth.

One important change is the fact that the ltdl library must be initialized with a call to lt_dlinit, whereas the dl library need not be initialized at all. This complicates the code a little--in fact, it may appear to do so much more than it really does, just by virtue of the fact that jupiter is so ridiculously simple. In a larger program, the complexity overhead of calling lt_dlinit and lt_dlexit are amortized over a much larger code base.

Another important detail is the addition of the LTDL_SET_PRELOADED_SYMBOLS macro. This macro is used to configure global variables required by the lt_dlopen and lt_dlsym functions on systems that don't support shared libraries. It's benign on systems where shared libraries are used.

One last detail that I should mention is that the return type of dlopen was void *, or a generic pointer, whereas the return type of lt_dlopen is actually lt_dlhandle. This abstraction is important so that ltdl can be ported to systems that have a return type not compatible with a void pointer.

When a system doesn't support shared libraries, Libtool actually links all modules that might be loaded right into the program. Thus, the jupiter program's linker command line must contain some form of reference to these modules. This is done using the -dlopen <modulename> construct, in this manner:

*src/Makefile.am*

{% highlight shell %}
...
jupiter_LDADD = ../libjup/libjupiter.la -dlopen modules/hithere/hithere.la
...
{% endhighlight %}

Now, this begs the question: What do you do when there is a choice of module to be loaded, as in the case of the jupiter program? If Libtool links them all into a program, and they all provide a get_salutation function, then there will be a conflict of public symbols. Which one will be used? The GNU Libtool manual provides for this condition by defining a convention for symbol naming:

<p style="padding-left:15px">
1. All exported interface symbols should be prefixed with <modulename>_LTX_ (eg., hithere_LTX_get_salutation).
</p>
<p style="padding-left:15px">
2. All remaining non-static symbols should be reasonably unique. The Libtool way is to prefix them with _<modulename>_ (eg., _jupiter_internal_function).
</p>
<p style="padding-left:15px">
3. Modules should, of course, be named differently, even if they're built in different directories.
</p>

Although (unfortunately) it's not explicitly stated in the GNU Libtool manual, the lt_dlsym function first searches for the specified symbol as <modulename>_LTX_<symbolname>, and then, if it can't find a prefixed version of the symbol, for exactly <symbolname>.

You can see that this convention, or something like it, is necessary, but only for cases where Libtool may statically link such loadable modules directly into the application on systems that don't support shared libraries. Libtool's ltdl library makes it possible to have the appearance of shared libraries on platforms that don't support shared libraries, but the price you have to pay for this level of portability is pretty high. This is another reason why people avoid the use of ltdl.

To fix the hithere module's source code so that it's in conformance with this convention, I have to make the following changes:

*src/modules/hithere/hithere.c*

{% highlight shell %}
#define get_salutation hithere_LTX_get_salutation

#include "../../module.h"

char * get_salutation(void)
{
    return "Hi there";
}
{% endhighlight %}

While it is indeed rather odd to have a preprocessor definition above a header file inclusion statement, in this case, it makes sense. By defining the replacement for get_salutation above the inclusion of the module.h header file, I'm also able to change the prototype in the header file so that it matches the modified version of the function name. Because of the way the C preprocessor works, this substitution only affects the function prototype in module.h, not the quoted symbol string, or the type definition.

**Checking it all out**

You can test your program and modules for both static and dynamic shared library systems by using the --disable-shared option on the configure command line:

{% highlight shell %}
$ ./configure --disable-shared && make
...
$ cd src
$ ls -1p modules/hithere/.libs
hithere.a
hithere.la
hithere.lai
$ ./jupiter
Hi there, from ./jupiter!
$
$ cd ..
$ make clean
...
$ ./configure && make
$ cd src
$ ls -1p modules/hithere/.libs
hithere.a
hithere.la
hithere.lai
hithere.o
hithere.so
$ ./jupiter
Hi there, from ...jupiter!
{% endhighlight %}

As you can see, in both configurations, the output contains the hithere salutation, and yet in the --disable-shared version, the shared library doesn't even exist. It seems that ltdl is doing its job.

The Jupiter code base has become rather fragile, because I've ignored the issue of where to find shared libraries at run-time. This problem would ultimately have to be fixed in a real program. But, given that I've finished my task of showing you how to properly use the Libtool ltdl library, and that I've taken the "Hello, world!" concept much farther than anyone has a right to, I think I'll just leave the rest as an exercise.

**Summary**

That was a lot to assimilate. Libtool, as with the other packages in the Autotools tool chain gives you a lot of functionality and flexibility. As you've probably noticed, with this functionality and flexibility comes complexity.

Libtool can make your life easier, or more difficult, depending on how you choose to use the options and flexibility it offers you. But with this background, you can decide the degree to which you will embrace the optional features of Libtool, like the ltdl library, for example. The decision to use shared libraries brings with it a whole truck-load of issues. Each must be dealt with if you're interested in maximum portability. The ltdl library is not a solution to every problem. It solves some problems, but brings others to the surface. Suffice it to say that using ltdl has trade-offs.

Hopefully, by spending a little time going through the exercises in this book, you've been able to "get your head around" the Autotools enough to at least be comfortable with how they work and what they're doing for you. At this point, you should be very comfortable Autotool-izing your own projects--at least at the basic level.

<h4>Chapter 6: FLAIM: an Autotools example</h4>

In this book, I've taken you on a whirlwind tour of the main features of Autoconf, Automake and Libtool. I believe I've explained them in a manner that was not only simple to digest, but also to retain--especially if you had the time and inclination to follow my lead with your own copies of the examples. I've always believed that no form of learning comes anywhere close to the learning that happens while *doing*.

This chapter has [downloads](/assets/download/Autotools-Chapter-6.tgz)!

In this chapter, I'll continue this learning-by-doing pattern by converting an existing open source project to use the GNU Autotools.

The project I've chosen is called FLAIM, which is (what else?) an acronym that stands for FLexible Adaptable Information Management. FLAIM is actually a highly scalable database management system, written entirely in C++, and built on its own thin portability layer called the FLAIM Tool Kit (FTK).

**What's FLAIM?!**

Some of you out there may recognize FLAIM as the database used by both Novell eDirectory and the Novell GroupWise server. Novell eDirectory currently uses this particular version of FLAIM today to manage directory information bases (DIBs) containing over a billion objects. GroupWise actually uses a much earlier spin-off of FLAIM.

Novell made the FLAIM source code available as an open source project licensed under the GNU General Public License (GPL) version 2 in 2006. **The FLAIM project** is hosted by **Novell's forge site**. As a side note, if you're interested in looking at FLAIM yourself, you'll need to set up a Novell account. This is simple to do, and costs nothing. You'll be given the opportunity to create a Novell account the first time you attempt to access the Novell forge site.

**Why FLAIM?**

While FLAIM is not a mainstream OSS project, it has several qualities that make it the perfect choice of project to convert to GNU Autotools in this chapter. For instance, it's currently built using a hand-coded makefile--and a beast of makefile it is, too, containing well over 2000 lines of complex make script. The FLAIM makefile contains a number of GNU-make specific constructs, and thus can only be processed using GNU make. Individual (but nearly identical) makefiles are used to build the flaim, xflaim, and flaimsql database libraries, as well as the FLAIM tool kit (ftk) and several utility and sample programs on GNU/Linux, Unix, Windows and NetWare.

The existing FLAIM build system targets several different flavors of Unix, including AIX, Solaris, and HP/UX, as well as Apple's OS X. It also targets multiple compilers on these systems. These features make FLAIM ideal for my example conversion project, because I can show you how to handle differences in operating systems and tool sets in the new configure.ac files.

The existing build system also contains rules for many of the standard GNU Autotools targets, such as distribution tarballs. In addition, it provides rules for building binary installation packages, as well as RPMs for systems that can build and install RPM packages. Finally, it even provides targets for building doxygen description files, which it then uses to build source documentation. I'll spend a few paragraphs discussing how these types of targets can be added to the infrastructure provided by Automake.

The FLAIM tool kit is a portability library that can be built and consumed in its own right by third-party applications or libraries. This gives me the opportunity to demonstrate Autoconf's ability to manage separate sub-projects as optional sub-directories within a project. That is, if the FLAIM tool kit happens to already be installed on the end-user's build machine, then the installed version may be used, or optionally overridden with the local copy. On the other hand, if the FLAIM tool kit is not installed, then the local, sub-directory based copy will be used by default.

The FLAIM project also provides code to build both Java and CSharp language bindings, which allows me to delve a bit into those esoteric realms. I'll not go into great detail on building either Java or CSharp applications, but I will cover how to write a Makefile.am file that does.

The FLAIM project makes good use of unit tests, which are built as individual programs that run without command line parameters. Thus, I can easily show you how to add unit tests to the new FLAIM build system using Automake's trivial test framework. (Autoconf supplies a more extensive test framework called Autotest, but I'll not discuss Autotest at this time.)

The FLAIM project, and its original build system happen to use a reasonably modular directory layout, making it rather easy to convert to GNU Autotools, which simply run better in projects that follow such good design principles. As one of my goals is ultimately to submit this build system back to the project maintainers, it's nice not to have to rearrange too much of the source code. A simple directory tree diff should suffice.

Finally, I also chose FLAIM because I have some limited experience with it. Although I have been given check-in rights to the project, I'm not really a FLAIM developer, and my experience is pretty much limited to using it for simple database projects occasionally.

**Why hasn't FLAIM already been converted?**

There are several good reasons why FLAIM hasn't already been converted to use the GNU Autotools.

<p style="padding-left:15px">
1. FLAIM is still a fairly new open source project, having only been released a couple of years ago.
</p>
<p style="padding-left:15px">
2. FLAIM's existing build system is well-understood by the developers, and they have limited experience with the GNU Autotools.
</p>
<p style="padding-left:15px">
3. FLAIM's build system targets three different kinds of platform, Windows, Unix and NetWare, using only GNU makefiles. This makes it difficult to give up, because one makefile is used to build FLAIM for all target platforms.
</p>

But FLAIM's build system is not well understood by the open source community. Since FLAIM's release into the "wild", several people have complained about FLAIM's "nasty" makefile on the FLAIM mailing lists. The GNU makefile that FLAIM uses is more or less an unmaintainable monstrosity, from the perspective of new developers. This negative attitude has an almost viral effect on the usefulness of the entire project within the community.

These community critics are accurate in their assessment of FLAIM's build system, with respect to an open source project. The FLAIM team recognizes this and has voiced the desire to establish an Autotools build system, at least for GNU/Linux and Unix platforms. This means that duplicate build systems would have to be created for NetWare and Windows (as per my personal philosophy with respect to using Autotools on non-Unix systems). But, as they say in the shoe business, "The customer is always right!".

**An initial look**

Let me just start by saying that converting FLAIM from GNU makefiles to an Autotools build system is a non-trival project. It took me a couple of weeks. Much of that time was spent determining exactly what to build, and how to do it--in other words, analyzing the existing FLAIM build system. Another significant portion of my time was spent on converting aspects of the FLAIM build system that lay on the outer fringes of Autotools functionality. For example, I spent more time converting build system rules for building CSharp language bindings than I did for building the core C++ FLAIM libraries.

Working on the outer fringes of Autotools capabilities can be a frustrating experience. I'll readily admit that this is where most people get disgusted with the GNU Autotools--especially with Automake. It's my hope that this Chapter will put you ahead of most others in this area. Once you learn a few tricks, working on the outer fringe is pretty simple.

The first step in this conversion project is to analyze the existing directory structure and build system. What components are actually built, and which components depend on others? Can individual components be built, distributed and consumed independently? These types of component-level relationships are important, because they'll often determine how you want to layout your project directory structure.

The FLAIM project is actually several small projects, combined into one large umbrella project within its Subversion repository. There are three separate and distinct database products, flaim, xflaim and flaimsql. The flaim sub-project is the original FLAIM database library used by eDirectory and GroupWise. The xflaim project is a hierarchical XML database, optimized for node-based access. This version was developed for internal projects at Novell. The flaimsql project is FLAIM with integrated SQL semantics exposed through the FLAIM API. This was an experiment, which frankly isn't quite finished.

The point is that all three of these database libraries are separate and unrelated to each other; none of them depend on the others. Since they may easily be used independently of one another, they can actually be shipped as individual distributions. Each could be considered an individual project, in its own right. This, then will become one of my primary goals--to allow the FLAIM project to be easily broken up into smaller projects, which may be managed independently of one another.

The FLAIM tool kit is also an independent project. While it's tailored specifically for the FLAIM database projects, providing just the system service abstractions required for a DBMS, it depends on nothing but itself, and may easily be used as the basis for portability within another project, without dragging any unnecessary database baggage along. As you might guess, its file I/O abstraction is highly optimized.

The existing FLAIM project is laid out in its Subversion repository like this:

{% highlight shell %}
trunk
    flaim
        flaim
            sample
            src
            util
        ftk
            src
            util
        sql
            src
        xflaim
            csharp
            java
            sample
            src
            util
{% endhighlight %}

The complete tree is fairly deep and broad, and there are significant utilities, tests and other such binaries that are built by the existing FLAIM build system. At some point during the downward trek into this hierarchy, I have to simply stop and consider whether it's worth converting that additional utility or layer. If I don't, this chapter will be as long as all the others combined!

To this end, I've decided to convert:

- the libraries themselves
- the unit and library tests
- the utilities and other such high-level programs found in the various util directories
- the Java and CSharp language bindings.

I'll also convert the CSharp unit tests, but I won't go into the Java unit tests because (believe it or not), attempting to work within the Automake-provided Java framework is more painful than just writing the rules yourself. Since Automake provides no help for CSharp, I have to provide everything myself.

**Getting started**

My first true design decision was centered around how to organize this one FLAIM project into sub-projects. As it turns out, the existing layout is perfect for what I've ultimately done. I've created a master configure.ac file in the top-level flaim directory--the one just under trunk. This configure.ac file acts as a sort of Autoconf control file for each of the four lower-level projects, ftk, flaim, flaimsql and xflaim.

I've managed the database library dependencies on the FLAIM tool kit (ftk) by treating it as a pure external dependency, defined by make variables FTKINC and FTKLIB. In this way, I've conditionally defined these variables to point to one of a couple of different sources, including installed libraries, or even user-specified configure options.

**Adding the configure.ac scripts**

The directory structure under the Autotools build system won't change much. In the following directory layout, I've indicated where I've placed individual configure.ac files. You'll recall that each configure.ac file represents a separate and individual project, which may be packaged and distributed independently.

{% highlight shell %}
trunk
    flaim       configure.ac (master)
        flaim     configure.ac (flaim)
            sample
            src
            util
        ftk       configure.ac (ftk)
            src
            util
        sql       configure.ac (flaimsql)
            src
        xflaim    configure.ac (xflaim)
            csharp
            java
            sample
            src
            util
            java
{% endhighlight %}

After these design decisions were made, the next task was to create these configure.ac scripts. The top-level script was trivial, so I created it by hand. The project-specific scripts were more complex, so I allowed the autoscan utility to do the bulk of the work for me. Right now, take a look at that top-level configure.ac script:

{% highlight shell %}
#                 -*- Autoconf -*-
# Process this file with autoconf to produce a c...

AC_PREREQ([2.62])
AC_INIT([flaim-projects], [1.0])
AC_CANONICAL_SYSTEM
AM_INIT_AUTOMAKE([-Wall -Werror foreign])
LT_PREREQ([2.2])
LT_INIT([dlopen])

AC_CONFIG_MACRO_DIR([m4])
AC_CONFIG_SUBDIRS([ftk flaim sql xflaim])
AC_CONFIG_FILES([Makefile])
AC_OUTPUT
{% endhighlight %}

This file is short and simple, because it doesn't do much. Nevertheless, there are some new and important concepts in this file that I'd like to discuss. Since its only job is to configure several lower-level projects, I've taken some shortcuts. The project name and version number, for instance, are really rather unimportant, as this project will probably never be distributed in one large tarball. Regardless, some values had to be used, so I invented the name flaim-projects, and the version number 1.0. These are not likely to change unless really dramatic changes take place in the project directory structure in the future.

The most important aspect of this script is the use of the AC_CONFIG_SUBDIRS macro. This new macro, which I haven't yet covered in this book, lists the sub-projects to be built, along with the current project. This macro is effectively the Autoconf equivalent of the Automake SUBDIRS variable. It allows the maintainer to set up a hierarchy of projects, in much the same way that SUBDIRS configures the directory hierarchy for Automake within a single project.

Because the four sub-projects actually contain all of the functionality, this configure.ac script acts simply as a control file, passing all specified configuration options to each of the sub-projects successively, in the order that they're specified in AC_CONFIG_SUBDIRS. The ordering is important, because the FLAIM tool kit project must be built first, since the other projects depend on it.

Another important new concept in this file is the use of the AC_CANONICAL_SYSTEM macro. This macro causes the environment variables, $host, $build and $target to be defined. These variables contain canonicalized CPU, operating system and manufacturer values for the host, build and target systems. This information can easily be parsed later in the configure.ac file in order to configure system-specific options. I'll dive more deeply into this concept in the project-specific scripts below.

**Automake in the umbrella project**

Automake usually requires the existence of several text files in the top-level project directory. These include the AUTHORS, COPYING, INSTALL, NEWS, README, and ChangeLog files. In the case of this umbrella project, it would be nice not to have to deal with these files, as they are rather redundant here. I could do this by not using Automake at all, but then I'd either have to create my own Makefile.in template for this directory, or use Automake once to generate one for me. I could then check this template into the repository as part of the project, along with the install-sh and missing scripts that are installed by autoreconf -i. Once I have these files in place, I could then remove the AM_INIT_AUTOMAKE macro from the master configure.ac file, and Autoconf will create the final makefile from the preserved template.

Another option would be to keep the AM_INIT_AUTOMAKE macro, and use the foreign option in the macro's optional parameter. The foreign option tells Automake that the project will not follow GNU standards, and thus Automake will not require the usual GNU project text files. This is the path I decided to take, because I might wish to alter the list of subordinate projects at some point in the future, and I don't want to have to hand-tweak the generated Makefile.in template.

The AM_INIT_AUTOMAKE parameter contains a string of white-space separated options that should be assumed by Automake. When Automake parses the configure.ac script, it notes these options, and enables them as if they'd been passed on the command line. I've also passed the -Wall and -Werror options, which indicate that Automake should enable all (Automake) warnings, and report them as errors. Note that these options have nothing to do with the compilation environment--only Automake processing.

**Why add the Libtool macros?**

You may be wondering at this point why I've included those expensive Libtool macros. The reason is more complicated than I wish it were. Even though I don't do anything with Libtool in the umbrella project, the lower level projects expect that a containing project will provide all the necessary scripts, and the LT_INIT macro provides the ltmain.sh script.

If you don't initialize Libtool in the umbrella project, then tools like autoreconf, which actually look in the parent directory to determine if the current project is itself a sub-project, will fail when it can't find scripts that its configure.ac file requires. For instance, within the ftk project's top-level directory, autoreconf expects to find a file called ../ltmain.sh. Note the reference to the parent directory--autoreconf noticed by examining the parent directory that ftk was actually a sub-project of a larger project. Rather than install all of the auxilliary scripts multiple times, it causes sub-projects to look in their parent project's directory for them, so they can be installed once in a multi-project package.

If I don't use LT_INIT in the umbrella project, then I can't successfully run autoreconf in the sub-projects, because the ltmain.sh file will not have been installed in the parent project's top-level directory.

*NOTE: For the rather small disk space savings it provides, I personally don't think it's worth breaking modularity in this manner just to manage this odd child-to-parent relationship.*

**Adding a macro sub-directory**

Another new construct used in the top-level configure.ac script is the AC_CONFIG_MACRO_DIR macro. This macro indicates the name of a sub-directory in which the aclocal utility can find all project-local M4 macro files. These files are ultimately combined into the aclocal.m4 file used by Autoconf. The use of this macro replaces the original single acinclude.m4 file with a directory containing .m4 files.

*NOTE: This entire system of combining (one or more) M4 macro files into a single aclocal.m4 file is a bit of a band-aid over a system that was never originally designed for more than one macro file. In my opinion, it could use a major overhaul, by doing away with aclocal entirely, and just having Autoconf read the macro files in the specified (or defaulted) macro directory, along with other macro files found in system locations.*

I've indicated by the parameter to this macro that all of the local macro files to be added to aclocal.m4 can be found in a sub-directory called m4. As a side benefit, when autoreconf -i is run, and then, when it subsequently executes the required Autotools with their respective "add missing" options, these tools will note the use of AC_CONFIG_MACRO_DIR in configure.ac, and add all missing required system macro files to the m4 directory.

The actual reason for my choosing to do this is that Libtool will not add its additional macro files to the project if you haven't enabled the macro directory option in this manner. Instead, it complains loudly that you should add these files to acinclude.m4 yourself. I found that none of the macros in the Libtool system macro files were required by my project, but that didn't stop it from complaining, and it may not be the case for your projects.

Since I wanted the Autotools to do the job for me, and this is a fairly complex project anyway, I decided to begin using this "macro sub-directory" feature. In point of fact, a future release of Autotools will require this form anyway, as it's considered the more modern way of adding macro files to aclocal.m4, as opposed to using a single user-generated acinclude.m4 file.

**The top-level Makefile.am file**

The only other point to be covered regarding the umbrella project is the top-level Makefile.am file. This file contains the following code:

{% highlight shell %}
ACLOCAL_AMFLAGS = -I m4

EXTRA_DIST = libflaim.changes libxflaim.changes

SUBDIRS = ftk flaim sql xflaim

rpms srcrpm:
    for dir in $(SUBDIRS); do \
        $(MAKE) -C $$dir $@; \
    done

.PHONY: rpms srcrpm
{% endhighlight %}

The ACLOCAL_AMFLAGS variable is a requirement of using a macro sub-directory. According to the Automake documentation, this variable should be defined in the top-level Makefile.am file of any project that uses AC_CONFIG_MACRO_DIR in its configure.ac file. These flags indicate to aclocal where it should look for macro files when it's executed by rules defined in Makefile.am.

I've used the EXTRA_DIST variable here to ensure that additional top-level files get distributed. This isn't critical to the umbrella project, since I don't intend to create distributions at this level, but I like to be complete.

The SUBDIRS variable is a duplicate of the information in the configure.ac file's AC_CONFIG_SUBDIRS macro.

I'll discuss the remaining code later, when I cover adding new make targets to your build system. These particular targets allow the end-user to build RPM packages for rpm-based GNU/Linux systems.

**The sub-projects**

Each of the sub-projects, flaim, ftk, flaimsql and xflaim, are set up just as in the Jupiter project. I'll start with the FLAIM toolkit (ftk) project. Because all of the others are dependent on it, it will have to be built first, anyway.

This configure.ac script was generated for me by autoscan. Autoscan is a bit finicky when it comes to where it will look for information. If your project doesn't contain a makefile file named exactly "Makefile", or if your project already contains an Autoconf Makefile.in template, then autoscan will not add any information about required libraries to the configure.scan output file. It has no other way of determining this information, except by looking into your old build system, and it won't do this unless conditions are just right.

As mentioned earlier, the FLAIM project did contain a rather large makefile, and frankly I was quite impressed with autoscan's ability to parse it for library information, given the complex nature of this multi-platform GNU makefile. Here's a snippet of the ftk project's configure.scan file:

{% highlight shell %}
...
AC_PREREQ(2.62)
AC_INIT(FULL-PACKAGE-NAME, VERSION, BUG-REPORT-ADDRESS)
AC_CONFIG_SRCDIR([util/ftktest.cpp])
AC_CONFIG_HEADERS([config.h])

# Checks for programs.
AC_PROG_CXX
AC_PROG_CC
AC_PROG_INSTALL

# Checks for libraries.
# FIXME: Replace `main' with a function in `-lc':
AC_CHECK_LIB([c], [main])
# FIXME: Replace `main' with a function in...
AC_CHECK_LIB([crypto], [main])
...
AC_CONFIG_FILES([Makefile])
AC_OUTPUT
{% endhighlight %}

I substituted real values for the place-holder values left by autoscan in the AC_INIT macro. I added calls to AM_INIT_AUTOMAKE, LT_PREREQ and LT_INIT. I added a call to AC_CONFIG_MACRO_DIR here, as well. Why not? I'd already done it in the umbrella project above, and this, after all, is the new "UL Approved" method for managing project-local macro files. I then changed the AC_CONFIG_SRCDIR file that autoscan recommended, for one that made more sense to me. And I deleted the use of the AC_PROG_CC macro; this project is written entirely in C++.

Next, I deleted the comments above each of the AC_CHECK_LIB macro calls, and then I started to replace the main place-holders in these macros with actual library function names. I say I started to do that, but I stopped because I wondered if all of those libraries were really necessary. Sometimes I've noticed, where hand-coded build systems are concerned, the author will often cut and paste sets of library names into the makefile until the program builds and runs correctly. (For some reason, this activity is especially prevalent when libraries are being built, although programs are not immune to it.) Also, since autoscan build this list by parsing the original makefile, I figured it probably tried to include everything that it thought might be a library.

Instead of blindly continuing this trend, I chose to simply comment out all of the calls to AC_CHECK_LIB, and see how far I was able to get in the build, and then add them back in one at a time, as required, in order to resolve missing symbols during the build. Unless your project consumes literally hundreds of libraries, this only takes a few extra minutes, but it can save you a lot of time later when builds are speedier than they otherwise might be. And personally, I like to be accurate in my build systems, using only those libraries that really are required. When used religiously, this ideology is also a good form of project-level documentation.

The configure.scan file contained 14 such calls to AC_CHECK_LIB. As it turned out, only three of them were actually required by the FLAIM tool kit on my 64-bit Linux system, pthread, ncurses, and rt. So I deleted the "cruft" and swapped out the place-holder parameters for real functions in the remaining three.

Finally, I added references to src/Makefile and util/Makefile to the AC_CONFIG_FILES macro, and then added the echo statement at the bottom, for some visual verification of my configuration status.

Note that I left all of the header file and library function checks in place, as originally specified by autoscan. I figure that autoscan is probably pretty accurate in noting the use of header files and functions in my source code. Who am I to argue?

Here's the final ftk configure.ac file (*slightly edited, as usual, to satisfy column width requirements*):

{% highlight shell %}
#                 -*- Autoconf -*-
# Process this file with autoconf to produce a c...

AC_PREREQ([2.62])
AC_INIT([FTK], [1.1], [flaim-users@forge.novell.com])
AM_INIT_AUTOMAKE([-Wall -Werror])
LT_PREREQ([2.2])
LT_INIT([dlopen])

AC_LANG(C++)
AC_CONFIG_MACRO_DIR([m4])
AC_CONFIG_SRCDIR([src/flaimtk.h])
AC_CONFIG_HEADERS([config.h])

# Checks for programs.
AC_PROG_CXX
AC_PROG_INSTALL

# Checks for optional programs.
AC_PROG_TRY_DOXYGEN

# Configure options: --enable-debug[=no].
AC_ARG_ENABLE([debug], [AS_HELP_STRING([--enable-debug],
              [enable debug code (default is no)])],
              [debug="$withval"], [debug=no])

# Configure option: --enable-openssl[=no].
AC_ARG_ENABLE([openssl], [AS_HELP_STRING([--enable-openssl], 
              [enable the use of openssl (default is no)])], 
              [openssl="$withval"], [openssl=no])

# Check for doxygen program.
if test -z "$DOXYGEN"; then
    echo "-----------------------------------------"
    echo " No Doxygen program found - continuing"
    echo " without Doxygen documentation support."
    echo "-----------------------------------------"
fi

AM_CONDITIONAL([HAVE_DOXYGEN],[test -n "$DOXYGEN"])

# Checks for libraries.
AC_CHECK_LIB([ncurses], [initscr])
AC_CHECK_LIB([pthread], [pthread_create])
AC_CHECK_LIB([rt], [aio_suspend])
if test "x$openssl" = xyes; then 
    AC_DEFINE([FLM_OPENSSL], [], [Define to use openssl])
    AC_CHECK_LIB([ssl], [SSL_new])
    AC_CHECK_LIB([crypto], [CRYPTO_add])
    AC_CHECK_LIB([dl], [dlopen])
    AC_CHECK_LIB([z], [gzopen])
fi

# Checks for header files.
AC_HEADER_RESOLV
AC_CHECK_HEADERS([arpa/inet.h fcntl.h limits.h \
            malloc.h netdb.h netinet/in.h stddef.h stdlib.h \
            string.h strings.h sys/mount.h sys/param.h \
            sys/socket.h sys/statfs.h sys/statvfs.h \
            sys/time.h sys/vfs.h unistd.h utime.h])

# Checks for typedefs, structures, and compiler ...
AC_HEADER_STDBOOL
AC_C_INLINE
AC_TYPE_INT32_T
AC_TYPE_MODE_T
AC_TYPE_PID_T
AC_TYPE_SIZE_T
AC_CHECK_MEMBERS([struct stat.st_blksize])
AC_TYPE_UINT16_T
AC_TYPE_UINT32_T
AC_TYPE_UINT8_T

# Checks for library functions.
AC_FUNC_LSTAT_FOLLOWS_SLASHED_SYMLINK
AC_FUNC_MALLOC
AC_FUNC_MKTIME
AC_CHECK_FUNCS([atexit fdatasync ftruncate getcwd \
        gethostbyaddr gethostbyname gethostname gethrtime \
        gettimeofday inet_ntoa localtime_r memmove memset \
        mkdir pstat_getdynamic realpath rmdir select \
        socket strchr strrchr strstr])

# Configure DEBUG source code, if requested.
if test "x$debug" = xyes; then
    AC_DEFINE([FLM_DEBUG], [], [Define to enable FLAIM debug features])
fi

# Configure global pre-processor definitions.
AC_DEFINE([_REENTRANT], [], [Define for reentrant code])
AC_DEFINE([_LARGEFILE64_SOURCE], [], [Define for 64-bit data files])
AC_DEFINE([_LARGEFILE_SOURCE], [], [Define for 64-bit data files])

# Configure supported platforms' compiler and li...

case $host in
    sparc-*-solaris*)
        LDFLAGS="$LDFLAGS -R /usr/lib/lwp"
        if "x$CXX" != "xg++"; then
            if "x$debug" = xno; then
                CXXFLAGS="$CXXFLAGS -xO3"
            fi
            SUN_STUDIO=`"$CXX" -V | grep "Sun C++"`
            if "x$SUN_STUDIO" = "xSun C++"; then
                CXXFLAGS="$CXXFLAGS -errwarn=%all -errtags -erroff=hidef,inllargeuse,doubunder"
            fi
        fi
    ;;

    *-apple-darwin*)
        AC_DEFINE([OSX], [], [Define if building on Apple OSX.])
    ;;

    *-*-aix*)
        if "x$CXX" != "xg++"; then
            CXXFLAGS="$CXXFLAGS -qthreaded -qstrict"
        fi
    ;;

    *-*-hpux*)
        if "x$CXX" != "xg++"; then
            # Disable "Placement operator delete
            # invocation is not yet implemented" warning
            CXXFLAGS="$CXXFLAGS +W930"
        fi
    ;;
esac

AC_CONFIG_FILES([Makefile
        docs/Makefile
        docs/doxyfile
        obs/Makefile
        obs/ftk.spec
        src/Makefile
        util/Makefile])

AC_OUTPUT
echo "
    ($PACKAGE_NAME) version $PACKAGE_VERSION
    Prefix.........: $prefix
    Debug Build....: $debug
    Using OpenSSL..: $openssl
    C++ Compiler...: $CXX $CXXFLAGS $CPPFLAGS
    Linker.........: $LD $LDFLAGS $LIBS
    Doxygen........: ${DOXYGEN:-NONE}
"
{% endhighlight %}

Note that I did *not* use the foreign keyword in the AM_INIT_AUTOMAKE macro this time. This is a real project, and I expect it will be packaged as such. Thus, the developers will (should) want these files. I used the touch command to create empty versions of the GNU project text files.

Another new construct near the top of the file is the AC_LANG macro. This macro indicates which language should be assumed when executing compilation tests within the configure script. I've passed "C++" as the parameter, so that Autoconf will generate compilation tests using the C++ compiler via the $CXX variable, rather than the default C code using the $CC macro.

Moving down a few more lines will have you staring at a macro called AC_PROG_TRY_DOXYGEN. Try as you might, you won't find this macro in the Autoconf documentation, because I wrote it myself. Here's the source code, which can be found in ftk/m4/ac_prog_try_doxygen.m4 in the sample code download archive:

{% highlight shell %}
AC_DEFUN([AC_PROG_TRY_DOXYGEN],[
    AC_REQUIRE([AC_EXEEXT])dnl
    test -z "$DOXYGEN" &&\
    AC_CHECK_PROGS([DOXYGEN], [doxygen$EXEEXT])dnl
])
{% endhighlight %}

The macro tests first to see if the end-user has already set the DOXYGEN environment variable. If not, it then uses the standard AC_CHECK_PROG macro to locate it on the host machine, if it's installed. If AC_CHECK_PROG finds it, it sets the DOXYGEN variable to the name of the program, allowing the build system to later locate the actual executable in the system path. If it's not found, it doesn't set the DOXYGEN variable.

There are other more standard macros that check for specific programs. In fact, as simple as this macro is, I could have just used AC_CHECK_PROGS in the configure.ac file, instead of writing my own macro. I wanted to encapsulate the "test and check" construct:

{% highlight shell %}
test -z "$DOXYGEN" && AC_CHECK_PROGS...
{% endhighlight %}

Additionally, I knew I'd need this test in each of the four projects, so it was simpler to create a macro file that could just be copied into the individual projects' m4 directories. Besides, and probably most importantly for this chapter, it's more readable to see AC_PROG_TRY_DOXYGEN, than to see test -z....

Why AC_PROG_TRY_DOXYGEN and not simply AC_PROG_DOXYGEN? Because traditionally, the AC_PROG_* macros fail the configuration process if the associated program is not found. I wanted the DOXYGEN variable to be populated if the doxygen program was found on the system, but be left empty otherwise. That way I could conditionally build the doxygen documentation.

In fact, if you look a bit farther down, you'll see some text that looks like this:

{% highlight shell %}
...
# Check for doxygen program.
if test -z "$DOXYGEN"; then
    echo "-----------------------------------------"
    echo " No Doxygen program found - continuing"
    echo " without Doxygen documentation support."
    echo "-----------------------------------------"
fi
AM_CONDITIONAL([HAVE_DOXYGEN],[test -n "$DOXYGEN"])
...
{% endhighlight %}

This tests whether or not my AC_PROG_TRY_DOXYGEN macro actually found a doxygen program, and acts on the results. If doxygen is not installed on the user's system, then the configure script prints out a large, hard-to-miss message stating that doxygen documentation will not be built. No big deal, really, unless the user was, in fact, counting on it. In that case, she can simply install doxygen and rebuild.

The AM_CONDITIONAL macro defines an automake variable called HAVE_DOXYGEN, which can be used in the project's Makefile.am files to do something conditionally, based on whether or not doxygen can successfully be called (via the $DOXYGEN variable). The first parameter is the Automake conditional variable to be defined, and the second parameter is the test to be run by the configure script in order to determine how the variable should be defined in the makefile. Just one caveat: AM_CONDITIONAL must not be used conditionally (eg., within a shell if statement) in the configure.ac script.

Immediately following the DOXYGEN AM_CONDITIONAL statement, you'll find the library checks. The first three are the ones that autoscan told me about that I found I actually needed after experimenting a bit. The next four are checked within an if statement. Additionally, a preprocessor macro is defined using the AC_DEFINE macro:

{% highlight shell %}
...

if test "x$openssl" = xyes; then 
    AC_DEFINE([FLM_OPENSSL], [], [Define to use openssl])
    AC_CHECK_LIB([ssl], [SSL_new])
    AC_CHECK_LIB([crypto], [CRYPTO_add])
    AC_CHECK_LIB([dl], [dlopen])
    AC_CHECK_LIB([z], [gzopen])
fi
...
{% endhighlight %}

These libraries are included conditionally based on the user's use of the --enable-openssl command-line argument defined in a previous call to the AC_ARG_ENABLE macro. The openssl variable is defined to either yes or no, based on the default value given to AC_ARG_ENABLE, and the user's command-line choices.

The AC_DEFINE macro call ensures that the C++ preprocessor variable, FLM_OPENSSL is defined in the config.h file, and the AC_CHECK_LIB macro calls ensure that -lssl, -lcrypto, -ldl, and -lz strings are added to the $LIBS variable. But only if the openssl macro is defined as yes.

The last item I'll cover here is the conditional use of the AC_DEFINE macro, based on the contents of the debug variable:

{% highlight shell %}
...
# Configure DEBUG source code, if requested.
if test "x$debug" = xyes; then
    AC_DEFINE([FLM_DEBUG], [], [Define to enable FLAIM debug features])
fi
...
{% endhighlight %}

This is another preprocessor definition, conditionally defined, based on the results of a command-line parameter given to configure. The --enable-debug option ultimately enables the definition of FLM_DEBUG within config.h. Both FLM_OPENSSL and FLM_DEBUG are consumed within the FLAIM project source code. Using AC_DEFINE in this manner allows the user to determine what sort of features are compiled into his binaries.

I'll cover the details of the platform-specific checks later in this chapter. This code is identical in all of the projects' configure.ac scripts, as the four original GNU makefiles contained identical such checks.

**The ftk/Makefile.am file**

Discounting the code for doxygen and rpm targets, the ftk/Makefile.am file is fairly trivial:

{% highlight shell %}
ACLOCAL_AMFLAGS = -I m4

EXTRA_DIST = COPYRIGHT GNUMakefile netware

SUBDIRS = src util obs

if HAVE_DOXYGEN
    SUBDIRS += docs
endif

doc_DATA = AUTHORS ChangeLog COPYING COPYRIGHT INSTALL NEWS README

rpms srcrpm: dist
    $(MAKE) -C obs $(AM_MAKEFLAGS) $@
    rpmarch=`rpm --showrc | grep ^build\ arch | sed 's/\(.*: \)\(.*\)/\2/'`; \
    test -z $$rpmarch || ( mv $$rpmarch/* .; rm -rf $$rpmarch )
    -rm -rf $(distdir)



dist-hook:
    -rm -rf `find $(distdir) -name .svn`

.PHONY: srcrpm rpms
{% endhighlight %}

Here, you find the usual ACLOCAL_AMFLAGS, EXTRA_DIST and SUBDIRS variable definitions. But you can also see the use of an Automake conditional. The if statement allows us to append another directory (docs) to the SUBDIRS list, but only if you have access to the doxygen program. If you try to use such a conditional without a corresponding AM_CONDITIONAL in the configure.ac file, then Automake will complain about it.

Another new construct--at least in a top-level Makefile.am file--is the use of the doc_DATA variable. The FLAIM toolkit provides some extra documentation files in its top-level directory that I'd like to have installed. By using the doc prefix on the DATA primary in this manner, I'm telling Automake that I'd like to have these files installed as data files in the $(docdir) directory, which ultimately resolves to the $(prefix)/share/doc directory.

An interesting effect of the use of the DATA primary is that files mentioned in DATA variable are not automatically distributed, so you have to mention them in the EXTRA_DIST variable as well. You'll note that I did not have to mention the standard GNU project text files in EXTRA_DIST. These are always distributed automatically. However, I did have to mention the standard text files in the doc_DATA variable. This is because Automake makes no assumptions about the files that you want installed.

Once again, I'll defer a discussion of the RPM targets until later.

**Automake "-hook" and "-local" rules**

At this point, I'd like to discuss the use of the dist-hook target. Automake recognizes two types of extensions. I call these -local targets and -hook targets. Both of these types of targets represent Automake extension points. Automake recognizes and honors -local extensions for the following standard Automake targets:

- all
- info
- dvi
- ps
- pdf
- html
- check
- install-data
- install-dvi
- install-exec
- install-html
- install-info
- install-pdf
- install-ps
- uninstall
- installdirs
- installcheck
- mostlyclean
- clean
- distclean
- maintainer-clean

Adding a -local version of any of these to your Makefile.am files will cause Automake to ensure that the commands associated with these rules are executed before the associated standard target. Automake does this by generating the rule for the standard target such that the -local version is one of its dependencies (if it exists), thus the -local version is run before the commands for the standard target. Shortly, I'll show an example of this, using a clean-local target.

The -hook targets are a bit different in that they are executed after the corresponding standard target is executed. Automake does this by adding another command to the end of the standard target command list that executes make (via the $(MAKE) variable) on the same Makefile, with the -hook target as the command-line target. Thus, the -hook target is executed at the end of the standard target commands.

The following standard Automake targets support -hook versions:

- install-data
- install-exec
- uninstall
- dist
- distcheck

In this example, I use the dist-hook target to "adjust" the distribution directory before Automake create a tarball from its contents.

{% highlight shell %}
...
dist-hook:
    -rm -rf `find $(distdir) -name .svn`
...
{% endhighlight %}

The `rm` command removes extraneous files and directories that become part of the distribution directory as a result of my adding entire directories to the EXTRA_DIST variable. When you add a directory name to EXTRA_DIST, *everything* in that directory is added to the distribution--even hidden Subversion control files and directories. I certainly don't want this stuff in my tarball, so I use the dist-hook target to add commands that remove these unwanted files after the distribution directory has been created, but before it's "zipped" up into a tarball.

Here's a portion of the generated Makefile, showing how dist-hook is used by Automake:

{% highlight shell %}
...
distdir: $(DISTFILES)
    ... # copy files into distdir
    $(MAKE) $(AM_MAKEFLAGS) \
    top_distdir="$(top_distdir)" \
    distdir="$(distdir)" dist-hook
    ... # change attributes of files in distdir
...

dist dist-all: distdir
    tardir=$(distdir) && $(am__tar) | \
    GZIP=$(GZIP_ENV) gzip -c >$(distdir).tar.gz
    $(am__remove_distdir)
...

.PHONY: ... dist-hook ...
...
dist-hook:
    -rm -rf `find $(distdir) -name .svn`
{% endhighlight %}

Don't be afraid to dig into the generated makefiles to see just exactly what Automake is doing with your code. While there's a fair amount of ugly shell code in there, most of it can be ignored. You're usually more interested in the make rules that Automake is generating, and these are easily separated out. Once you understand the rules, you are well on your way to becoming an Automake expert.

Designing the ftk/src/Makefile.am file

I've left the most difficult task for last. I now need to create appropriate Makefile.am files in the src and utils directories. I want to ensure that all of the original functionality is preserved from the old build system as I'm creating these files. Basically, this includes:

properly building the ftk shared and static libraries;
properly specifying installation locations for all installed files;
setting the ftk library version information correctly;
ensuring that all remaining unused files are distributed;
ensuring that platform-specific compiler options are used.
Besides a few additions to ftk's configure.ac file, the following framework should cover most of the points above, so I'll be using it for all of the FLAIM library projects, with appropriate additions and subtractions, based on the needs of each individual library:

{% highlight shell %}
EXTRA_DIST = ...
lib_LTLIBRARIES = ...
include_HEADERS = ... 
xxxxx_la_SOURCES = ... 
xxxxx_la_LDFLAGS = -version-info x:y:z
{% endhighlight %}

The original GNU makefile told me that the library was named libftk.so. This is a bad name for a library on Linux, as most of the three-letter acronyms are already taken for other purposes within the file system, so I've made an executive decision here and renamed the ftk library to flaimtk. I added the libtool library name, libflaimtk.la to the lib_LTLIBRARIES list, and then changed the xxxxx portions of the remaining macros to libflaimtk.

To get the source files, I could have entered them all by hand, but I noticed while reading the original makefile that it used the GNU make function macro, $(wildcard src/*.cpp) in order to build the file list for the library from the contents of the src directory. This tells me that all of the .cpp files within the source directory are required by the library. To get the file list into Makefile.am, I used a simple shell command to concatenate the file list to the end of the Makefile.am file (assuming I'm in the ftk/src directory):

{% highlight shell %}
$ ls >> Makefile.am
{% endhighlight %}

This leaves me with a single column list of all of the files in the ftk/src directory appended to the bottom of the ftk/src/Makefile.am file. I deleted the Makefile.am file from this list, and then moved the list to just below the libflaimtk_la_SOURCES = entry. I added a BACKSLASH character after the EQUAL sign, and at the end of each of the files except the last one. This gives me a clean file list. Another formatting technique is to simply wrap the line every 70 characters or so with a BACKSLASH and a CARRIAGE RETURN. I prefer to put each file on a separate line--especially early on in the conversion process, so that I can easily extract or add files to the lists.

For the header files, I had to manually examine each one to determine its use in the project. There are only four header files in the src directory, and as it turns out, the only one not used by ftk on Unix and GNU/Linux platforms is ftknlm.h. This file is specific to the NetWare build. I added this file to the EXTRA_DIST list.

The ftk.h file (now renamed to flaimtk.h) is the only public header file, so I moved that one into the include_HEADERS list. The other two are used internally in the library build, so I left them in the libflaimtk_la_SOURCES list.

Finally, I noted in the original makefile, that the last ftk library that was released to the public in a distribution sported an interface version of 4.0.0. However, since I change the name of the library from libftk to libflaimtk, I reset this value to 0.0.0 because it's a different library now, so I replaced x:y:z with 0:0:0 in the -version-info option within the libflaimtk_la_LDFLAGS variable. (_NOTE: Version 0.0.0 is the default, so I could have simply removed the -version-info argument entirely for the same effect.) Here's (most of) the final ftk/src/Makefile.am file:

{% highlight shell %}
EXTRA_DIST = ftknlm.h

lib_LTLIBRARIES = libflaimtk.la
include_HEADERS = flaimtk.h

libflaimtk_la_SOURCES = \
                    ftkarg.cpp \
                    ftkbtree.cpp \
                    ftkcmem.cpp \
                    ftkcoll.cpp \
                    ...
                    ftksys.h \
                    ftkunix.cpp \
                    ftkwin.cpp \
                    ftkxml.cpp

libflaimtk_la_LDFLAGS = -version-info 0:0:0
{% endhighlight %}

That's it! You know--I don't know about you, but I'd much rather maintain this file, than a 2200 line GNU makefile! Granted, I'm not really done yet, but (trust me) it won't get much worse than this.

**Moving on to the ftk/util directory**

Properly designing a Makefile.am file for the util directory requires examining the original makefile again for more products--those built from files in the ftk/util directory. A quick glance at the ftk/util directory showed that there was only one source file, ftktest.cpp. This appeared to be some sort of testing program for the ftk library, so I had a design decision to make here: should I build this as a normal program, or as a "check" program.

The difference, of course, is that normal programs are always built, but "check" programs are only built when make check is executed. Remember also that check programs are never installed. Thus, if I chose to always build ftktest, I'd then have to decided whether or not I want it to be installed. If I want it built all the time, but not installed, I'd have to specify the program using the noinst prefix, rather than the usual bin prefix.

In either case, I probably want to add the ftktest binary to the list of tests run during make check, so the two questions here are (1) whether or not I might wish to run ftktest manually at times after a build, and (2) do I want to install the ftktest program? Given that ftk is rather mature at this point, I opted to not install ftktest and only build it during make check. Here's my final ftk/util/Makefile.am file:

{% highlight shell %}
FTKINC=-I$(top_srcdir)/src
FTKLIB=../src/libflaimtk.la

check_PROGRAMS = ftktest

ftktest_SOURCES = ftktest.cpp
ftktest_CPPFLAGS = $(FTKINC)
ftktest_LDADD = $(FTKLIB)

TESTS = ftktest
{% endhighlight %}

Note that I could easily have left out the FTKINC and FTKLIB variables, replacing their references with the appropriate text in the CPPFLAGS and LDADD variables, but since this will be a pattern used quite often in the new FLAIM build system, because of the external dependency between the database projects and the tool kit, I've decided to start the habit right here and now.

I hope by now that you can see the relationship between TEST and check_PROGRAMS. To be blunt, there really is no relationship between the files listed in check_PROGRAMS and those listed in TEST. TEST can refer to anything that can be executed without command line parameters, and these programs or scripts are executed during make check after all of the check_PROGRAMS are built (if any). This separation of duties makes for a very clean and flexible system.

**Designing the xflaim build system**
Now that I've finished with the tool kit, I'll move on to the xflaim project. I'm choosing xflaim, rather than flaim, because it supplies the most build features that can be converted to GNU Autotools, including the Java and CSharp language bindings. After xflaim, covering the remaining database projects would be redundant, as the processes are identical (but simpler). However, you can find the other build system files in the attached source archive, as usual.

I generated the configure.ac script using autoscan again. It's important to use autoscan in each of the individual projects, because the source code of each project is different, and will thus cause different macros to be written into each configure.scan file. I then used the same techniques to create xflaim's configure.ac script as I used with the tool kit.

**The xflaim configure.ac script**

After hand-modifying the generated configure.scan file and renaming it to configure.ac, I found this configure.ac script to be similar in many ways to ftk's configure.ac script. As it's fairly long, I'll only show you the most significant differences here:

{% highlight shell %}
...
# Checks for optional programs.
AC_PROG_TRY_CSC
AC_PROG_TRY_CSVM
AC_PROG_TRY_JAVAC
AC_PROG_TRY_JAVAH
AC_PROG_TRY_JAVADOC
AC_PROG_TRY_JAR
AC_PROG_TRY_DOXYGEN

# Configure variables: FTKLIB and FTKINC.
AC_ARG_VAR([FTKLIB], [where libflaimtk.la is at])
AC_ARG_VAR([FTKINC], [where flaimtk.h is at])

# Ensure that both or neither are specified.
if (test -n "$FTKLIB" && test -z "$FTKINC") || \
   (test -n "$FTKINC" && test -z "$FTKLIB"); then
    AC_MSG_ERROR([both or neither FTK variables])
fi 

# Not specified? Check for FTK in standard places.
if test -z "$FTKLIB"; then
    # Check for flaim tool kit as a sub-project.
    if test -d "$srcdir/ftk"; then
        AC_CONFIG_SUBDIRS([ftk])
        FTKINC='$(top_srcdir)/ftk/src'
        FTKLIB='$(top_builddir)/ftk/src'
    else
        # Check for flaim tool kit as a super-project.
        if test -d "$srcdir/../ftk"; then
            FTKINC='$(top_srcdir)/../ftk/src'
            FTKLIB='$(top_builddir)/../ftk/src'
        fi
    fi
fi

# Still empty? Check for installed flaim tool kit.
if test -z "$FTKLIB"; then
    AC_CHECK_LIB([flaimtk], [ftkFastChecksum], 
            [AC_CHECK_HEADERS([flaimtk.h])
            LIBS="-lflaimtk $LIBS"],
            [AC_MSG_ERROR([No FLAIM Took Kit found.])])
fi

# AC_SUBST command line variables.
if test -n "$FTKLIB"; then
    AC_SUBST([FTK_LTLIB], ["$FTKLIB/libflaimtk.la"])
    AC_SUBST([FTK_INCLUDE], ["-I$FTKINC"])
fi

# Check for Java compiler.
have_java=yes 
if test -z "$JAVAC"; then have_java=no; fi
if test -z "$JAVAH"; then have_java=no; fi
if test -z "$JAR"; then have_java=no; fi
if test "x$have_java" = xno; then
    echo "-----------------------------------------"
    echo " Some Java tools not found - continuing"
    echo " without XFLAIM JNI support."
    echo "-----------------------------------------"
fi
AM_CONDITIONAL([HAVE_JAVA], [test "x$have_java" = xyes])

# Check for CSharp compiler.
if test -z "$CSC"; then
    echo "-----------------------------------------"
    echo " No CSharp compiler found - continuing"
    echo " without XFLAIM CSHARP support."
    echo "-----------------------------------------"
fi
AM_CONDITIONAL([HAVE_CSHARP], [test -n "$CSC"])

...

echo "
    ($PACKAGE_NAME) version $PACKAGE_VERSION
    Prefix.........: $prefix
    Debug Build....: $debug
    C++ Compiler...: $CXX $CXXFLAGS $CPPFLAGS
    Linker.........: $LD $LDFLAGS $LIBS
    FTK Library....: ${FTKLIB:-INSTALLED}
    FTK Include....: ${FTKINC:-INSTALLED}
    CSharp Compiler: ${CSC:-NONE} $CSCFLAGS
    CSharp VM......: ${CSVM:-NONE}
    Java Compiler..: ${JAVAC:-NONE} $JAVACFLAGS
    JavaH Utility..: ${JAVAH:-NONE} $JAVAHFLAGS
    Jar Utility....: ${JAR:-NONE} $JARFLAGS 
    Javadoc Utility: ${JAVADOC:-NONE}
    Doxygen........: ${DOXYGEN:-NONE}
"
{% endhighlight %}

First, you'll notice that I've invented a few more of my AC_PROG_TRY macros. In the first portion, I'm checking for the optional existence of the following programs: a CSharp compiler, a CSharp virtual machine, a Java compiler, a JNI header and stub generator, a javadoc generation tool, a Java archive tool, and of course, doxygen. As before, I've written separate macro files for each of these checks, and added them to my xflaim/m4 directory.

As with the AC_PROG_TRY_DOXYGEN macro, each of these macros attempts to locate the associated program, but doesn't go apoplectic if it's not found, because I want to be able to use the program if it's there, but not require my users to have them in order to build some of the most useful functionality of the FLAIM projects.

Next, you'll find a new macro, AC_ARG_VAR. Like the AC_ARG_ENABLE and AC_ARG_WITH macros, AC_ARG_VAR allows the project maintainer to extend the interface to the configure script. This variable is different, however, in that it adds a public variable to the list of variables that the configure script cares about. In this case, I'm adding two public variables, FTKINC and FTKLIB. These variables will show up in the configure script's help text under the section entitled "Some influential environment variables:".

These variables are also automatically substituted into the Makefile.in templates generated by Automake. However, I don't really need this substitution functionality, as I'm going to build other variables out of these variables, and I'll want these derived variables to be substituted, as you'll soon see.

These variables may be set by the user in the environment, or specified on the configure script's command line in this manner:

{% highlight shell %}
$ ./configure FTKINC='$HOME/dev/ftk/include' ...
{% endhighlight %}

The large chunk of code that follows the AC_ARG_VAR macros actually uses these variables to set other variables used in the build system:

{% highlight shell %}
...

# Ensure that both or neither are specified.
if (test -n "$FTKLIB" && test -z "$FTKINC") || \
   (test -n "$FTKINC" && test -z "$FTKLIB"); then
    AC_MSG_ERROR([both or neither FTK variables])
fi 

# Not specified? Check for FTK in standard places.
if test -z "$FTKLIB"; then
    # Check for flaim tool kit as a sub-project.
    if test -d "$srcdir/ftk"; then
        AC_CONFIG_SUBDIRS([ftk])
        FTKINC='$(top_srcdir)/ftk/src'
        FTKLIB='$(top_builddir)/ftk/src'
    else
        # Check for flaim tool kit as a super-project.
        if test -d "$srcdir/../ftk"; then
            FTKINC='$(top_srcdir)/../ftk/src'
            FTKLIB='$(top_builddir)/../ftk/src'
        fi
    fi
fi

# Still empty? Check for installed flaim tool kit.
if test -z "$FTKLIB"; then
    AC_CHECK_LIB([flaimtk], [ftkFastChecksum], 
                 [AC_CHECK_HEADERS([flaimtk.h])
                  LIBS="-lflaimtk $LIBS"],
                 [AC_MSG_ERROR([No FLAIM Took Kit found.])])
fi

# AC_SUBST command line variables.
if test -n "$FTKLIB"; then
    AC_SUBST([FTK_LTLIB], ["$FTKLIB/libflaimtk.la"])
    AC_SUBST([FTK_INCLUDE], ["-I$FTKINC"])
fi
...
{% endhighlight %}

First, I check to see that either both variables are specified, or neither. If only one of them is given, then I have to fail with an error. The user isn't allowed to tell me where to find half the tool kit. I need both the include file and the library.

If neither is specified, then I go searching for them. First I look for a sub-directory called ftk. If I find one, then I configure that directory as a sub-project to be processed by Autoconf, by using the AC_CONFIG_SUBDIRS macro. Note that you can use this macro conditionally, and multiple times within the same configure.ac file. I also set the variables to point to the appropriate relative locations within the ftk project.

If I don't find it as a sub-directory, then I look for it in the parent directory. If I find it there, I set the FTK variables appropriately. This time I don't need to configure the located ftk directory as a sub-project, because I'm assuming that the current project (xflaim) is already a sub-project of the umbrella project.

If I don't find it in either place, I use the standard AC_CHECK_LIB and AC_CHECK_HEADERS macros to see if it's installed on the user's host machine. If so, I need only add -lflaimtk to the $LIBS variable. The header file will be found in the standard location--usually /usr(/local)/include. Note that normally, AC_CHECK_LIB would automatically add the library reference to the $LIBS variable, but since I've overridden the default functionality in the third parameter, I have to add it myself.

If I don't find it installed, then I give up with an error message, indicating that xflaim can't be built without the FLAIM tool kit.

However, after making it through the checks, if the FTKLIB variable is no longer empty, then I use AC_SUBST to "publish" FTK_INCLUDE and FTK_LTLIB variables, containing derivations of the FTK variables appropriate for the C++ preprocessor and the linker.

The remaining code (excluding the trailing echo statement) calls AM_CONDITIONAL for Java and CSharp tools in a manner similar to the way I handled doxygen. Again, I generate bold messages to the user that the Java or CSharp portions of the xflaim project will not be built if those tools can't be found, but I allow the build to continue.

**Creating the xflaim/src/Makefile.am file**

I wrote the xflaim/src/Makefile.am file by following the same design principles used in the ftk/src version of that file. It looks very similar to its ftk counterpart, with one exception: According to the original build system makefile, the Java native interface (JNI) and CSharp native language binding sources are compiled and linked right into the xflaim shared library.

This is not an uncommon practice, because it alleviates the need for extra library objects specifically for these languages. Essentially, the xflaim shared library exports native interfaces for these languages, that are then consumed by their corresponding language binding wrappers.

I'm going to ignore these language binding interfaces for now. However, keep them in the back of your mind, because later when I've finished with the entire xflaim project, I'll turn my attention back to properly hooking these bindings into the library. Except for the language bindings then, the Makefile.am file looks almost identical to its ftk counterpart:

{% highlight shell %}
SUBDIRS = 

if HAVE_JAVA
    SUBDIRS += java
    JNI_LIBADD=java/libxfjni.la
endif

if HAVE_CSHARP
    SUBDIRS += cs
    CSI_LIBADD=cs/libxfcsi.la
endif

SUBDIRS += .

lib_LTLIBRARIES = libxflaim.la
include_HEADERS = xflaim.h

libxflaim_la_SOURCES = \
                     btreeinfo.cpp \
                     f_btpool.cpp \
                     f_btpool.h \
                     ...
                     rfl.h \
                     scache.cpp \
                     translog.cpp

libxflaim_la_CPPFLAGS = $(FTK_INCLUDE)
libxflaim_la_LIBADD = $(JNI_LIBADD) $(CSI_LIBADD) $(FTK_LTLIB)
libxflaim_la_LDFLAGS = -version-info 3:2:0
{% endhighlight %}

As I did with the docs directory in the top-level Makefile.am file, I've conditionally defined the SUBDIRS variable here, based on the Automake conditional specified in configure.ac. What's different here is that I've pre-defined SUBDIRS to be empty before checking the condition, and then added the current directory (.) at the end.

These directories must be processed (if they can be) before the current directory, as they generate libraries that must be linked into the library built by this makefile. I had to initialize SUBDIRS to empty because the PLUS-EQUAL (+=) Automake extension operator will only work properly if the variable is already defined--even if it must be defined as empty.

Since I initialized it to empty, I removed the implicit current directory, so I added it back in after the conditional checks. It's a bit clumsy, I know, but it works.

The library interface version information was once again extracted from the original xflaim project makefile.

**Turning to the xflaim/util directory**

The util directory for xflaim is a bit more complex. According to the original makefile, it generates several utility programs, as well as a convenience library that is consumed by each of these utilities.

In addition, the task of finding out which source files belong to which utilities, and which were not used at all was more difficult. It turns out that there are several files in the xflaim/util directory that are not used by any of the utilities. I suppose the project developers thought there might be some future value in these source files, so they kept them around. Well, that leaves us with another decision: Do we distribute these "extra" source files? I chose to do so, as they were already being distributed by the original build system, and adding them to the EXTRA_DIST list makes it obvious to later observers that they aren't used in the build.

Here's my version of the xflaim/util/Makefile.am file:

{% highlight shell %}
EXTRA_DIST = dbdiff.cpp dbdiff.h domedit.cpp diffbackups.cpp xmlfiles

XFLAIM_INCLUDE=-I$(top_srcdir)/src
XFLAIM_LDADD=../src/libxflaim.la

## Utility Programs
bin_PROGRAMS = xflmcheckdb xflmrebuild xflmview xflmdbshell

xflmcheckdb_SOURCES = checkdb.cpp
xflmcheckdb_CPPFLAGS = $(XFLAIM_INCLUDE) $(FTK_INCLUDE)
xflmcheckdb_LDADD = libutil.la $(XFLAIM_LDADD)
xflmrebuild_SOURCES = rebuild.cpp
xflmrebuild_CPPFLAGS = $(XFLAIM_INCLUDE) $(FTK_INCLUDE)
xflmrebuild_LDADD = libutil.la $(XFLAIM_LDADD)
xflmview_SOURCES = viewblk.cpp view.cpp \
                 ... viewmenu.cpp viewsrch.cpp
xflmview_CPPFLAGS = $(XFLAIM_INCLUDE) $(FTK_INCLUDE)
xflmview_LDADD = libutil.la $(XFLAIM_LDADD)
xflmdbshell_SOURCES = domedit.h fdomedt.cpp fshell.cpp fshell.h xshell.cpp
xflmdbshell_CPPFLAGS = $(XFLAIM_INCLUDE) $(FTK_INCLUDE)
xflmdbshell_LDADD = libutil.la $(XFLAIM_LDADD)

## Utility Convenience Library 
noinst_LTLIBRARIES = libutil.la
libutil_la_SOURCES = flm_dlst.cpp flm_dlst.h flm_lutl.cpp flm_lutl.h \
                     sharutil.cpp sharutil.h

libutil_la_CPPFLAGS = $(XFLAIM_INCLUDE) $(FTK_INCLUDE)

## Check Programs
check_PROGRAMS = ut_basictest ut_binarytest \
                 ...
                 ut_xpathtest ut_xpathtest2

check_DATA = copy-xml-files.stamp
check_HEADERS = flmunittest.h

ut_basictest_SOURCES = flmunittest.cpp basictestsrv.cpp
ut_basictest_CPPFLAGS = $(XFLAIM_INCLUDE) $(FTK_INCLUDE)
ut_basictest_LDADD = libutil.la $(XFLAIM_LDADD)
...

ut_xpathtest2_SOURCES = flmunittest.cpp xpathtest2srv.cpp
ut_xpathtest2_CPPFLAGS = $(XFLAIM_INCLUDE) $(FTK_INCLUDE)
ut_xpathtest2_LDADD = libutil.la $(XFLAIM_LDADD)

## Unit Tests
TESTS = ut_basictest ... ut_xpathtest2

## Miscellaneous rules required by Check Programs
copy-xml-files.stamp:
    cp $(srcdir)/xmlfiles/*.xml .
    echo Timestamp > $@

clean-local: 
    -rm -rf ix2.*
    -rm -rf bld.*
    -rm -rf tst.bak
    -rm -f *.xml
    -rm -f copy-xml-files.stamp
{% endhighlight %}

In this example, you can see by the ellipses that I left out several long lists of files and products. There are, for instance, 22 unit tests built by this makefile. I only left the descriptions for two of them, because they're all identical, except for naming differences and the source files from which they're built.

But here's something curious. Take a look at the definition for the xflmcheckdb program:

{% highlight shell %}
...
xflmcheckdb_SOURCES = checkdb.cpp
xflmcheckdb_CPPFLAGS = $(XFLAIM_INCLUDE) $(FTK_INCLUDE)
xflmcheckdb_LDADD = libutil.la $(XFLAIM_LDADD)
...
{% endhighlight %}

Notice that the xflmcheckdb_CPPFLAGS variable uses both the XFLAIM_INCLUDE and FTK_INCLUDE variables. The utility clearly requires information from both sets of header files. But the xflmcheckdb_LDADD variable only uses the XFLAIM_LDADD variable. Why? Because Libtool manages inter-library dependencies for you. Since I reference libxflaim.la (through XFLAIM_LDADD) when building the utilities and unit tests, and since libxflaim.la lists libflaimtk.la as a dependency, I don't need to explicitly reference that library here.

You can get a clearer picture of this if you take a look at the contents of libxflaim.la (in your build directory under xflaim/src). You'll find a few lines like this somewhere in the middle of the file:

{% highlight shell %}
...
# Libraries that this one depends upon.
dependency_libs=
 ' .../flaim/build/ftk/src/libflaimtk.la -lrt -lpthread -lncurses'
...
{% endhighlight %}

Notice that the path information for libflaimtk.la is listed here, thus we don't have to specify it in the LDADD variables for the xflaim utilities. The linker still requires this information, but the libtool script effectively hides this requirement by extracting the information from the .la file and appending it to the linker command line when building the utility files.

As an aside, when libxflaim.la is installed, Libtool modifies the installed version of this file such that it references the installed versions of the libraries, rather than those in the build directory structure.

**Stamp targets**

In creating this makefile, I ran across another minor problem that I hadn't anticipated. At least one of the unit tests (probably several) seemed to require that some XML data files be present in the directory from which the test is run. What brought this to my attention was the fact that that particular unit test failed. When I dug into it, I noticed that it was trying to open some specifically named XML data files. Searching around a bit lead me to the xmldata directory, beneath the xflaim/util directory. This directory contained several dozen XML data files.

Somehow I needed to copy those files into the build hierarchy's xflaim/util directory before I could run the unit tests. Well, I know that check programs are built before TESTS are executed. As it turns out other primaries prefixed with check are also processed before TESTS are executed. Notice the check_DATA variable:

{% highlight shell %}
...
check_DATA = copy-xml-files.stamp
...
copy-xml-files.stamp:
    cp $(srcdir)/xmlfiles/*.xml .
    echo Timestamp > $@
...
{% endhighlight %}

It refers to a file called copy-xml-files.stamp. This is a special type of file target called a "stamp" target. It's purpose is to replace a bunch of unspecified files, or a non-file-based operation, with one single representative file. This stamp file is used to indicate to the make system that the operation of copying all of the XML data files into the test directory has been done. Automake uses stamp files quite often in its own generated rules.

The rule for generating the stamp file (near the bottom of the example above), also copies the XML data files into the test execution directory. The echo statement simply creates a file named copy-xml-files.stamp, and containing the single word, "Timestamp". The file may contain anything, really. The important point here is that the file exists, and has a time and date associated with it. The make utility uses this information to determine whether or not the copy operation needs to be executed. In this case, since copy-xml-files.stamp has no dependencies, its mere existence indicates to make that the operation has already been done, and need not be done again.

To get make to perform the copy operation on the next build, simply delete the stamp file. This is a sort of hybrid between a true file-based rule, and a phony target. Phony targets are always executed, because they aren't real files, so make has no way of knowing whether or not the associated operation should be performed. The time stamps of file-based rules can be checked against their dependency lists to determine if they should be re-executed, or not. Stamp rules like this one are executed only if the stamp file is missing.

All files placed in the build directory should be cleaned up when the user enters make clean at the command prompt. Since I placed these XML data files into this directory, I need to clean them up also. Files listed in DATA variables are not cleaned up automatically, because DATA files are usually not generated files. Most often, the DATA primary is used to list existing project files that need to be installed. In this case, I actually created a bunch of XML files and a stamp file, so I need to clean these up when make clean is executed.

NOTE: Be careful when using this technique on files that need to be copied from the source directory into the same corresponding location in the build directory. Special care needs to be taken to ensure you don't inadvertently delete source files from the source tree when building in the source tree.

**Cleaning your room**

There is another way to ensure that files created using your own make rules get cleaned up during execution of the clean target. You may also define the CLEANFILES variable to contain a white space separated list of files or wild card specifications to be removed. The CLEANFILES variable is the more "approved" method of removing extra files during make clean.

If that's so, then why did I use clean-local in this case? Because the CLEANFILES variable has one caveat: it won't remove directories, only files. Each of the rm commands above removes a wild card file specification that contains at least one directory, so I had to use clean-local in this case. I'll show you a proper use of CLEANFILES shortly.

{% highlight shell %}
...
clean-local: 
    -rm -rf ix2.*
    -rm -rf bld.*
    -rm -rf tst.bak
    -rm -f *.xml
    -rm -f copy-xml-files.stamp
...
{% endhighlight %}

Here, I needed to remove all files ending in .xml, plus the stamp file. In addition, the unit tests themselves are not well written, in that they leave "droppings" behind. Let this be a lesson: when you write unit tests that generate files and directories, remove all such droppings before terminating your test. That way, you won't have to write such clean rules in your makefiles.

Another way of managing this is would be to write a script that calls the tests, and then cleans up left-over files and directories. This script then becomes the entry in the TESTS variable.

I use the Automake supported clean-local target here as a way to extend the clean target. The clean-local target is executed as a dependency of (and thus before) the clean target, if it exists. Here's the corresponding code from the Automake-generated Makefile:

{% highlight shell %}
...
clean: clean-am

clean-am: clean-binPROGRAMS clean-checkPROGRAMS clean-generic clean-libtool clean-local clean-noinstLTLIBRARIES mostlyclean-am
...
.PHONY: ... clean-local ...
...
clean-local: 
    -rm -rf ix2.*
    -rm -rf bld.*
    -rm -rf tst.bak
    -rm -f *.xml
    -rm -f copy-xml-files.stamp
...
{% endhighlight %}

Automake noted that I had a target named clean-local in my Makefile.am file, so it added clean-local to the dependency list for clean-am, and then added it to the .PHONY list. Had I not written a clean-local target, these references would have been missing from the generated Makefile.

When cleaning up files in a build directory using wild cards in this manner, you need to remember that the user may be building in the source directory. Try to make your wild cards as specific as possible so you don't inadvertently remove source files.

**Building Java sources using Autotools**

The most significant barrier to building Java sources using the GNU Autotools is the (apparently nearly intentional) misdirection in the existing documentation. Now, I know better than to think it was done on purpose, but time and time again, what you find in internet searches, or in the GNU Automake documentation is just enough information, presented in just such a way as to allow you to really hang yourself well when you try to use it. There's nothing quite as frustrating as finding dozens of implications that something can be done, but finding no information telling you exactly how to do it.

There are two sections in the GNU Automake manual that refer to building Java sources using the GNU Autotools. The first is section 8.15, entitled, "Java Support". The second is section 10.4, entitled simply, "Java". (The major section 10 is entitled, "Other GNU Tools".)

In the first place, the contents of these two sections should probably be swapped. Section 8.15 actually discusses using the GCJ front end to the GNU compiler suite to compile and link Java source code into native executables. This is nothing that the average Java purist would understand without a little hand-holding, because Sun Java doesn't do anything of the sort. The information in this section would be better placed under a section entitled, "Other GNU Tools" (like section 10, for instance).

On the other hand, section 10.4 talks about building Java sources using whatever javac compiler happens to be found in the system path. This is much more likely to be something a Java developer might actually wish to do in a Makefile.am file, so I'm going to ignore section 8.15 (native compilation, using GCJ), and talk strictly about section 10.4.

**Autotools Java support**

Autoconf has no built-in support for java. For example, it provides no macros that locate Java tools in the end user's environment. Automake's support for building Java classes is minimal, but getting it to work is not that difficult if you know what you're doing.

Automake provides a built-in primary (JAVA) for building Java sources. Automake does not provide any preconfigured installation location prefixes for installing Java classes. However, the usual place to install Java classes and .jar files is in the $(datadir)/java directory. So, creating a proper prefix is as simple as using the Automake prefix extension mechanism of defining a variable suffixed with dir:

{% highlight shell %}
...
javadir = $(datadir)/java
java_JAVA = file_a.java file_b.java ...
...
{% endhighlight %}

Note that you don't often want to install Java sources, which is what you will accomplish when you define your JAVA primary with this sort of prefix. Rather, you want the class files to be installed, or more likely a .jar file containing all of your .class files. So I find it more useful to define the JAVA primary with the noinst prefix. Additionally, files in the JAVA primary list are not distributed by default, so you may even want to use the dist super-prefix, in this manner:

{% highlight shell %}
...
dist_noinst_JAVA = file_a.java file_b.java ...
...
{% endhighlight %}

When you define a list of Java source files in a variable containing the JAVA primary, Automake generates a make rule that builds that list of files all in one command, using the following command line syntax:

{% highlight shell %}
...
JAVAROOT = $(top_builddir)
JAVAC = javac
CLASSPATH_ENV = CLASSPATH=$(JAVAROOT):$(srcdir)/$(JAVAROOT):$$CLASSPATH
...
classdist_noinst.stamp: $(dist_noinst_JAVA)
    @list1='$?'; list2=; \
    if test -n "$$list1"; then \
        for p in $$list1; do \
            if test -f $$p;
                then d=; \
            else d="$(srcdir)/"; \
            fi; \
            list2="$$list2 $$d$$p"; \
        done; \
        echo '$(CLASSPATH_ENV) $(JAVAC) -d $(JAVAROOT) $(AM_JAVACFLAGS) \
            $(JAVACFLAGS) '"$$list2"; $(CLASSPATH_ENV) $(JAVAC) \
            -d $(JAVAROOT) $(AM_JAVACFLAGS) $(JAVACFLAGS) $$list2; \
    else :;
    fi
    echo timestamp > classdist_noinst.stamp
...
{% endhighlight %}

Most of the "stuff" you see in the command above is for prepending the $(srcdir) prefix onto each file in the user-specified list, in order to properly support VPATH builds. This code uses a shell for statement to split the list into individual files, prepend $(srcdir), and then reassemble the list.

*NOTE: It's interesting to note that this file list munging process could have been done in a half-line of GNU make-specific code, but Automake is designed to generate makefiles that can be executed by many older make programs.*

The part that actually does the work is found in one line, near the bottom. To make it simpler to read, I'll reformat this example, removing the cruft:

{% highlight shell %}
...
JAVAROOT = $(top_builddir)
JAVAC = javac
CLASSPATH_ENV = CLASSPATH=$(JAVAROOT):$(srcdir)/$(JAVAROOT):$$CLASSPATH
...
classdist_noinst.stamp: $(dist_noinst_JAVA)
...
$(CLASSPATH_ENV) $(JAVAC) -d $(JAVAROOT) $(AM_JAVACFLAGS) $(JAVACFLAGS) $$list
...
{% endhighlight %}

You may have noticed Automake's use of a stamp file here. This is done because the single $(JAVAC) command generates several .class files from several .java files. Rather than just pick one of these at random to use in the rule, Automake generates and uses a stamp file. This is important to know, because using a stamp file in the rule causes make to ignore the associations between individual .class files and their corresponding .java files. That is, if you delete a .class file, the rules in the Makefile will not cause it to be rebuilt. The only way to cause the re-execution of the $(JAVA) command is to either modify one or more of the .java files, thereby causing their timestamps to become newer than that of the the stamp file, or to delete the stamp file entirely.

The variables used in the build environment, and on the command line include JAVAROOT, JAVAC, JAVACFLAGS, AM_JAVACFLAGS and CLASSPATH_ENV. Each of these may be specified by the developer in the Makefile.am file. If they're not specified, then the defaults you see in this example are used instead. Where you don't see a default value set, you may assume the default value is empty.

One important point about this code is that all of the files specified in the JAVA primary list are compiled using a single command line, which could pose a problem on systems with limited command line lengths. If you find you have such a problem, you may have to develop your own make rules for building Java classes. Given the limited support that Automake currently provides, this isn't really a very daunting task.

The CLASSPATH_ENV variables sets the Java classpath environment variable for the javac command such that it contains the contents of JAVAROOT ($(top_builddir), by default), the same value prefixed with $(srcdir), and then any class path that might be specified in the environment by the user.

The JAVAC variable contains javac by default. The hope here is that javac can be found in the system path. The AM_JAVACFLAGS variable may be set in the Makefile.am file by the developer. As usual, the non-Automake version of this variable (JAVACFLAGS) is considered a "user" variable, and shouldn't be set in makefiles.

The JAVAROOT variable is used to specify the location of the java root directory, which is where the Java compiler will expect to find the start of packages directory hierarchies belonging to your project.

This is fine as far as it goes, but it doesn't go nearly far enough. In this (relatively simple) project, I also need to generate JNI header files using the javah utility, and I need to generate a .jar file from the .class files built from my Java sources. Automake-provided Java support doesn't even begin to handle these tasks. So I'll have to do the rest with hand-coded make rules. I'll start with Autoconf macros to ensure that I have a good Java build environment.

**Using ac-archive macros**

I did a little hunting around on the internet, and found that the ac-archive project on sourceforge.net does in fact supply Autoconf macros that come close to what I need in order to ensure that I have a good Java development environment. I downloaded the latest ac-archive source package, and just hand-installed the .m4 files that I needed into my xflaim/m4 directory.

Then I modified them (and their names) such that they work the way my AC_PROG_TRY_DOXYGEN macro works. I wanted to locate Java tools if they exist, but be able to continue without them if they're missing. Given the current politics surrounding the existence of Java tools in GNU/Linux distributions at this time, this is probably a wise approach.

*NOTE: The other way to use the ac-archive package is to actually install it on your system, which will place the ac-archive .m4 files into the /usr/(local/)share/ac-archive directory. The documentation for ac-archive provides instructions on how you might pass flags to the aclocal utility from within your project's top-level Makefile.am file that tell it how to access the installed ac-archive macros during an execution of autoreconf, or aclocal.*

I created the following macros and files from those found in the ac-archive:

- AC_PROG_TRY_JAVAC is defined in ac_prog_try_javac.m4 and ac_prog_javac_works.m4
- AC_PROG_TRY_JAVAH is defined in ac_prog_try_javah.m4
- AC_PROG_TRY_JAVADOC is defined in ac_prog_try_javadoc.m4
- AC_PROG_TRY_JAR is defined in ac_prog_try_jar.m4
- AC_PROG_TRY_CSC is defined in ac_prog_try_csc.m4 and ac_prog_csc_works.m4
- AC_PROG_TRY_CSVM is defined in ac_prog_try_csvm.m4 and ac_prog_csvm_works.m4

With only a little more effort, I was also able to create the CSharp macros I needed to accomplish the same tasks for the CSharp language bindings. I'll discuss CSharp in the next section. Here's a portion of the xflaim configure.ac file, repeated here for your information:

{% highlight shell %}
...
# Checks for optional programs.
AC_PROG_TRY_CSC
AC_PROG_TRY_CSVM
AC_PROG_TRY_JAVAC
AC_PROG_TRY_JAVAH
AC_PROG_TRY_JAVADOC
AC_PROG_TRY_JAR
...
# Check for Java compiler.
have_java=yes 
if test -z "$JAVAC"; then have_java=no; fi
if test -z "$JAVAH"; then have_java=no; fi
if test -z "$JAR"; then have_java=no; fi
if test "x$have_java" = xno; then
    echo "-----------------------------------------"
    echo " Some Java tools not found - continuing"
    echo " without XFLAIM JNI support."
    echo "-----------------------------------------"
fi
AM_CONDITIONAL([HAVE_JAVA], [test "x$have_java" = xyes])

# Check for CSharp compiler.
if test -z "$CSC"; then
    echo "-----------------------------------------"
    echo " No CSharp compiler found - continuing"
    echo " without XFLAIM CSHARP support."
    echo "-----------------------------------------"
fi
AM_CONDITIONAL([HAVE_CSHARP], [test -n "$CSC"])
...
{% endhighlight %}

These macros set the CSC, CSVM, JAVAC, JAVAH, JAVADOC and JAR variables to the location of their respective CSharp and Java tools, and then substitute them into the xflaim project's Makefile.in templates using AC_SUBST. If any of these variables are already set in the user's environment when the configure script is executed, their values are left untouched, allowing the user to override the values that would have been set by the macros.

I also added some shell code to set a variable, have_java to either yes or no, depending on whether or not all three tools could be found. If they are found, have_java becomes yes, which fact is later used in the call to AM_CONDITIONAL. Recall that this Automake macro conditionally sets the HAVE_JAVA variable, which is later used in xflaim/src/Makefile.am file to conditionally build the java sub-directory hierarchy.

**Canonical system information**

The only non-obvious bit of information you need to know about using these ac-archive extensions is that they rely on the built-in Autoconf macro, AC_CANONICAL_TARGET. Autoconf provides a way to automatically expand any existing macros inside the definition of a macro, so that macros required by the one being defined can be made available immediately. However, if AC_CANONICAL_TARGET is not used before certain other macros (including, unfortunately, LT_INIT), then autoreconf will generate about a dozen warning messages.

To alleviate these warnings, I added AC_CANONICAL_SYSTEM to my top-level and xflaim-level configure.ac files, immediately after the call to AC_INIT. As I mentioned earlier in this chapter, this macro and those that it calls, AC_CANONICAL_BUILD, AC_CANONICAL_HOST and AC_CANONICAL_TARGET, are designed to ensure that the $host, $build and $target environment variables are defined by the configure script, such that they contain appropriate values describing the user's host, build and target systems.

These variables contain canonical values for the host, build and target CPU, vendor and operating system. Values like these are very useful to extension macros. If a macro can assume these variables are set properly, then it saves quite a bit of code duplication in the macro definition.

The values of these variables are calculated using two helper scripts, config.guess and config.sub, which are distributed with Autoconf. The config.guess script uses a combination of uname commands to ferret out information about the host system, and munge it into a canonical value. The config.sub script is used to reformat host, build and target information specified by the user on the configure command line into a canonical value.

The key point here, however, is that I had to use the AC_CANONICAL_SYSTEM macro well before I called the ac-archive extension macros in my configure.ac script.

**The xflaim/java directory structure**

The original source layout had the Java JNI and CSharp native sources located in entirely different directory structures than xflaim/src. The JNI sources were located in xflaim/java/jni, and the CSharp native sources were located in xflaim/csharp/xflaim. While Automake has no problem generating rules for accessing files well outside the current directory hierarchy, I find it a bit silly to put these files so far away from the only library they can really belong to. Thus, I broke my own rule of thumb about not rearranging files in this case. I moved the contents of these two directories to directories under xflaim/src. I named the JNI directory xflaim/src/java and the CSharp native sources directory xflaim/src/cs.

{% highlight shell %}
flaim
    xflaim
        src
        cs
        java
            wrapper
                xflaim
{% endhighlight %}

As you can see, I also added a wrapper directory beneath the java directory, in which I rooted the xflaim wrapper package hierarchy. Since the Java xflaim wrapper classes are part of the Java xflaim package, they have to be located in a directory called xflaim. Nevertheless, the build happens in the wrapper directory. There are no build files found in the wrapper/xflaim directory, or any directories below that point.

Note that it doesn't matter how deep your package hierarchy is. You will still build the java classes in the wrapper directory--this is the JAVAROOT directory for this project.

**The xflaim/src/Makefile.am file**

At this point the configure.ac script is doing about all it can for me to ensure that I have a good Java build environment. If I have a good Java build environment, my build system will be able to generate my JNI wrapper classes and header files, and build my C++ JNI sources. If my end user's system doesn't provide these tools, then she simply can't build or link in the JNI language bindings on that host.

Have a look at the xflaim/src/Makefile.am file, and examine the portions that are relevant to building the Java and CSharp language bindings:

{% highlight shell %}
SUBDIRS = 

if HAVE_JAVA
    SUBDIRS += java
    JNI_LIBADD=java/libxfjni.la
endif

if HAVE_CSHARP
    SUBDIRS += cs
    CSI_LIBADD=cs/libxfcsi.la
endif

SUBDIRS += .
...
libxflaim_la_LIBADD = $(JNI_LIBADD) $(CSI_LIBADD) $(FTK_LTLIB)
...
{% endhighlight %}

I've already explained the use of the conditionals to ensure that the java and cs directories only get built if the proper conditions are met. You can now see how this fits into the build system I've created so far.

Notice that I'm also conditionally defining two new library variables. If I can build the Java language bindings, then the JNI_LIBADD variable will refer to the library that is built in the java directory. If I can build the CSharp language bindings, then the CSI_LIBADD variable will refer to the library that is built in the cs directory. In either case, if the required tools are not found by the configure script, then the associated variable will remain undefined. When an undefined variable is referenced, it expands to nothing, so there's no harm in using it in the libxflaim_la_LIBADD variable.

**Building the JNI C++ sources**

Now, allow me to turn your attention to the xflaim/src/java/Makefile.am file:

{% highlight shell %}
SUBDIRS = wrapper

XFLAIM_INCLUDE=-I$(srcdir)/..

noinst_LTLIBRARIES = libxfjni.la

libxfjni_la_SOURCES = \
                    jbackup.cpp \
                    jdatavector.cpp \
                    jdb.cpp \
                    jdbsystem.cpp \
                    jdomnode.cpp \
                    jistream.cpp \
                    jniftk.cpp \
                    jniftk.h \
                    jnirestore.cpp \
                    jnirestore.h \
                    jnistatus.cpp \
                    jnistatus.h \
                    jostream.cpp \
                    jquery.cpp

libxfjni_la_CPPFLAGS = $(XFLAIM_INCLUDE) $(FTK_INCLUDE)
{% endhighlight %}

Again, I want the wrapper directory to be built first, because it will build the class files and JNI header files required by the JNI convenience library sources. This time, it's not conditional. If I've made it this far into the build hierarchy, then I know I have all the Java tools I need. This Makefile.am file simply builds a convenience library containing my JNI C++ interface functions.

Because of the way Libtool builds both shared and static libraries from the same sources, this convenience library will become part of both the xflaim shared and static libraries. The original build system makefile accounted for this by linking the JNI and CSharp native interface objects into only the shared library.

The fact that these libraries are added to both the shared and static xflaim libraries is not really a problem. Objects in a static library remain unused in applications or libraries linking to the static library, as long as code in those objects remain unreferenced. However, I'll admit that it's a bit of a "wart" on the side of my new build system.

**The Java wrapper classes and JNI headers**

Finally, the xflaim/src/java/wrapper/Makefile.am file takes us to the heart of the matter. I've tried many different configurations for building Java JNI wrappers, and this one always comes out on top. Here's the wrapper directory's Automake intput file:

{% highlight shell %}
JAVAROOT = .

jarfile = $(PACKAGE)jni-$(VERSION).jar
jardir = $(datadir)/java
pkgpath = xflaim
jhdrout = ..

$(jarfile): $(dist_noinst_JAVA) 
    $(JAR) cf $(JARFLAGS) $@ $(pkgpath)/*.class

jar_DATA = $(jarfile)

java-headers.stamp: $(dist_noinst_JAVA)
    @list="`echo $(dist_noinst_JAVA) |\
    sed -e 's|\.java||g' -e 's|/|.|g'`"; \
    for class in $$list; do \
        echo "$(JAVAH) -jni -d $(jhdrout)\
        $(JAVAHFLAGS) $$class"; \
        $(JAVAH) -jni -d $(jhdrout)\
        $(JAVAHFLAGS) $$class; \
    done
    
    @echo "JNI headers generated" > java-headers.stamp

all-local: java-headers.stamp

CLEANFILES = $(jarfile) $(pkgpath)/*.class java-headers.stamp $(jhdrout)/xflaim_*.h

dist_noinst_JAVA = $(pkgpath)/BackupClient.java $(pkgpath)/Backup.java \
                   ...
                   $(pkgpath)/XFlaimException.java $(pkgpath)/XPathAxis.java
{% endhighlight %}

I've set the JAVAROOT variable to DOT (.), mainly because I want Automake to be able to tell the Java compiler that this is where the package hierarchy begins. The xflaim Java wrapper classes are found in the xflaim package. The default value for JAVAROOT is $(top_builddir), which would have the wrapper class belong to the xflaim.src.java.wrapper.xflaim package. That's not right.

I then created a variable called jarfile, deriving its value from $(PACKAGE) and $(VERSION). This is how the destdir variable is derived also, from which the name of the tarball comes. A make rule indicates how the .jar file should be built. Here, I'm using the JAR variable, whose value was calculated for me by the results of the AC_PROG_TRY_JAR macro in the configure script. This rule is fairly straight forward.

I've defined a new installation variable called jardir--the place where .jar files are to be installed, presumably. And I've used it as the prefix for a DATA primary. Any files that Automake doesn't understand--basically, any files that you build using your own rules--are just considered by Automake to be data files, and are installed as such.

I'm using another stamp file in the rule that builds the JNI header files from the .class files. I'm doing this for the same reason that Automake used a stamp file in the rule that it uses to build .class files from .java source files.

This is the most complex part of this makefile, so I'll try to break it into simple pieces. The rule states that the stamp file depends on the files listed in the dist_noinst_JAVA variable. The command is a bit of complex shell script that strips the .java extensions from the file list, and converts all the SLASH characters in to DOT characters. The reason for this is that the javah utility wants a list of class names, not a list of file names. The last line, of course, generates the stamp file.

Finally, I hooked my java-headers.stamp target into the all target by adding it as a dependency to the all-local target. When the all target (the default for all Automake-generated makefiles) is executed in this makefile, java-headers.stamp will be built, along with the JNI headers.

Here, I've also added the .jar file, all of the .class files, the java-headers.stamp file and all of the generated JNI header files to the CLEANFILES variable, so that Automake will clean them up for me when make clean is executed on this makefile. Again, I can use the CLEANFILES variable here because I'm not trying to delete any directories.

**A caveat about using the JAVA primary**

There's one important caveat to using the JAVA primary. You may only define one JAVA primary variable per Makefile.am file. The reason for this is that multiple classes may be generated from a single .java file, and the only way to know which classes came from which .java file would be to parse the .java files. Rather than do this, Automake allows only one JAVA primary per file, so all .class files generated within a given build directory are installed in the location specified by the single JAVA primary variable prefix.

Realizing this gives me pause for thought. It seems that I've broken this rule by assuming in my java-headers.stamp rule that the source for class information is the list of files specified in the dist_noinst_JAVA variable. In reality, I should probably be looking in the current build directory for all .class files found after the rules for the JAVA primary are executed.

It's a good thing I don't need to install my JNI header files. I have no way of knowing what they're called from within my Makefile.am file! You should by now be able to see the problems that Autotools has with Java. In fact, these problems are not so much related to the poor design of Autotools, as they are the poor design of the Java language itself. This will become clear in the next section, as I cover the rules that build the CSharp native interfaces.

**Building the CSharp sources**
Returning now to the xflaim/src/cs directory brings us to a discussion of building source for a language for which Automake has *no* support: CSharp. Here's the Makefile.am file that I wrote for the cs directory:

{% highlight shell %}
SUBDIRS = wrapper

XFLAIM_INCLUDE=-I$(srcdir)/..

noinst_LTLIBRARIES = libxfcsi.la

libxfcsi_la_SOURCES = Backup.cpp DataVector.cpp Db.cpp DbInfo.cpp \
                      DbSystem.cpp DbSystemStats.cpp DOMNode.cpp \
                      IStream.cpp OStream.cpp Query.cpp

libxfcsi_la_CPPFLAGS = $(XFLAIM_INCLUDE) $(FTK_INCLUDE)
{% endhighlight %}

Not surprisingly, this looks almost identical to the Makefile.am file found in the xflaim/src/java directory. I'm building a simple convenience library from C++ source files found in this directory, just as I did in the java directory. As in the java version, this makefile is specifying a sub-directory called wrapper, which Automake builds first.

The wrapper/Makefile.am file looks like this:

{% highlight shell %}
EXTRA_DIST = xflaim cstest sample xflaim.ndoc

xfcs_sources = xflaim/BackupClient.cs xflaim/Backup.cs \
               ...
               xflaim/RestoreClient.cs xflaim/RestoreStatus.cs

cstest_sources = \
              cstest/BackupDbTest.cs \
              cstest/CacheTests.cs \
              ...
              cstest/StreamTests.cs \
              cstest/VectorTests.cs

TESTS = cstest_script
AM_CSCFLAGS = -d:mono -nologo -warn:4 -warnaserror+ -optimize+

#AM_CSCFLAGS += -debug+ -debug:full\
# -define:FLM_DEBUG

all-local: xflaim_csharp.dll

clean-local:
    -rm xflaim_csharp.dll xflaim_csharp.xml
    -rm cstest_script cstest.exe libxflaim.so
    -rm Output_Stream 
    -rm -rf abc backup test.*

check-local: cstest.exe cstest_script

install-exec-local:
    test -z "$(libdir)" || $(MKDIR_P) "$(DESTDIR)$(libdir)"
    $(INSTALL_PROGRAM) xflaim_csharp.dll "$(DESTDIR)$(libdir)"

install-data-local:
    test -z "$(docdir)" || $(MKDIR_P) "$(DESTDIR)$(docdir)"
    $(INSTALL_DATA) xflaim_csharp.xml "$(DESTDIR)$(docdir)"

uninstall-local:
    rm "$(DESTDIR)$(libdir)/xflaim_csharp.dll"
    rm "$(DESTDIR)$(docdir)/xflaim_csharp.xml"

xflaim_csharp.dll: $(xfcs_sources)
    @list1='$+'; list2=; \
    if test -n "$$list1"; then \
        for p in $$list1; do \
            if test -f $$p; then d=; \
            else d="$(srcdir)/"; fi; \
            list2="$$list2 $$d$$p"; \

        done; \
        echo '$(CSC) -target:library $(AM_CSCFLAGS) $(CSCFLAGS) -out:$@ -doc:$(@:.dll=.xml) '
             "$$list2"; $(CSC) -target:library $(AM_CSCFLAGS)\
             $(CSCFLAGS) -out:$@ -doc:$(@:.dll=.xml) $$list2; \

    else :;
    fi

cstest.exe: xflaim_csharp.dll $(cstest_sources)
    @list1='$(cstest_sources)'; \
    list2=; if test -n "$$list1"; then \
        for p in $$list1; do \
            if test -f $$p; then d=; \
            else d="$(srcdir)/"; fi; \
            list2="$$list2 $$d$$p"; \
        done; \

        echo '$(CSC) $(AM_CSCFLAGS) $(CSCFLAGS)\
        -out:$@ '"$$list2"'\
        -reference:xflaim_csharp.dll'; \
        $(CSC) $(AM_CSCFLAGS) $(CSCFLAGS)\
        -out:$@ $$list2\

        -reference:xflaim_csharp.dll; \
    else :; fi

libxflaim.so:
    $(LN_S) ../../.libs/libxflaim.so libxflaim.so

cstest_script: cstest.exe libxflaim.so
    echo "#!/bin/sh" > cstest_script
    echo "$(CSVM) cstest.exe" >> cstest_script
    chmod 0755 cstest_script
{% endhighlight %}

The default target for this Makefile.am file is, of course, the all target. I've hooked the all target with my own code by implementing the all-local target, which depends on a file named xflaim_csharp.dll.

*NOTE: This executable file name may be a bit confusing to those who are new to CSharp. In essence, the creators of CSharp (Microsoft) designed the CSharp VM to execute Microsoft native (or almost native) binaries. In porting the CSharp virtual machine to Unix, the Mono team decided against breaking the naming conventions defined by Microsoft, so that Microsoft generated programs could be executed by the Mono CSharp virtual machine implementation. Nevertheless, it still suffers from problems that need to be managed occasionally by name-mapping configuration files.*

{% highlight shell %}
...
xfcs_sources = ...
...
all-local: xflaim_csharp.dll
...
xflaim_csharp.dll: $(xfcs_sources)
    @list1='$+'; list2=; \
    if test -n "$$list1"; then \
        for p in $$list1; do \
            if test -f $$p; then d=; \
            else d="$(srcdir)/"; fi; \
            list2="$$list2 $$d$$p"; \
        done; \

        echo '$(CSC) -target:library\
        $(AM_CSCFLAGS) $(CSCFLAGS) -out:$@\
        -doc:$(@:.dll=.xml) '"$$list2"; \
        $(CSC) -target:library $(AM_CSCFLAGS)\
        $(CSCFLAGS) -out:$@ -doc:$(@:.dll=.xml)\
        $$list2; \
    else :; fi
...
{% endhighlight %}

The xflaim_csharp.dll binary depends on the list of CSharp source files specified in the xfcs_sources variable. I take no credit for the commands in this rule. They're copied from the Automake-generated java/wrapper/Makefile, and slightly modified to build CSharp binaries from CSharp source files.

This isn't a lesson in building CSharp sources--the point here is that the default target is automatically built by hooking the all target via the all-local target.

This Makefile.am file also builds a set of unit tests in CSharp that test the CSharp language bindings. Here are the relevant portions of the file:

{% highlight shell %}
...
cstest_sources = ...

TESTS = cstest_script
...
check-local: cstest.exe cstest_script
...
cstest.exe: xflaim_csharp.dll $(cstest_sources)
    @list1='$(cstest_sources)'; \
    list2=; if test -n "$$list1"; then \
        for p in $$list1; do \
            if test -f $$p; then d=; \
            else d="$(srcdir)/"; fi; \
            list2="$$list2 $$d$$p"; \
        done; \

        echo '$(CSC) $(AM_CSCFLAGS) $(CSCFLAGS)\
        -out:$@ '"$$list2"'\
        -reference:xflaim_csharp.dll'; \
        $(CSC) $(AM_CSCFLAGS) $(CSCFLAGS)\
        -out:$@ $$list2\
        -reference:xflaim_csharp.dll; \
    else :; fi

libxflaim.so:
    $(LN_S) ../../.libs/libxflaim.so libxflaim.so

cstest_script: cstest.exe libxflaim.so
    echo "#!/bin/sh" > cstest_script
    echo "$(CSVM) cstest.exe" >> cstest_script
    chmod 0755 cstest_script
{% endhighlight %}

The test sources are built into a CSharp executable named cstest.exe. The rules state that cstest.exe depends on xflaim_csharp.dll and the source files. I again copied the commands from the rule for building xflaim_csharp.dll, and modified them for building CSharp programs.

Ultimately, the Automake-generated makefile will attempt to execute the scripts or executables listed in the TESTS variable, so the idea here is to ensure that all necessary components get built before these files are executed. The cstest_script is a script built for the sole purpose of executing the cstest.exe binary in the CSharp virtual machine referenced by the CSVM variable. This variable was defined in my configure script by the code generated by the AC_PROG_TRY_CSVM macro.

The script depends on the executable, and on a link to the libxflaim.so file. This file must be present in the current directory, or its location must be specified somehow on the mono ($CSVM) command line. I chose to simply create a link in the current directory to the location of the actual built library--located up a few directories, and then down into the xflaim/src/.libs directory.

**Manual installation**

Since I'm doing everything myself here, I can't rely on Automake to install files for me. I have to write my own installation rules. Here again are the relevant portions of the makefile:

{% highlight shell %}
...

install-exec-local:
    test -z "$(libdir)" || $(MKDIR_P) "$(DESTDIR)$(libdir)"
    $(INSTALL_PROGRAM) xflaim_csharp.dll "$(DESTDIR)$(libdir)"

install-data-local:
    test -z "$(docdir)" || $(MKDIR_P) "$(DESTDIR)$(docdir)"
    $(INSTALL_DATA) xflaim_csharp.xml "$(DESTDIR)$(docdir)"

uninstall-local:
    rm "$(DESTDIR)$(libdir)/xflaim_csharp.dll"
    rm "$(DESTDIR)$(docdir)/xflaim_csharp.xml"
...
{% endhighlight %}

Notee that, as per the rules defined in the GNU Coding Standards, the installation targets do not depend on the binaries they install. I don't want make install to build anything. If they haven't been built yet, I'll have to exit out of the root account, back into my own user account and build the binaries with make all first.

**Cleaning up again**

As usual, things must be cleaned up properly. The clean-local target handles this nicely for me:

{% highlight shell %}
...
clean-local:
    -rm xflaim_csharp.dll xflaim_csharp.xml
    -rm cstest_script cstest.exe libxflaim.so
    -rm Output_Stream 
    -rm -rf abc backup test.*
...
{% endhighlight %}

**Configuring compiler options**

The original GNU build system was doing a lot for the user. By specifying a list of auxiliary targets on the make command line, the user could indicate that she wanted a debug or release build, force a 32-bit build on a 64-bit system, indicate that she wanted to generate generic SPARC code on a Solaris sytem, etc.

Oddly, this turn-key approach to build systems is quite common in commercial code. Whereas, in open source code, the more common practice is to omit much of this framework, allowing the user to set her own options in the standard user variables, CC, CPP, CXX, CFLAGS, CXXFLAGS, CPPFLAGS and others. What's strange about this situation is that commercial software is developed by experts working in the industry, while open source software is often built and consumed by hobbyists. And yet the experts are the ones using the menu-driven rigid-options framework, while the hobbyists have to manually configure their compiler options.

I suppose the most reasonable explanation for this is that commercial software relies on carefully crafted builds that must be able to be duplicated. Open source hobbyists are more carefree, and would rather not give up the flexibility afforded by the lack of such turn-key systems.

To this end, I've added some of the options supported by the original GNU makefile-based build system, but left others out. Here's the portion of the configure.ac file that I'm talking about:

{% highlight shell %}
...
# Configure global pre-processor definitions.
AC_DEFINE([_REENTRANT], [], [Define for reentrant code])
AC_DEFINE([_LARGEFILE64_SOURCE], [], [Define for 64-bit data files])
AC_DEFINE([_LARGEFILE_SOURCE], [], [Define for 64-bit data files])

# Configure supported platforms' compiler and li...
case $host in
    sparc-*-solaris*)
        LDFLAGS="$LDFLAGS -R /usr/lib/lwp"
        if "x$CXX" != "xg++"; then
            if "x$debug" = xno; then
                CXXFLAGS="$CXXFLAGS -xO3"
            fi
            SUN_STUDIO=`"$CXX" -V | grep "Sun C++"`
            if "x$SUN_STUDIO" = "xSun C++"; then
                CXXFLAGS="$CXXFLAGS -errwarn=%all -errtags -erroff=hidef,inllargeuse,doubunder"
            fi
        fi
    ;;

    *-apple-darwin*)
        AC_DEFINE([OSX], [], [Define if building on Apple OSX.])
    ;;

    *-*-aix*)
        if "x$CXX" != "xg++"; then
            CXXFLAGS="$CXXFLAGS -qthreaded -qstrict"
        fi
    ;;

    *-*-hpux*)
        if "x$CXX" != "xg++"; then
            # Disable "Placement operator delete
            # invocation is not yet implemented" warning
            CXXFLAGS="$CXXFLAGS +W930"
        fi
    ;;
esac
...
{% endhighlight %}

Here, I've used the $host variable to determine the type of system for which I'm building. The config.guess and config.sub files are your friends here. If you need to write code like for your project, then you'll need to examine these files to find common traits for the processes and systems for which you'd like to set various compiler and linker options.

Note also that in each of these cases (except for the definition of the OSX preprocessor variable on Apple Darwin systems), I'm really only setting flags for native compilers. The GNU compiler tools seem to be able to handle any sort of code thrown at them without monkeying around with compiler options. This is a good thing, and a lesson could be learned by compiler vendors from this fact.

**Hooking Doxygen into the build process**
I wanted to generate documentation as part of my build process, if possible. That is, if the user has doxygen installed on her system, then the build system will use it to build doxygen documentation as part of the make all process. As I've already mentioned, I used the AM_CONDITIONAL macro to conditionally build the docs directory.

Now, relative to the xflaim project, this is probably not the right thing to do, as I want non-doxygen documentation to be installed even if doxygen isn't available. The right approach to this problem would be to have a doxygen directory beneath the docs directory that handles only generated documentation. The docs directory itself would be limited to simply installing existing documentation. I've combined them to save space in this book, but I'll probably fix this problem before committing my build system to the project.

For the FLAIM tool kit project, this configuration works fine for now, because there is no other documentation to be installed. I say "for now" because at some point in the future, someone may write some tool kit documentation, and then I'll have to move things around to get the end-user experience I want.

Doxygen uses a configuration file (often called doxyfile) to configure literally hundreds of doxygen options. This configuration file contains some information that is known to Autoconf. This sounds like the perfect opportunity to use an Autoconf-generated file. To this end, I've written a file called doxyfile.in that contains most of what a normal doxyfile would contain, except it also has a few Autoconf substitution variable references:

{% highlight shell %}
...
PROJECT_NAME           = @PACKAGE_NAME@
PROJECT_NUMBER         = @PACKAGE_VERSION@ 
...
STRIP_FROM_PATH        = @top_srcdir@
...
{% endhighlight %}

There are many other lines in this file, but they are all identical to the output file, so I've omitted them for the sake of space and clarity. The key here is that Autoconf will replace these values with those defined in configure.ac, and by Autoconf itself. If these values change in configure.ac, the generated file will be written with the new values. I've added a reference to ftk/docs/doxyfile to the AC_CONFIG_FILES list in ftk's configure.ac file. That's all it takes.

Here's the ftk/docs/Makefile.am file:

{% highlight shell %}
docpkg = $(PACKAGE_TARNAME)-doxy-$(PACKAGE_VERSION).tar.gz
doc_DATA = $(docpkg)
$(docpkg): doxygen.stamp
    tar chof - html | gzip -9 -c >$@

doxygen.stamp: doxyfile
    $(DOXYGEN) $(DOXYFLAGS) $<
    echo Timestamp > $@

CLEANFILES = doxywarn.txt doxygen.stamp $(docpkg)

clean-local:
    -rm -rf html
{% endhighlight %}

In this file, I've created a package name for the tarball that will contain the doxygen documentation files. It's basically the same as the distribution tarball for the ftk project, except that it contains the text -doxy after the package name.

I've also defined a doc_DATA variable containing the name of the doxygen tarball. This file will be installed in the $(docdir) directory, which by default is $(datarootdir)/doc/$PACKAGE_TARNAME. And $(datarootdir) is configured as $(prefix)/share, by default.

Note again here that the DATA primary brings with it significant Automake functionality--installation is managed automatically. And, while I must build the doxygen documentation package myself, the DATA primary automatically hooks the all target for me, so that my package is built when the user executes make all.

I'm using another stamp file here because doxygen generates literally hundreds of html files from my input file (and from the source tree). Rather then attempt to figure out a rational way to assign dependencies, I simply generate one stamp file, and then use that to determine whether or not the documentation is out of date.

*Note that this is wrong, but much simpler than attempting to list every source file used in the generation of the documentation as a dependency of the stamp file. (In fact, this is quite trivial in this project because the only source file currently containing documentation markup, and thus, listed in the doxyfile as an input file, is the flaimtk.h header file. However, this could easily change in the future.)*

For cleaning my generated files, I've used a combination of the CLEANFILES variable and a clean-local rule--just to show you that it can be done.

**Adding a new rpms target**

Adding a new non-standard target is a little different than hooking an existing target. In the first place, you don't need to use AM_CONDITIONAL and Autoconf checks to see if you have the tools you need. You may do everything from the Makefile.am file, if you wish. After all, if the user was building on a Debian system, why in the world did she type make rpms in the first place?! Nonetheless, you still have to account for the possibility that the user will experiment.

First, I created a directory called obs to contain the Makefile.am file for building RPM package files. OBS is an acronym for "Opensuse Build Service", which is an online package building service (found at [http://build.opensuse.org][build_opensuse])that I fell in love with almost as soon as it came out. I've had some experience building distro packages, and I can tell you, it's far less painful with the OBS than it is using more traditional techniques.

Furthermore, packages built with the OBS can be published on the OBS web site for others to access immediately after they're built (in this case, [http://software.opensuse.org/search][search]).

Building RPM package files is done using a configuration file, called a "spec" file, which is very much like the doxyfile is used to configure doxygen for a specific project. As with the doxyfile, the rpm spec file contains information that Autoconf knows about regarding the project package. So, I wrote an ftk.spec.in file, adding substitution variables where appropriate, and then I added another file reference to the AC_CONFIG_FILES macro. Here is the relevant portion of the ftk.spec.in file:

{% highlight shell %}
Name: @PACKAGE_TARNAME@
BuildRequires: gcc-c++ libstdc++ libstdc++-devel doxygen
Summary: FTK is the FLAIM cross-platfomr toolkit.
URL: http://forge.novell.com/modules/xfmod/project/?flaim
Version: @PACKAGE_VERSION@
Release: 1
License: GPL
Vendor: Novell, Inc.
Group: Development/Libraries/C and C++
Source: %{name}-%{version}.tar.gz
BuildRoot: %{_tmppath}/%{name}-%{version}-build
...
{% endhighlight %}

I used @PACKAGE_TARNAME@ and @PACKAGE_VERSION@. Now the tar name is not likely to change much over the life time of this project, but the version will change quite often. Without the Autoconf substitution mechanism, I'd have to remember to update this version number whenever I updated the version in the configure.ac file. Here's the obs/Makefile.am file:

{% highlight shell %}
rpmspec = $(PACKAGE_TARNAME).spec

rpmmacros =\
          --define='_rpmdir $(PWD)'\
          --define='_srcrpmdir $(PWD)'\
          --define='_sourcedir $(PWD)'\
          --define='_specdir $(PWD)'\
          --define='_builddir $(PWD)'

rpmopts = --nodeps --buildroot='$(PWD)/_rpm'

rpmcheck:
    @which rpmbuild &> /dev/null; \
    if [ $$? -ne 0 ]; then \
        echo "*** This make target requires an rpm-based linux distribution."; \
        (exit 1); exit 1; \
    fi



srcrpm: rpmcheck $(rpmspec)
    rpmbuild -bs $(rpmmacros) $(rpmopts) $(rpmspec)

rpms: rpmcheck $(rpmspec)
    rpmbuild -ba $(rpmmacros) $(rpmopts) $(rpmspec)

.PHONY: rpmcheck srcrpm rpms
{% endhighlight %}

Building RPM package files is rather simple, as you can see. The targets provided by this makefile include srcrpm and rpms. The rpmcheck target is only used internally. How can you tell? Well, you can't really tell from here. In order to find out which targets in a lower-level Makefile.am file are supported by a top-level build, you have to look at the top-level Makefile.am file:

{% highlight shell %}
...
rpms srcrpm: dist
    $(MAKE) -C obs $(AM_MAKEFLAGS) $@
    rpmarch=`rpm --showrc | grep ^build\ arch | sed 's/\(.*: \)\(.*\)/\2/'`; \
    test -z $$rpmarch || ( mv $$rpmarch/* .; rm -rf $$rpmarch )
    -rm -rf $(distdir)
...
.PHONY: srcrpm rpms
{% endhighlight %}

As you can see from the first command in this rule, when a user targets rpms or srcrpm from the top-level build directory, the commands are recursively passed on to the obs/Makefile. The remaining commands simply remove droppings left behind by the RPM build process that are simpler to remove at this level. (Try building an rpm sometime, and you'll see what I mean!)

Notice also that both of these top-level makefile targets depend on the dist target. That's because the RPM build process requires the distribution tarball. Adding it as a dependency simply ensures that the distribution tarball is there when the rpmbuild utility needs it.

**Summary**
While using Autotools, there are a myriad of details to manage, most of which, as they say in the free software world, "can wait for the next release!" The take-away lesson here is that a build system is never really finished. It should be incrementally improved over time, as you find time in your schedule to work on it. And it can be rewarding to do so.

I've shown you a number of new features--features I didn't cover directly in the earlier chapters on the individual tools. There are many many more features that I couldn't begin to cover. You'll need to study the GNU Autotools manuals to become truly proficient. At this point, it should be pretty simple to pick up this additional information yourself.


[ref1]: http://freesoftwaremagazine.com/books/autotools_a_guide_to_autoconf_automake_libtool/
[gnucodingstd]: https://www.gnu.org/prep/standards/
[fsstd]: http://www.pathname.com/fhs/
[makemanual]: https://www.gnu.org/software/make/manual/
[book_make2]: http://shop.oreilly.com/product/9780937175903.do
[book_make3]: http://shop.oreilly.com/product/9780596006105.do?CMP=AFC-ak_book&ATT=Managing+Projects+with+GNU+Make
[build_opensuse]: https://build.opensuse.org/
[search]: http://software.opensuse.org/search
[gnuautoconf]: https://www.gnu.org/software/autoconf/manual/
[gnuautomake]: https://www.gnu.org/software/automake/manual/
[gnulibtool]: https://www.gnu.org/software/libtool/manual/
[ref2]: http://www.sourceware.org/autobook/
[autotools]: https://www.lrde.epita.fr/~adl/autotools.html
[gnum4]: https://www.gnu.org/software/m4/manual/
[gnused]: https://www.gnu.org/software/sed/manual/
[ref3]: http://www.grymoire.com/Unix/
[ref4]: http://ac-archive.sourceforge.net/
