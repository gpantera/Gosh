# Gosh C++ Coding Style

By Robin Rowe 2016/12/18 rev. 2

## Version Numbering

1. Create a Version.h file that contains:

	\#define PROGRAM_VERSION "0.1"

2. Display the version # when the program runs. Use printf() in a console app or display it in the About box for a GUI app.

3. Bump the version number whenever you development is "done". That is, for each build turned over for testing. So, at the end of your sprint or end of a SQA clean-up.

## Naming Conventions

1. FirstCap class names.
2. camelCase variable names. 
3. ALL_CAPS for enums and #defined constants.
4. Braces aligned, not as C-style hanging.
5. Header include guards use same case as class name.

	// MyClass.h Short description here...
	
	\#ifndef MyClass_h
	
	\#define MyClass_h
	
	...
	\#endif

6. Avoid Microsoft-style Hungarian naming.
7. Avoid snake_case. Never leading or trailing underscores.
8. Name files the same as the class they contain:

	FirstCap.h
	FirstCap.cpp

## Braces

Use C++ aligned braces, not C-style hanging braces.

## Complexity and Code Flow

Elegant simplicity is what's left after removing unnecessary complexity. 

Avoid excessively clever solutions, especially when it relies on obscure C++ language features that will make it difficult for other programmers to read your code.

Early return: Use if-not/return C++ code flow, not C-style if/else-return design.

	const bool ok = Foo();
	if(!ok)
	{    return;
	}
	Bar();

Not...

	if(ok)
	{    Bar();
	}
	else
	{    return;
	}

## Class Layout

1. Use comma-first in initializer (only).

	class Point
	{	int x;
		int y;
		int z;
	public:
		Point()
		:	x(0)
		,	y(0)
		,	z(0)
		{}
	};

2. Organize class members with private members first. This is the default for classes, so no need to specify private. (Private members first because it makes code review go faster.)

	class Foo
	{	int x;
		void DangerousBar();
	public:
		Foo();
		Bar();
	};

## OOP

Use object-oriented programming. 

When you write a constructor or Open() method for a class that calls several methods on the same member pointer, that is a big hint that your design isn't OOP enough. Move those calls into the constructor of a new class. Any class with an Open() or Close() method that you add functionality should probably be encapsulated in a new class.

In a GUI app, we typically have an App class and an AppMenu class.

Use override and final wherever appropriate. 

Delete the copy constructor wherever appropriate:

	MyClass(const MyClass&) = delete;

## Exceptions

Avoid them. Never throw. Use return false instead.

Never specify throw in a function declaration.

## Command-line Args

We use libunistd portable::CommandLine for parsing. When creating a new program don't implement command-line parsing, simply create variables with the best default values.
 
## Comments

1. Don't comment out large sections of code with // or /* */.

To comment out large sections of code, for example, because you intend to remove the code later, use preprocessor directive.

	#if 0
	
	#endif

2. Don't comment-out or remove function parameter names to silence compiler warnings about unused variables in function parameters. Cast to void instead.

	int Foo(int x,int y)
	{	(void) y;
		return x;
	}

Of course, it's generally best to remove unused parameters entirely.	
	
## Memory Management

1. Avoid excessive calls to new. Don't create a class with lots of pointer members and calls to new when you could simply new the whole class, members and all.

2. Use unique_ptr to avoid memory leaks.

3. Avoid excessive use of static class members. (It can become tightly coupled code that's difficult to refactor later.)

## Const Correctness

Make code const-correct, that is, use const wherever feasible. 

## main()

Name the file main.cpp that contains main(). (The name of the program may change later.)

Write a class App class that encapsulates the entire application. Avoid global variables by putting them in App.

C++ example:

	#include "App.h"
	
	int main(int argc,char* argv[])
	{	static App app;
		return app.Run(argc,argv);
	}

Qt example:

	int main(int argc, char *argv[])
	{   QCoreApplication app(argc, argv);
		static Server server;
		if(!server.Open(argc,argv))
		{   return -1;
		}
		QObject::connect(&server, &Server::closed, &app, &QCoreApplication::quit);
		return app.exec();
	}

## Third-Party Libs

1. We created the open source library libunistd (available on github) that enables Linux POSIX-compatible code to compile on Windows. It also has a set of classes for writing portable apps in a more C++ way than using C-based POSIX. The libunistd library is almost entirely header-only, quite lightweight.

2. Don't use Boost or other 3rd-party libraries unless directed to do so.

## Threads

1. Use std::thread where needed to make app responsive.

2. Avoid creating more than a few threads. 

3. Use thread(Main,this) to launch a thread on yourself:

	class Foo
	{   std::mutex fooMutex;
		std::condition_variable fooCondition;
		bool isGo;
		void Bar();
		static void Main(Foo* self)
		{   self->Run();
		}
		void Run()
		{	while(isGo)
			{   std::unique_lock<std::mutex> lock(fooMutex);
				fooCondition.wait(lock);
				Bar();
		}	}
	public:
		Foo();
		void Start()
		{   if(isGo)
			{   return false;
			}
			isGo=true;
			std::thread worker(Main,this);
			worker.detach();
		}
		void Wake()
		{	fooCondition.notify_one();
		}
		void Stop()
		{   isGo=false;
			Wake();
		}
	};

Use unique_lock in your thread loop, lock_guard other threads.

4. Avoid recursive locks.

## Networking

Use POSIX htonl() or relevant functions in libunistd to keep endian consistent.

## Build System

Use cmake, unless project is Qt-based. For Qt projects, use its qmake build system, not cmake. 

Set .gitignore to ignore sln and vcproj files. Those should be generated by cmake, not checked in.

Project should build on Windows without making changes to sln file created by cmake. When a project has open source dependencies, use DownloadProject (https://github.com/Crascit/DownloadProject) to make those auto-download. Like this in CMakeLists.txt...

	# download/include/build external projects
	include(DownloadProject)
	## download unistd
	download_project(PROJ                libunistd
					 PREFIX              "${CMAKE_BINARY_DIR}/libunistd/"
					 GIT_REPOSITORY https://github.com/robinrowe/libunistd.git
					 GIT_TAG             master
					 ${UPDATE_DISCONNECTED_IF_AVAILABLE}
	)
	include_directories(SYSTEM "${libunistd_SOURCE_DIR}")
	add_definitions( -DCMAKE_SOURCE_DIR="${CMAKE_SOURCE_DIR}" )

	if(WIN32)
	include_directories(SYSTEM "${libunistd_SOURCE_DIR}/vcpp")
	endif

## Use git

Save your changes to the master branch:

	git pull
	git status
	git add SomeFile.cpp
	git commit -a -m "SomeFile class does something"
	git push

Create an appropriate .gitignore file. If everything seems fine, but you can't push, that may be because we need to unprotect the master branch in gitlab.

-000-
