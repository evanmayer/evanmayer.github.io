# Table of Contents

* [How I learned Arduino is Little-Endian](#2023-02-21)
* [Version Control of Large Files with `git-lfs`](#2022-07-30)
* [How to write unit tests](#2022-07-19)
* [Is my application "real-time"?](#2022-07-14)

# How I learned Arduino is Little-Endian<a name="2023-02-21"></a>

---

*2023-02-21*

I thought it would be interesting to create a piece of hardware that emulates the function of a gyroscope, so I could use it in testing without having expensive flight hardware around, program it to send arbitrary gyro signals to the flight software, and learn more about the flight software.

I decided to slice into the flight software at the serial interface. There are a couple of ways I know how to approach this:

1. [Set up a pseudoterminal](https://linux.die.net/man/4/ptmx) using another C program (or another thread in the current program) and send bytes through that to emulate bytes from a real serial port
2. Create a device that emulates the gyros and attach it to the real serial port

\1. is definitely possible; I do this as part of a unit test already. Part of a `cmocka` fixture helps with mocking stepper motors that talk over serial:

```c
// Spoof actuator bus: open a pseudoterminal and give it to the bus init,
// where the attributes (baud, etc.) will be set.
int fd = getpt();
char *ttyName;
ttyName = ptsname(fd);
if (-1 == unlockpt(fd)) {
    fail_msg("Failed to open pseudoterminal in SetupEzBus(): %s", strerror(errno));
}
```

From that point on, the program would just use the created pseudoterminal's name, `ttyName`, to open a connection, configure parameters, and start talking. We'd need to symlink whatever name we've been given in `ttyName` to tie it to the port the flight code expects to talk to the gyros on, `/dev/ttyGYRO0`/`/dev/ttyGYRO1`, which is a little more difficult on the fly, but doable.

\2. should be possible, but I've never done it before, and it seems way cooler. This should be possible with any Arduino with a high enough clock speed to support higher baudrates, like the required 921600 of the gyros.

I wrote this code to write gyro packets and send them out via serial:

{% raw %}
```c
#include <CRC.h>


// The gyro packet set by a KVH dsp1760
typedef union
{
    struct
    {
        uint32_t header;
        union
        {
            float x;  /// X-axis rotational speed or incremental angle in radians or degrees (config dependent)
            uint32_t x_raw;
        };
        union
        {
            float y;  /// Y-axis rotational speed or incremental angle in radians or degrees (config dependent)
            uint32_t y_raw;
        };
        union
        {
            float z;  /// Z-axis rotational speed or incremental angle in radians or degrees (config dependent)
            uint32_t z_raw;
        };

        uint8_t reserved[12]; /// Unused

        uint8_t status;     /// status per gyro.  Use DSP1760_STATUS_MASK* 1=valid data, 0=invalid data
        uint8_t sequence;   /// 0-127 incrementing sequence of packets
        int16_t temp;       /// Temperature data, rounded to nearest whole degree (C or F selectable)
        uint32_t crc;
    }__attribute__((packed));
    uint8_t raw_data[36];
} dsp1760_std_t;

// a basis for each packet we will send
dsp1760_std_t packet_primitive = {{
    .header = 0xFE81FF55,
    .x_raw = 0,
    .y_raw = 0,
    .z_raw = 0,
    .reserved = {42,42,42,42,42,42,42,42,42,42,42,42,},
    .status = 1,
    .sequence = 0,
    .temp = 42,
    .crc = 0,
}};

dsp1760_std_t new_gyro_packet(void) {
    static uint8_t i = 0;
    dsp1760_std_t pkt = packet_primitive;

    // Add fake gyro data here
    uint32_t amp = 40;
    // x,y,z, ramps over all available values with different phases
    pkt.x_raw = (i + (amp / 4)) % amp;
    pkt.y_raw = (i + (amp / 2)) % amp;
    pkt.z_raw = (i + (3 * amp / 4)) % amp;
    pkt.sequence = i;

    uint32_t crc = crc32((byte*) &pkt, 32, 0x04C11DB7, 0xFFFFFFFF, 0, false, false);

    i = (i + 1) % 127;

    return pkt;
}

void setup() {
    Serial.begin(921600);
}

void loop() {
    dsp1760_std_t pkt = new_gyro_packet();
    
    Serial.write( (byte*) &pkt, sizeof(pkt));

    delay(1);
}
```
{% endraw %}

The packet structure as lifted from the flight code is interesting: a union, such that it can be accessed by fields or as 36 raw bytes, and some fields can be read as floats or raw bytes, because they're unions too!

The most difficult part is creating the [CRC](https://en.wikipedia.org/wiki/Cyclic_redundancy_check), which is an implementation from a CRC-calculating [library](https://github.com/RobTillaart/CRC).

After flashing the code to an [Adafruit Trinket M0](https://www.adafruit.com/product/3500)[^1], plugging it in, and adding the new USB device to the [VirtualBox USB passthrough](https://www.linuxbabe.com/virtualbox/access-usb-from-virtualbox-guest-os),  it was ready to spew at the flight software running in the virtual machine.

It shows up as `/dev/ttyACM0`, so we pretend it is actually a gyro that the flight software is looking for:

`sudo ln -s /dev/ttyACM0 /dev/ttyGYRO0`

Then run the flight software in the debugger:

`sudo gdb --args ./mcp -x`

We see that we attempt to churn through the serial buffer, throwing away bytes until we see the unique gyro header, which is `0xFE81FF55`. In `dsp1760.c:270`:

```c
/// We loop here to catch multiple packets that may have been delivered since we last read
    while (ph_bufq_discard_until(m_serial->rbuf, header, 4)) {
```

where `const char header[4] = { 0xFE, 0x81, 0xFF, 0x55 };` can be interpreted in decimal as `254 129 255 85`. The only problem is, we don't ever seem to see the header!

Dredging deep into the bowels of `ph_bufq_discard_until`, we come to the comparison of the expected packet header:

```
(gdb) x/4ub delim
0x7fffffffd3a4:	254	129	255	85
```

with the raw buffer memory:

```
(gdb) x/4ub bstart
0x55555713ec60:	85	255	129	254
```

Wow! What gives? It looks like the buffer has the header, but it's all backward! What's in the rest of the packet?

```
(gdb) x/36ub bstart
0x55555713ec60:	85	255	129	254	21	0	0	0
0x55555713ec68:	31	0	0	0	1	0	0	0
0x55555713ec70:	42	42	42	42	42	42	42	42
0x55555713ec78:	42	42	42	42	1	11	42	0
0x55555713ec80:	10	11	226	208
```

Breaking it down according to the packet struct above, we should have:

```
header: 85	255	129	254
x_raw: 21	0	0	0
y_raw: 31	0	0	0
z_raw: 1	0	0	0
reserved: 42 42	42	42	42	42	42	42	42	42	42	42
status: 1
sequence: 11 
temperature: 42 0
crc: 10	11	226	208
```

We see that all the struct member boundaries fall in the right places, but the byte order is reversed. The header is backward, the fake raw gyro values are 21, 31, and 1, but have empty trailing bytes, and the 2-byte temperature integer also has an empty trailing byte.

Sure enough, Arduino `Serial.write()` sends the [least significant byte first](https://forum.arduino.cc/t/understanding-byte-order-little-endian-in-arduino-serial-communication/126797/2). 

So how can we fix it so the header is recognized and the rest of the data isn't scrambled?

Well, maybe we could have the poor microcontroller write one byte of the struct at a time with individual calls to `Serial.write()`. That's very uncool for various reasons. We could also employ a version of `htonl()` and `htons()`, and convert these integers to network-long and network-short byte order, because network byte order is big-endian:

```c
#define htons(x) ( ((x)<< 8 & 0xFF00) | \
                   ((x)>> 8 & 0x00FF) )
#define htonl(x) ( ((x)<<24 & 0xFF000000UL) | \
                   ((x)<< 8 & 0x00FF0000UL) | \
                   ((x)>> 8 & 0x0000FF00UL) | \
                   ((x)>>24 & 0x000000FFUL) )
```

Now what do we get when we run our code?

```
(gdb) x/36ub bstart
0x55555713ec60:	254	129	255	85	0	0	0	13
0x55555713ec68:	0	0	0	23	0	0	0	33
0x55555713ec70:	42	42	42	42	42	42	42	42
0x55555713ec78:	42	42	42	42	1	3	0	42
0x55555713ec80:	88	221	0	0
```

Looking much better! The gyro code is now receiving fake packets, and we have no indication of corrupted data, meaning the CRC we attached is valid.

```
-Feb-23-23 03:48:37.094- Schedu: dsp1760.c:309 (dsp1760_process_data):Gyro0: x_raw=0, y_raw=10, z_raw=20, reserved = e331272c, status=1, seq=46, temp=42, crc=ff3b50f6
-Feb-23-23 03:48:37.095- Schedu: dsp1760.c:309 (dsp1760_process_data):Gyro0: x_raw=1, y_raw=11, z_raw=21, reserved = e3312750, status=1, seq=47, temp=42, crc=dc3f6ee8
-Feb-23-23 03:48:37.096- Schedu: dsp1760.c:309 (dsp1760_process_data):Gyro0: x_raw=2, y_raw=12, z_raw=22, reserved = e3312774, status=1, seq=48, temp=42, crc=74f62fd8
-Feb-23-23 03:48:37.097- Schedu: dsp1760.c:309 (dsp1760_process_data):Gyro0: x_raw=3, y_raw=13, z_raw=23, reserved = e3312798, status=1, seq=49, temp=42, crc=57f211c6
...
```

Now, we have a programmable surrogate for the gyroscope.

[^1]: Chosen for its absurdly high clock speed/$ ratio: 48 MHz = 3x Arduino Uno, for <$10!



<br>
<br>
<br>

# Version Control of Large Files with `git-lfs`<a name="2022-07-30"></a>

---

*2022-07-30*

[Git Large File Storage](https://git-lfs.github.com/)

Committing large files (and multiple versions of large files) can balloon the size of a repo on disk even after large files are removed from version control, since the file versions are stored in blobs in the `.git` dir as history. To mitigate this issue, we use `git-lfs` with GitHub.

The basic premise is to work around `git` by storing pointers to remotely hosted versions of the files, which are automatically replaced with the real versions by `git-lfs` on checkout.

With `git-lfs`, you can commit and pull/push large files as normal.

## Q&A

### What if I don't have `git-lfs` installed?

You will check out a small file with the information needed by `git-lfs` to retrieve the file instead of the actual file.

To install `git-lfs`, instructions [here](https://docs.github.com/en/repositories/working-with-files/managing-large-files/installing-git-large-file-storage).

> Note: Debian 10, Ubuntu 18.04 and newer can just do `sudo apt-get install git-lfs && git lfs install`. Mac OSX can just do `brew install git-lfs && git lfs install` if `brew` is up to date.


<details>
<summary>
For posterity:
</summary>

1. Navigate to git-lfs.github.com and click Download.

> Tip: For more information about alternative ways to install Git LFS for Linux, see this Getting started guide.

2. On your computer, locate and unzip the downloaded file.

3. Open Terminal.

4. Change the current working directory into the folder you downloaded and unzipped: `cd ~/Downloads/git-lfs-1.X.X`

> Note: The file path you use after cd depends on your operating system, Git LFS version you downloaded, and where you saved the Git LFS download.

5. To install the file, run this command:

```
./install.sh`
Git LFS initialized.
```

> Note: You may have to use `sudo ./install.sh` to install the file.

6. Verify that the installation was successful:

```
git lfs install
Git LFS initialized.
```

</details>


You only need to do this once per machine.

### What file types are managed by `git-lfs` already?

Run `git lfs track` at the root of the repo for the most up-to-date list.

### I have a new file type I want to commit and it is big (>~few MB). What do I do?

If you haven't already, don't `git add` the new file. Make `git-lfs` track the file name/type first, then `git add` it. 

> Note: GitHub recommends that you should always run these commands from the root of the repository to ensure all files of the type are caught, no matter which directory they are in. Since we already have a ton of pdfs and images squirreled away everywhere, I recommend a more targeted approach, making and versioning a `.gitattributes` file for each subdirectory. A `.gitattributes` file at the root will catch all sorts of stuff in `external_libs`, etc. You can see this at work in `docs/`.

[More docs](https://docs.github.com/en/repositories/working-with-files/managing-large-files/configuring-git-large-file-storage).

<details>
<summary>
For posterity:
</summary>

If there are existing files in your repository that you'd like to use GitHub with, you need to first remove them from the repository and then add them to Git LFS locally. For more information, see "[Moving a file in your repository to Git LFS](https://docs.github.com/en/articles/moving-a-file-in-your-repository-to-git-large-file-storage)."

If there are referenced Git LFS files that did not upload successfully, you will receive an error message. For more information, see "[Resolving Git Large File Storage upload failures](https://docs.github.com/en/articles/resolving-git-large-file-storage-upload-failures)."

1. Open Terminal.

2. Change your current working directory to an existing repository you'd like to use with Git LFS.

3. To associate a file type in your repository with Git LFS, enter `git lfs track` followed by the name of the file extension you want to automatically upload to Git LFS.

For example, to associate a `.psd` file, enter the following command:

```
git lfs track "*.psd"`
Adding path *.psd
```

Every file type you want to associate with Git LFS will need to be added with `git lfs track`. This command amends your repository's `.gitattributes` file and associates large files with Git LFS.

> Note: We strongly suggest that you commit your local `.gitattributes` file into your repository. Relying on a global `.gitattributes` file associated with Git LFS may cause conflicts when contributing to other Git projects. Including the `.gitattributes` file in the repository allows people creating forks or fresh clones to more easily collaborate using Git LFS. Including the `.gitattributes` file in the repository allows Git LFS objects to optionally be included in ZIP file and tarball archives.

4. Add a file to the repository matching the extension you've associated:

```
git add path/to/file.psd
```

Commit the file and push it to GitHub:

```
git commit -m "add file.psd"
git push
```

You should see some diagnostic information about your file upload:

```
Sending file.psd
44.74 MB / 81.04 MB  55.21 % 14s
64.74 MB / 81.04 MB  79.21 % 3s
```

</details>

### I want to convert an already-committed file to `git-lfs` tracking. What can I do?

[More docs](https://docs.github.com/en/repositories/working-with-files/managing-large-files/moving-a-file-in-your-repository-to-git-large-file-storage).

<details>
<summary>
For posterity:
</summary>

If you've set up Git LFS, and you have an existing file in your repository that needs to be tracked in Git LFS, you need to first remove it from your repository.

After installing Git LFS and configuring Git LFS tracking, you can move files from Git's regular tracking to Git LFS. For more information, see "[Installing Git Large File Storage](https://docs.github.com/en/github/managing-large-files/installing-git-large-file-storage)" and "[Configuring Git Large File Storage](https://docs.github.com/en/github/managing-large-files/configuring-git-large-file-storage)."

If there are referenced Git LFS files that did not upload successfully, you will receive an error message. For more information, see "[Resolving Git Large File Storage upload failures](https://docs.github.com/en/articles/resolving-git-large-file-storage-upload-failures)."

> Tip: If you get an error that "this exceeds Git LFS's file size limit of 100 MB" when you try to push files to Git, you can use `git lfs migrate` instead of `filter branch` or the BFG Repo Cleaner, to move the large file to Git Large File Storage. For more information about the `git lfs migrate` command, see the [Git LFS 2.2.0](https://github.com/blog/2384-git-lfs-2-2-0-released) release announcement.

1. Remove the file from the repository's Git history using either the `filter-branch` command or BFG Repo-Cleaner. For detailed information on using these, see "[Removing sensitive data from a repository](https://docs.github.com/en/articles/removing-sensitive-data-from-a-repository)."
2. Configure tracking for your file and push it to Git LFS. For more information on this procedure, see "[Configuring Git Large File Storage](https://docs.github.com/en/articles/configuring-git-large-file-storage)."

</details>


### Are there limitations?

GitHub imposes as 2 GB per file size limit.

Unless a local `git-lfs` host has been set up, this will not work on the ice, since files must be pulled from/pushed to an Internet server. In this case, it's probably best to check out a repo to download all docs to local machines before leaving fast Internet access zones.

<br>
<br>
<br>

# How to write unit tests <a name="2022-07-19"></a>

---

*2022-07-19*

"If it isn't tested, it doesn't work." - [Bob Colwell](https://courses.cs.washington.edu/courses/cse403/06wi/misc/colwell-testing.pdf)

## Why To Write Unit Tests

Testing is essential, testing is hopeless. If you are changing code that works, fixing code that doesn't, or writing new code, you need a way to tell when it isn't broken. Computers and the demands placed on them by software applications are now so complicated that any hope of testing them to saturation is lost. You cannot wait long enough for every possible input to each permutation of your code execution path to be analyzed. Without unit testing, you will not be able to prove that a piece of code does what it is intended to do, unless you exercise the entire system under realistic conditions - the malady a developer named Lindsay Page once referred to as "driving the car to see if the radio works." So if you need to prove that a piece of code works, and you don't have infinite time and money to devote to doing that, you need unit tests. You also need to write unit tests to crystallize something essential which is sometimes separate from the code - the _developer's intent_.

It's your job as the software engineer to exercise judgment, dicing the code into manageable units and distilling as many of the most important cases you can think of into a set of inputs and the expected set of outputs from those units. 

## What You Need

1. Expected inputs
2. Expected outputs
3. Some code
4. A way to run 3. and provide 1. to check the results against 2.

If you don't have 1. or 2., you are working on a piece of software with no requirements. If you are working on generative art, this is OK. If you are not, you need to show stakeholders (people with money or risk involved in your project) that your software does what they want, and doesn't do what they don't want. 

Having specific requirements that change on timescales longer than your workday is essential to writing tests, and it may be a significant amount of work to get them from stakeholders so you can begin writing tests.

If you don't have 3., don't get any, otherwise you will have to write unit tests.

If you don't have 4., you are in the right document. 

## How To Do It

The process of unit testing is: isolate a chunk of code as much as possible, write code to compare the output to an expected value, run it, do this many times, and collect the results.

To isolate the unit under test (UUT), we will write code that `#include`s the code to test, or links to a compiled library. As necessary, we will lift dependencies by introducing stub functions for them or overwriting them with our own, simpler ones.

To write the test code, we will leverage a unit testing framework that provides macros to make checking for agreement between returned values and expected values quick and easy.

To run it, we will add the test code to our build system, and make sure the test executable runs as a part of the build process.

To collect the results, we will again leverage the unit testing framework to generate a test report.

## When To Do It

Entire books have been written about this topic, so I will try to simplify by suggesting when more tests should be written for the your big ugly legacy project, given that it is ostensibly already-working, field-tested code. Sometimes, the most cost-effective thing to do is nothing if the code works, but if any of these are true:

* You are about to write a new module
* You are about to troubleshoot an old module
* Commits **repeatedly** result in regressions
* Flight code exhibits segmentation faults, race conditions, or memory leaks and **workarounds** are observed instead of root cause fixes
* More than 60% of dev time is spent debugging (heuristic, your mileage may vary)

then time needs be allocated to developing unit tests in the affected modules.

## Choosing a Unit Test Framework

There are [many more ways](https://en.wikipedia.org/wiki/List_of_unit_testing_frameworks) to write unit tests than there are programming languages, so rationales for choosing one test framework over the other like personal preference, organizational inertia, and random chance are all good ones. The most important aspects of your chosen test framework to focus on are, in order:

1. Having one
2. Speed: testing overhead must be small, so developers will want to run them early and often as part of their development routine
3. Extensible: it must be easy to add new tests, so developers will want to add new tests for their new features, and maintainers will want to add tests for legacy untested features

The goal of this document is to describe a method for testing C/C++ code that accomplishes 1., and arguments can be made that it accomplishes 2. and 3. For C/C++ code, some of the [most popular](https://www.jetbrains.com/lp/devecosystem-2021/cpp/) frameworks are Google Test, Catch, CppUnit, Boost.Test, and nothing. It would be nice to use Google Test for the TIM project because it's well documented and the chances are high that we will see it again if we continue to work on C/C++ projects. However, it introduces a C++11 dependency, and the cleanest and easiest way to install it and maintain it on our target system is a feature added in a more recent version of CMake (3.11) than we have access to (<= 3.0.5).

There are other test frameworks geared more toward C, and for this example we will use a lightweight one (2 headers, 1 source file) called [Unity](http://www.throwtheswitch.org/unity). This is advantageous for our project because it allows the flight software to continue to be self contained, and should not rely on fetching any code from the internet at build-time. Unity also has [more advanced companion tools](http://www.throwtheswitch.org/#download-section) that may become useful if our testing needs grow. 

Finally, learning any unit test framework is like learning a language, in that it will have benefits even if you go on to code in something other than C.

## Unity

Unity is made by [ThrowTheSwitch.org](http://www.throwtheswitch.org/).

There is a useful [Getting Started](https://github.com/ThrowTheSwitch/Unity/blob/master/docs/UnityGettingStartedGuide.md) guide.

A quick and dirty way to make the Unity source available could be embedding itin the repo so it can be found by more than one executable.

There is a tradeoff here: embedding a copy of the Unity source makes it more difficult to accept updates to it from upstream, but it makes writing CMakeLists.txt files easier. In this example, I set aside my usual preference to have dependencies managed separate from flight code because I think writing CMake will happen more often than pulling Unity updates.

### The Simple Example

An example of Unity in action might look like this:

```
evanmayer@framework:~/unittest_example$ tree -L 2
.
├── CMakeLists.txt
├── external
│   ├── CMakeLists.txt
│   └── Unity-2.5.2
└── src
    ├── add_example.c
    ├── add_example.h
    ├── CMakeLists.txt
    └── test_add.c
```

It was produced by following a tutorial by [Rainer Poisel](https://www.poisel.info/posts/2019-07-15-cmake-unity-integration/). 

The toplevel CMakeLists.txt points at the lower ones:

* `src/CMakeLists.txt` builds the code under test and builds the test code into an executable by linking to the code under test and Unity, then adds it to a list of tests registered with CTest
* `external/CMakeLists.txt` builds Unity into a public static library for the test code to link to

Here is what the test code in `test_add.c` looks like:

```c
#include <unity.h>
#include "add_example.h"

void setUp(void) {
    // set stuff up here, if needed
}

void tearDown(void) {
    // clean stuff up here, if needed
}

void test_AddOnePlusOne(void)
{
    int a = 1;
    int b = 1;
    TEST_ASSERT_EQUAL(2, add_example(a, b));
}

int main(void) {
    UNITY_BEGIN();
    RUN_TEST(test_AddOnePlusOne);
    // append additional tests here
    return UNITY_END();
}
```

To build and run the tests:

```bash
cmake -S . -B build
cmake --build build
cd build/ && ctest && cd ..
```

Which should yield:

```
Test project /home/evanmayer/unittest_example/build
    Start 1: add_example_test
1/1 Test #1: add_example_test .................   Passed    0.00 sec

100% tests passed, 0 tests failed out of 1

Total Test time (real) =   0.00 sec
```

As you can see, unit testing is compatible with an existing CMake build system, the test code can be made short and easy to write with Unity's macros, and running the tests with `ctest` generates a test report. All that's left is to apply this strategy to your code.

<br>
<br>
<br>

# Is my application "real-time"? <a name="2022-07-14"></a>

---

*2022-07-14*

## What is "real-time"?

Software for aerospace and industrial systems has different requirements than most software people regularly interact with. It may have stringent deadlines to meet when interacting with the real world or other systems which, if not met, result in behavior that could endanger the mission goals, the system itself, or things around the system (such as people or property).

For example: an electronic coffee machine heats water from a starting temperature to one set by the user. To heat the coffee, it energizes a heating element. To determine the present temperature, it reads a thermometer. In order to stop heating the water at the appropriate time to avoid overheating and boiling off all the water, it must measure the temperature at a given rate, perform a calculation (like the difference between the present temperature and the set point), determine whether the heater should be on or off, and command the heater if necessary. What happens if any one of these steps is delayed? If the temperature probe does not return a valid measurement for several seconds for some reason, and the rest of the system does not have any means to compensate for missing measurements, the temperature may miss the set point, resulting in poor temperature control or excess water boil-off.

The consequences of any mis-timing in even this simple system depend on the frequency and magnitude of the mis-timing event. If a temperature query or heater on/off command deadline is intended to happen every 100 Hz, but is delayed one time by a dozen milliseconds in a 3-minute run of the coffee machine, the end user will not notice and the coffee quality or boiler safety will not be impacted. However, if it happens more often or with greater severity, the chance of failure goes up! 

This simple illustration introduces a few principles of real-time systems that impact their design and testing:
   
* The system needs to be fast enough to minimize the severity of exceptional circumstances where a deadline is not met
* It also needs to be stable enough to minimize the number of these occurrences

In my opinion, there is no black and white "real-time," only "fast enough" and "stable enough" for the application's requirements! The coffee machine controller may be fine to heat a boiler of hot water, but risks failing spectacularly when it is required to control cooling of a pulsed laser. 

[Linux RT FAQ: What is real-time?](https://rt.wiki.kernel.org/index.php/Frequently_Asked_Questions#What_is_real-time.3F)

### "Hard" and "soft" real-time

You may also read authors that categorize systems as "hard" and "soft" real-time by the system's tolerance of missing a deadline. Some refer to "hard real-time" systems as those that exhibit catastrophic failure if a deadline is missed even once, and "soft real-time" systems as those that experience degraded performance, but not failure, under multiple missed deadlines. Other definitions exist. For example, some may differentiate between the two by their method of scheduling tasks.

## Linux real time

Running software for a complex vehicle on a PC with a Linux operating system turns the problems illustrated above up to eleven. There are many sensors and actuators to interact with the outside world, and the operating system itself has various tasks to run, which may not be on a predictable cadence. The resource needs of the system fluctuate. In order to deal with this fluctuation and the reality that there may sometimes be insufficient resources (CPU cycles vs. time, memory) to perform the necessary tasks, we must carefully set up our system.

If the system is not able to completely eliminate all events that would be severe enough to cause a failure, it must have safeguards in place to deal with them, such as giving up on a blocking process and re-using an older sensor value, or predicting a new one from past data.

The most important point is that ALL of the measures outlined below work together to limit the frequency and severity of events that "break real time."

> NOTE: This is a good (but evolving) resource on it: [https://rt.wiki.kernel.org/index.php/Main_Page](https://rt.wiki.kernel.org/index.php/Main_Page)

### Kernel options

The Linux kernel is a [low-level program](https://en.wikipedia.org/wiki/Kernel_(operating_system)) that controls important tasks in the operating system. Generally, these tasks cannot be stopped ("preempted") in order to execute a higher-priority task that comes along, except in [certain circumstances](https://rt.wiki.kernel.org/index.php/Frequently_Asked_Questions#What_are_real-time_capabilities_of_the_stock_2.6_linux_kernel.3F). This can lead to occasional latency between when a user-space program requests a resource to when it becomes available, on the order of milliseconds to seconds. The kernel code can be compiled with an option, `CONFIG_PREEMPT`, that expands the coverage of code that can be preempted by higher-priority tasks, making typical worst-case latency outside of some device driver code single digit milliseconds, and making latency much more predictable overall.

### The RT kernel patch

A [`CONFIG_PREEMPT_RT` patch](https://rt.wiki.kernel.org/index.php/Frequently_Asked_Questions#How_does_the_CONFIG_PREEMPT_RT_patch_work.3F) is available that further modifies kernel code to allow more chunks of code to be preempted, further decreasing the worst-case latency. 

The kernel code must be [downloaded, compiled, and the current kernel must be patched](https://chenna.me/blog/2020/02/23/how-to-setup-preempt-rt-on-ubuntu-18-04/).

There are [various options](https://lwn.net/Articles/146861/) that can change the behavior of the kernel to make it more suitable for real-time operation.

Or, you may be able to find a prepackaged kernel version to install with `apt`:

`sudo apt search linux-image`

`sudo apt install linux-image-rt-amd64`

### The application

**It's not enough to simply patch the kernel.** The application must be written and executed with specific attention paid to how resources are requested and used by the program.

I think this [how to build a RT application guide](https://rt.wiki.kernel.org/index.php/HOWTO:_Build_an_RT-application) is mandatory reading, even if you don't fully understand every word. A quick summary of the categories of gotchas to pay attention to:

* CPU stuff
    * Power management settings - OS may slow down or vary CPU clock speed. A slower CPU speed obviously changes performance, but *changing* the clock speed also has an overhead. Also goes for other CPU power states, e.g. putting CPUs to sleep and waking them up
    * CPU allocation - The application should be bound to a specific CPU or set of CPUs, to avoid overhead involved in changing between them
* Application priority
    * An elevated priority must be assigned to the application in order to allow it to preempt other things. Real-time processes have access to priorities from 1-99, typically something in the 80s is selected. Avoid 99, because it should be reserved for critical processes such as a watchdog to shut down infinite looping real-time processes.
* Memory usage
    * Page faults - roughly, attempting to access memory that has been marked for cleanup, any may not be in physical memory. Can trigger an action like loading from disk to RAM. Extremely difficult to predict, so random source of non-determinism. Avoid it by profiling an application's (or thread's) memory usage, and [locking down](https://www.oreilly.com/library/view/embedded-linux-for/9781787124202/ch32s10.html) enough memory as soon as you can in your program - call dibs. You can't avoid page faults at startup when you lock everything down, but you can start up the program and incur all your page faults before your system is put into a situation where it NEEDS to be real-time compliant.
        * This means you must program in a way many developers are unfamiliar with - avoiding dynamic memory allocation at all costs to avoid costly page faults means preallocating as much memory as you will EVER need for a task during an `initialize()` function.
* Loops
    * For strict determinism, all loops in software should execute a consistent number of times - loops with variable end conditions can add jitter
* Input/output
    * All I/O access will incur a penalty due to the laws of physics - HDD arms must move, SSD capacitors must charge/discharge, etc. They should be sequestered to their own thread/CPU to offload I/O from the RT parts of the code.

## More info:

* [https://lwn.net/Articles/146861/](https://lwn.net/Articles/146861/)
* [https://rt.wiki.kernel.org/index.php/Main_Page](https://rt.wiki.kernel.org/index.php/Main_Page)
* [https://elinux.org/images/d/de/Real_Time_Linux_Scheduling_Performance_Comparison.pdf](https://elinux.org/images/d/de/Real_Time_Linux_Scheduling_Performance_Comparison.pdf)
* [https://rt-labs.com/docs/p-net/linuxtiming.html](https://rt-labs.com/docs/p-net/linuxtiming.html)
* [https://chenna.me/blog/2020/02/23/how-to-setup-preempt-rt-on-ubuntu-18-04/](https://chenna.me/blog/2020/02/23/how-to-setup-preempt-rt-on-ubuntu-18-04/)
* [https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/cyclictest/start](https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/cyclictest/start)
* [http://www.cs.ru.nl/J.Hooman/DES/XenomaiExercises/Background.html](http://www.cs.ru.nl/J.Hooman/DES/XenomaiExercises/Background.html)
* [https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.55.2137&rep=rep1&type=pdf](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.55.2137&rep=rep1&type=pdf)