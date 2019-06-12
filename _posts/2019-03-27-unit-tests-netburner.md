---
layout: post
comments: true
title: How I run unit tests on NetBurner MOD5441X
excerpt: "An example of using the lest framework on NetBurner MOD5441X"
date: 2019-03-27 9:00:00
tags: netburner, unittesting, rtos
---

![](/assets/images/nb_unit_testing_cover_p1.jpg)

At my job, I participate in a large long-term project that uses the single board computer [NetBurner MOD5441X](https://www.netburner.com/products/system-on-modules/mod5441x/).

In this project all code was written only under MOD5441X and was tested exclusively by hand (often with an oscilloscope).
That guy who wrote this, by the way, has already left our company.

Now, practically every class needs to be refactored, and we need to at least somehow automatically monitor the regression in order to be sure that refactoring does not change the behavior of the system.

This project is essentially a large, unwieldy piece of legacy code with numerous smells and poor documentation without any automated tests!

I will show with simple examples how we added unit tests in our project.

<!--more-->

In my opinion, the best strategy for testing embedded systems is to write code cross-platform from the start of the project, mock code that works with real hardware, and test it directly on a PC. In order to build correct expectations of the operation of mock objects, you need to manually or automatically run certain unit tests on real hardware.

MOD5441X is running under modified µC/OS-II.

I needed a unit testing framework(s) that would satisfy the following requirements:
  - open source,
  - supports mock-objects,
  - cross-platform,
  - without linking to pthreads, because netburner's compiler compiled in single thread model,
  - preferably header-only,
  - supports C++11,
  - it has examples and detailed documentation,

However, looking at the list of frameworks in [Wikipedia](https://en.wikipedia.org/wiki/List_of_unit_testing_frameworks#C++), it became clear that we would have to find a pair of frameworks: unit-testing + mocking, which could work together.

After reviewing the comparative [article](https://lvee.org/en/abstracts/171) and trying several options, I settled on [lest](https://github.com/martinmoene/lest "lest") and [trompeloeil](https://github.com/rollbear/trompeloeil "trompeloeil") frameworks.

In this post I will adapt examples of test cases so that they can be executed on a netburner. The project was built in NBEclipse 2.8.7 with `lest` framework v1.33.3.
After including the header `lest/lest.hpp` to the project and copying the contents of one of the [examples](https://github.com/martinmoene/lest/blob/master/example/01-basic.cpp) to tests.cpp:

```
// tests.cpp
#include <ucos.h>
#include "lest/lest.hpp"
#include "tests.h"

const lest::test specification[] = {

    CASE( "Empty std::string has length zero (succeed)" ) {
        EXPECT(     0u == std::string(  ).length() );
        EXPECT(     0u == std::string("").length() );
        EXPECT_NOT( 0u <  std::string("").length() );
    },

    CASE( "Text compares lexically (fail)" ) {
        EXPECT(std::string("hello") == std::string("world") );
    },

    CASE( "Unexpected exception is reported" ) {
        EXPECT( (throw std::runtime_error("surprise!"), true) );
    },

    CASE( "Unspecified expected exception is captured" ) {
        EXPECT_THROWS( throw std::runtime_error("surprise!") );
    },

    CASE( "Specified expected exception is captured" ) {
        EXPECT_THROWS_AS( throw std::bad_alloc(), std::bad_alloc );
    },

    CASE( "Expected exception is reported missing" ) {
        EXPECT_THROWS( true );
    },

    CASE( "Specific expected exception is reported missing" ) {
        EXPECT_THROWS_AS( true, std::runtime_error );
    },
};

void test_main() {
    const std::vector<std::string> args{"-v", "-p" };
    lest::run( specification, args, std::cout );
}
```

main.cpp:
```
// main.cpp
#include <predef.h>
#include <ctype.h>
#include <startnet.h>
#include <autoupdate.h>
#include <smarttrap.h>
#include <taskmon.h>
// ...

void UserMain(void * pd) {
    InitializeStack();
    OSChangePrio(MAIN_PRIO);
    EnableAutoUpdate();
    EnableTaskMonitor();
    // ...
    test_main();                    // <== run test cases here
    iprintf("Testing is done\n");
    while (1) {
        OSTimeDly(TICKS_PER_SECOND);
    }
}
```

everything works out of the box:

[![](/assets/images/only_lest_output.png)](/assets/images/only_lest_output.png)

As you can see the output to the ExtraPuTTY console, test cases are executed as required.
But I do not recommend you overwhelm the console with diagnostic messages for test cases that pass, because this is a TDD anti-pattern called ["The LoudMouth"](http://archive.li/3acB). I display all diagnostic messages only for demonstration purposes.

Let's add more examples.
For example, rewrite the test cases for built-in RTC from [some NetBurner's guy](https://gist.github.com/fstanley/c13b04d8ca7510320135) using `lest`:

```
// ...
#include <iomanip>
#include <ucos.h>
#include <mcf5441x_rtc.h>
#include <ctime>
#include <constants.h>

// ...

namespace {
	// ...
	std::string incrementRTC(const char* setValue, int delay) {
		struct tm set_tm, incremented_tm;
		strptime(setValue, "%D %T %w %j", &set_tm);
		OSTimeDly(1); // Ensure we set the RTC as close to tick as possible
		MCF541X_RTCSetTime(set_tm);
		OSTimeDly(delay * TICKS_PER_SECOND);
		MCF541X_RTCGetTime(incremented_tm);
		std::ostringstream oss;
		oss << std::put_time(&incremented_tm, "%D %T %w %j");
		return oss.str();
	}
}

const lest::test specification[] = {

    CASE( "Expected minute increment" "[RTC]" ) {
    	EXPECT(incrementRTC("05/24/13 16:45:58 5 144", 4) == "05/24/13 16:46:02 5 144");
    },

    CASE( "Expected hour increment" "[RTC]"  ) {
    	EXPECT(incrementRTC("05/24/13 16:45:58 5 144", 4) == "05/24/13 16:46:02 5 144");
    	EXPECT(incrementRTC("12/10/14 12:59:58 3 344", 4) == "12/10/14 13:00:02 3 344");
    },

	CASE( "Expected day increment" "[RTC]" ) {
		EXPECT(incrementRTC("02/15/17 23:59:57 3 46", 4) == "02/16/17 00:00:01 4 047");
		EXPECT(incrementRTC("04/24/16 23:59:57 6 115", 4) == "04/25/16 00:00:01 0 116");
	},

	CASE( "Expected month increment" "[RTC]" ) {
		EXPECT(incrementRTC("01/31/12 23:59:59 2 31", 4) == "02/01/12 00:00:03 3 032");
		EXPECT(incrementRTC("11/30/12 23:59:59 5 335", 4) == "12/01/12 00:00:03 6 336");
	},

	CASE( "Expected year increment" "[RTC]" ) {
		EXPECT(incrementRTC("12/31/16 23:59:58 6 366", 4) == "01/01/17 00:00:02 0 001");
		EXPECT(incrementRTC("12/31/85 23:59:58 6 365", 4) == "01/01/86 00:00:02 0 001");
	},

	CASE( "Expected leap day increment" "[RTC]" ) {
		EXPECT(incrementRTC("02/28/12 23:59:59 2 59", 4) == "02/29/12 00:00:03 3 060");
		EXPECT(incrementRTC("02/29/12 23:59:59 3 60", 4) == "03/01/12 00:00:03 4 061");
		EXPECT(incrementRTC("02/28/13 23:59:59 4 59", 4) == "03/01/13 00:00:03 5 060");
		EXPECT(incrementRTC("02/28/14 23:59:59 5 59", 4) == "03/01/14 00:00:03 6 060");
		EXPECT(incrementRTC("02/28/15 23:59:59 6 59", 4) == "03/01/15 00:00:03 0 060");
	},
};

// ...
```

Yes, I know, rewritten test cases differ slightly from the original ones. Of course, I could write a separate function to convert the format string to the tm structure, but I was too lazy to do it.

At first, look at the output from original code:

[![](/assets/images/original_rtc_unit_test_output.png)](/assets/images/original_rtc_unit_test_output.png)

Oops, some tests did not pass, because, perhaps, netburner works on our custom board.

Ok, now build our code and upload it:

[![](/assets/images/rewritten_fstanley_testcases.jpg)](/assets/images/rewritten_fstanley_testcases.jpg)

The same result, but it's displayed in more convenient way (at least for me).
Workaround for failed test cases is to add time delay before the first test case or move rtc test cases to the end of `lest::test specification[]` array. Now it works well:

[![](/assets/images/fixed_rtc_test_cases.jpg)](/assets/images/fixed_rtc_test_cases.jpg)

If accidentally or for some other reasons the RTC code is broken, then the corresponding test case will fall.

[Project on GitHub](https://github.com/dmitofclubs/netburner-unit-testing-examples)
