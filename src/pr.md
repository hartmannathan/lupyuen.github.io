# Preparing a Pull Request for Apache NuttX RTOS

📝 _28 Nov 2022_

![Typical Pull Request for Apache NuttX RTOS](https://lupyuen.github.io/images/pr-title.jpg)

[__(Watch the Video on YouTube)__](https://youtu.be/p6ly3EBhtpo?si=ArHLgnu5JWLb5FtW)

This article explains how I prepared my Pull Requests for submission to [__Apache NuttX RTOS__](https://nuttx.apache.org/docs/latest/index.html). So if we're contributing code to NuttX, just follow these steps and things will (probably) go Hunky Dory!

(Like the fish)

Before we begin, please swim over to the official __Development Workflow__ for NuttX...

-   [__"NuttX Development Workflow"__](https://nuttx.apache.org/docs/latest/contributing/workflow.html)

Ready? Let's dive in! (Like the fish)

![Create Fork](https://lupyuen.github.io/images/pr-fork.png)

# NuttX Repositories

We begin by __creating our forks__ for the __`nuttx`__ and __`apps`__ repositories...

1.  Browse to __NuttX Repository__...

    [__github.com/apache/nuttx__](https://github.com/apache/nuttx)

    Click "__Fork__" to create our fork. (Pic above)

    Click "__Actions__" and enable workflows...

    ![Enable Workflows](https://lupyuen.github.io/images/pr-actions.png)

    (This will check that our code compiles OK at every commit)

1.  Do the same for the __NuttX Apps Repository__...

    [__github.com/apache/nuttx-apps__](https://github.com/apache/nuttx-apps)

    Click "__Fork__" to create our fork.

    Click "__Actions__" and enable workflows.

1.  As a principle, let's keep our __`master`__ branch __always in sync__ with the NuttX Mainline __`master`__ branch.

    (This seems cleaner for [__syncing upstream updates__](https://lupyuen.github.io/articles/pr#update-our-repositories) into our repo)

    Let's __create a branch__ to make our changes...

1.  In our NuttX Repository, click __`master`__.

    Enter the name of our new branch.
    
    Click "__Create Branch__"

    ![Create Branch](https://lupyuen.github.io/images/pr-branch.png)

    [(I named my branch __`gic`__ for Generic Interrupt Controller)](https://github.com/lupyuen2/wip-nuttx/tree/gic)

1.  Do the same for our __NuttX Apps Repository__

    (Because we should sync __`nuttx`__ and __`apps`__ too)

1.  Download the new branches of our __`nuttx`__ and __`apps`__ repositories...

    ```bash
    ## Download the "gic" branch of "lupyuen2/wip-nuttx"
    ## TODO: Change the branch name and repo URLs
    mkdir nuttx
    cd nuttx
    git clone \
      --branch gic \
      https://github.com/lupyuen2/wip-nuttx \
      nuttx
    git clone \
      --branch gic \
      https://github.com/lupyuen2/wip-nuttx-apps \
      apps
    ```

# Build and Test

We're ready to code!

1.  Consider breaking our Pull Request into __Smaller Pull Requests__.

    This Pull Request implements __One Single Feature__ (Generic Interrupt Controller)...

    [__"arch/arm64: Add support for Generic Interrupt Controller Version 2"__](https://github.com/apache/nuttx/pull/7630)

    That's called by another Pull Request...

    [__"arch/arm64: Add support for PINE64 PinePhone"__](https://github.com/apache/nuttx/pull/7692)

    Adding a NuttX Arch (SoC) and Board might need 3 Pull Requests (or more)...

    [__"Add the NuttX Arch and Board"__](https://lupyuen.github.io/articles/release#add-the-nuttx-arch-and-board)

1.  __Modify the code__ in __`nuttx`__ and __`apps`__ to implement our awesome new feature.

    [(Be careful with __CMake and CMakeLists.txt__)](https://lupyuen.github.io/articles/pr#appendix-build-nuttx-with-cmake)

    [(Also watch out for __NuttX Config defconfig__)](https://lupyuen.github.io/articles/pr#appendix-nuttx-configuration-files)

1.  __Build and test__ the modified code.

    [(I configured `F1` in VSCode to run this __Build Script__)](https://gist.github.com/lupyuen/5e2fba642a33bf64d3378df3795042d7)

1.  Capture the __Output Log__ and save it as a [__GitHub Gist__](https://gist.github.com/lupyuen/7537da777d728a22ab379b1ef234a2d1)...

    [__"NuttX QEMU Log for Generic Interrupt Controller"__](https://gist.github.com/lupyuen/7537da777d728a22ab379b1ef234a2d1)

    We'll add this to our Pull Request. __Test Logs are super helpful__ for NuttX Maintainers!

    (Because we can't tell which way the train went... Unless we have the Test Logs!)

1.  __Commit the modified code__ to our repositories.

    Sometimes the __GitHub Actions Workflow__ will fail with a strange error (like below). Just re-run the failed jobs. [(Like this)](https://lupyuen.github.io/images/pr-rerun.png)

    ```text
    Error response from daemon:
    login attempt to https://ghcr.io/v2/
    failed with status: 503 Service Unavailable"
    ```

## Regression Test

_Will our modified code break other parts of NuttX?_

That's why it's good to run a [__Regression Test__](https://en.wikipedia.org/wiki/Regression_testing) (if feasible) to be sure that other parts of NuttX aren't affected by our modified code.

For our Pull Request...

-   [__"arch/arm64: Add support for Generic Interrupt Controller Version 2"__](https://github.com/apache/nuttx/pull/7630)

We tested with [__QEMU Emulator__](https://www.qemu.org/docs/master/system/target-arm.html) our __new implementation__ of Generic Interrupt Controller v2...

```bash
tools/configure.sh qemu-armv8a:nsh_gicv2 ; make ; qemu-system-aarch64 ...
```

[(See the NuttX QEMU Log)](https://gist.github.com/lupyuen/7537da777d728a22ab379b1ef234a2d1)

And for Regression Testing we tested the __existing implementation__ of Generic Interrupt Controller v3...

```bash
tools/configure.sh qemu-armv8a:nsh ; make ; qemu-system-aarch64 ...
```

[(See the NuttX QEMU Log)](https://gist.github.com/lupyuen/dec66bc348092a998772b32993e5ed65)

Remember to capture the __Output Log__, we'll add it to our Pull Request.

(Yeah it will be hard to run a Regression Test if it requires hardware that we don't have)

## Documentation

Please update the __Documentation__. The Documentation might be in a __Text File__...

-   [__nuttx/.../README.txt__](https://github.com/apache/nuttx/blob/master/boards/arm64/qemu/qemu-armv8a/README.txt)

Or in the __Official NuttX Docs__...

-   [__nuttx/Documentation/.../index.rst__](https://github.com/apache/nuttx/blob/master/Documentation/platforms/arm/a64/boards/pinephone/index.rst)

To update the Official NuttX Docs, follow the instructions here...

-   [__"NuttX Documentation"__](https://nuttx.apache.org/docs/latest/contributing/documentation.html)

    [(See the Log)](https://gist.github.com/lupyuen/c061ac688f430ef11a1c60e0b284a1fe)

__For macOS:__ We may need to use "__brew install pipenv__" instead of "__pip install pipenv__". And we may need "__pip install setuptools__" to fix _"No module named pkg_resources"_...

```bash
## Build NuttX Docs on macOS
cd nuttx/Documentation
brew install pipenv
pipenv install
pipenv shell
pip install setuptools
rm -r _build
make html
```

[(See the Log)](https://gist.github.com/lupyuen/eeae419776fba2b59502fcce05bb1859)

![Check Coding Style with nxstyle](https://lupyuen.github.io/images/pr-nxstyle.jpg)

# Check Coding Style

Our NuttX Code will follow this Coding Standard...

-   [__"NuttX C Coding Standard"__](https://nuttx.apache.org/docs/latest/contributing/coding_style.html)

NuttX provides a tool __`nxstyle`__ that will check the Coding Style of our source files...

```bash
## Compile nxstyle
## TODO: Change "$HOME/nuttx" to our NuttX Project Folder
gcc -o $HOME/nxstyle $HOME/nuttx/nuttx/tools/nxstyle.c

## Check coding style for our modified source files
## TODO: Change the file paths
$HOME/nxstyle $HOME/nuttx/nuttx/arch/arm64/Kconfig
$HOME/nxstyle $HOME/nuttx/nuttx/arch/arm64/src/common/Make.defs
$HOME/nxstyle $HOME/nuttx/nuttx/arch/arm64/src/common/arm64_gic.h
$HOME/nxstyle $HOME/nuttx/nuttx/arch/arm64/src/common/arm64_gicv2.c
```

[(`nxstyle.c` is here)](https://github.com/apache/nuttx/blob/master/tools/nxstyle.c)

[(How I run `nxstyle` in my __Build Script__)](https://gist.github.com/lupyuen/5e2fba642a33bf64d3378df3795042d7)

_Will `nxstyle` check Kconfig and Makefiles?_

Not yet, but maybe someday? That's why we passed all the modified files to `nxstyle` for checking.

_How do we fix our code?_

The pic above shows the output from __`nxstyle`__. We'll see messages like...

```text
- C++ style comment
- Long line found
- Mixed case identifier found
- Operator/assignment must be followed/preceded with whitespace
- Upper case hex constant found
- #include outside of 'Included Files' section
```

We modify our code so that this...

```c
// Initialize GIC. Called by CPU0 only.
int arm64_gic_initialize(void) {
  // Verify that GIC Version is 2
  int err = gic_validate_dist_version();
  if (err) { sinfo("no distributor detected, giving up ret=%d\n", err); return err; }

  // CPU0-specific initialization for GIC
  arm_gic0_initialize();

  // CPU-generic initialization for GIC
  arm_gic_initialize();
  return 0;
}
```

[(Source)](https://github.com/lupyuen/nuttx/blob/pinephone/arch/arm64/src/common/arm64_gicv3.c#L717-L743)

Becomes this...

```c
/****************************************************************************
 * Name: arm64_gic_initialize
 *
 * Description:
 *   Initialize GIC. Called by CPU0 only.
 *
 * Input Parameters
 *   None
 *
 * Returned Value:
 *   Zero (OK) on success; a negated errno value is returned on any failure.
 *
 ****************************************************************************/

int arm64_gic_initialize(void)
{
  int err;

  /* Verify that GIC Version is 2 */

  err = gic_validate_dist_version();
  if (err)
    {
      sinfo("no distributor detected, giving up ret=%d\n", err);
      return err;
    }

  /* CPU0-specific initialization for GIC */

  arm_gic0_initialize();

  /* CPU-generic initialization for GIC */

  arm_gic_initialize();

  return 0;
}
```

[(Source)](https://github.com/apache/nuttx/blob/master/arch/arm64/src/common/arm64_gicv2.c#L1325-L1363)

If we see this...

```text
/* */ not balanced
```

Check that our stars "__`*`__" are aligned (heh)...

```text
/******************************
 * Name: missing_asterisk_below
 *
 *****************************/
```

After fixing, __test our code__ one last time.

# Write the Pull Request

Our Pull Request will have...

1.  __Title__: NuttX Subsystem and One-Line Summary

1.  __Summary__: What's the purpose of the Pull Request? What files are changed?

1.  __Impact__: Which parts of NuttX are affected by the Pull Request? Which parts _won't_ be affected?

1.  __Testing__: Provide evidence that our Pull Request does what it's supposed to do.

    (And that it won't do what it's _not supposed_ to do)

To write the above items, let's walk through these Pull Requests...

-   [__"arch/arm64: Add support for Generic Interrupt Controller Version 2"__](https://github.com/apache/nuttx/pull/7630)

-   [__"arch/arm64: Add support for PINE64 PinePhone"__](https://github.com/apache/nuttx/pull/7692)

## Title

Inside our Title is the __NuttX Subsystem__ and __One-Line Summary__...

>   _"arch/arm64: Add support for Generic Interrupt Controller Version 2"_

>   [(Source)](https://github.com/apache/nuttx/pull/7630)

Let's write this into a __Markdown File__ for easier copying...

-   [__Sample Pull Request__](https://gist.github.com/lupyuen/4dbe011143dfc5404e1791ba74a79deb)

    [(See the Markdown Code)](https://gist.githubusercontent.com/lupyuen/4dbe011143dfc5404e1791ba74a79deb/raw/b8731c9132c428050e6146c0d2097e7bb7c6eb03/sample-nuttx-pull-request.md)

We'll add the following sections to the Markdown File...

## Summary

In the Summary we write the __purpose of the Pull Request__...

>   _"This PR adds support for GIC Version 2, which is needed by Pine64 PinePhone."_

>   [(Source)](https://github.com/apache/nuttx/pull/7630)

And we list the __files that we changed__...

>   _"`arch/arm64/Kconfig`: Under "ARM64 Options", we added an integer option ARM_GIC_VERSION ("GIC version") that selects the GIC Version. Valid values are 2, 3 and 4, default is 3."_

>   [(Source)](https://github.com/apache/nuttx/pull/7630)

If it's a long list, we might __break into subsections__ like this...

-   [__arch/arm64: Add support for PINE64 PinePhone__](https://github.com/apache/nuttx/pull/7692)

For __Code Provenance__ it's good to state __how we created the code__...

>   _"This 64-bit implementation of GIC v2 is mostly identical to the existing GIC v2 for 32-bit Armv7-A ([`arm_gicv2.c`](https://github.com/apache/incubator-nuttx/blob/master/arch/arm/src/armv7-a/arm_gicv2.c), [`gic.h`](https://github.com/apache/incubator-nuttx/blob/master/arch/arm/src/armv7-a/gic.h)), with minor modifications to support 64-bit Registers (Interrupt Context)."_

>   [(Source)](https://github.com/apache/nuttx/pull/7630)

(Adding __GPL Code__? Please check with the NuttX Maintainers)

## Impact

Under "Impact", we write __which parts of NuttX are affected__ by the Pull Request. 

(And which parts _won't_ be affected)

>   _"With this PR, NuttX now supports GIC v2 on Arm64._

>   _There is no impact on the existing implementation of GIC v3, as tested below."_

>   [(Source)](https://github.com/apache/nuttx/pull/7630)

## Testing

Under "Testing", we provide evidence that our Pull Request __does what it's supposed to do.__ We fill in the...

- __Commands__ used for testing

- __Output Logs__ captured from our testing

Like this...

>   _"We tested with QEMU our implementation of GIC v2:_

>   _`tools/configure.sh qemu-armv8a:nsh_gicv2 ; make ; qemu-system-aarch64 ...`_

>   _[(See the NuttX QEMU Log)](https://gist.github.com/lupyuen/7537da777d728a22ab379b1ef234a2d1)_

>   _The log shows that GIC v2 has correctly handled interrupts"_

>   [(Source)](https://github.com/apache/nuttx/pull/7630)

If we have done a [__Regression Test__](https://lupyuen.github.io/articles/pr#regression-test), provide the details too...

>   _"For Regression Testing: We tested the existing implementation of GIC v3..."_

>   _`tools/configure.sh qemu-armv8a:nsh ; make ; qemu-system-aarch64 ...`_

>   _[(See the NuttX QEMU Log)](https://gist.github.com/lupyuen/dec66bc348092a998772b32993e5ed65)_

>   [(Source)](https://github.com/apache/nuttx/pull/7630)

__Test Logs are super helpful__ for NuttX Maintainers!

(Because we can't tell which way the train went... By staring at the track!)

Now we tidy up our commits...

![Squash Commits with GitHub Desktop](https://lupyuen.github.io/images/pr-squash1.png)

# Squash the Commits

_What's this "squashing"?_

"Squashing" means we're __combining Multiple Commits__ into One Single Commit.

Our __Commit History__ can get awfully messy during development...

```text
- Initial Commit
- Fixing Build
- Build OK!
- Oops fixing bug
- Tested OK yay!
```

[(Source)](https://github.com/lupyuen/nuttx/commits/pinephone/arch/arm64/src/qemu/qemu_serial.c)

So we always __Squash the Commits__ into One Single Commit (to help future maintainers)...

```text
- arch/arm64: Add support for Generic Interrupt Controller Version 2
```

[(Source)](https://github.com/apache/nuttx/pull/7630/commits)

_How do we squash the commits?_

We'll use [__GitHub Desktop__](https://desktop.github.com/) (because I'm terrible with the Git Command Line)...

1.  Install [__GitHub Desktop__](https://desktop.github.com/) and launch it

1.  Click "__File → Add Local Repository__"

    Select our downloaded __`nuttx`__ folder.

    Click "__Add Repository__"

1.  Click the "__History__" tab to reveal the Commit History

    [(Pic above)](https://lupyuen.github.io/images/pr-squash1.png)

1.  Select the Commits to be Squashed.

    Right-click the Commits.

    Select "__Squash Commits__"

    [(Pic above)](https://lupyuen.github.io/images/pr-squash1.png)

1.  Copy the __Title__ of our Pull Request and paste into the __Title Box__...

    ```text
    arch/arm64: Add support for Generic Interrupt Controller Version 2
    ```

    In the __Description Box__, erase the old Commit Messages.

    Copy the __Summary__ of our Pull Request and paste into the __Description Box__...

    ```text
    This PR adds support for GIC Version 2.
    - `boards/arm64/qemu/qemu-armv8a/configs/nsh_gicv2/defconfig`: Added the Board Configuration
    ```

    Click "__Squash Commits__"

    ![Squash Commits with GitHub Desktop](https://lupyuen.github.io/images/pr-squash2.png)

1.  Click "__Begin Squash__"

    ![Squash Commits with GitHub Desktop](https://lupyuen.github.io/images/pr-squash3.png)

1.  Click "__Force Push Origin__"

    ![Squash Commits with GitHub Desktop](https://lupyuen.github.io/images/pr-squash4.png)

1.  Click "__I'm Sure__"

    ![Squash Commits with GitHub Desktop](https://lupyuen.github.io/images/pr-squash5.png)

    And we're ready to merge upstream! (Like the salmon)

_What if we prefer the Git Command Line?_

Here are the steps to Squash Commits with the __Git Command Line__...

-   [__"Flight rules for Git: I need to combine commits"__](https://github.com/k88hudson/git-flight-rules#i-need-to-combine-commits)

![Taking a long walk](https://lupyuen.github.io/images/pr-walk.jpg)

# Meditate

__Breathe. Take a break.__

We're about to make __NuttX History__... Our changes will be recorded for posterity!

Take a long walk and ponder...

-   __Who might benefit__ from our Pull Request

-   How we might __best help them__

(I walked 12 km for 3 hours while meditating on my Pull Request)

If we get an inspiration or epiphany, touch up the Pull Request.

(And resquash the commits)

_What's your epiphany?_

Through my Pull Requests, I hope to turn NuttX into a valuable tool for teaching the internals of __Smartphone Operating Systems__.

That's my motivation for [__porting NuttX to PINE64 PinePhone__](https://github.com/apache/nuttx/pull/7692).

But for now... Let's finish our Pull Request!

![Submit the Pull Request](https://lupyuen.github.io/images/pr-pullrequest1.png)

# Submit the Pull Request

Finally it's time to submit our Pull Request!

1.  Create the __Pull Request__ (pic above)

1.  Verify that it has only __One Single Commit__ (pic above)

    [(Squash the Commits)](https://lupyuen.github.io/articles/pr#squash-the-commits)

1.  Copy these into the Pull Request...

    [__Title__](https://lupyuen.github.io/articles/pr#title)

    [__Summary__](https://lupyuen.github.io/articles/pr#summary)

    [__Impact__](https://lupyuen.github.io/articles/pr#impact)

    [__Testing__](https://lupyuen.github.io/articles/pr#testing)

    [(Like this)](https://github.com/apache/nuttx/pull/7630)

1.  Submit the Pull Request

1.  Wait for the __Automated Checks__ to be completed (might take an hour)...

    ![Automated Checks for Pull Request](https://lupyuen.github.io/images/pr-pullrequest2.png)

1.  __Fix any errors__ in the Automated Checks

    [(Watch out for __CMake and Ninja__ errors)](https://lupyuen.github.io/articles/pr#appendix-build-nuttx-with-cmake)

    [(Fixing the __NuttX Config defconfig__ errors)](https://lupyuen.github.io/articles/pr#appendix-nuttx-configuration-files)

1.  Wait for the NuttX Team to __review and comment__ on our Pull Request.

    This might take a while (due to the time zones)... Grab a coffee and standby for fixes!

    [(I bake sourdough while waiting)](https://lupyuen.github.io/articles/sourdough)

    If all goes Hunky Dory, our Pull Request will be __approved and merged!__ 🎉

Sometimes we need to __Rebase To The Latest Master__ due to updates in the GitHub Actions Workflow (Continuous Integration Script). Here's how...

1.  Browse to the __`master`__ branch of our __`nuttx`__ repository.

    Click "__Sync Fork → Update Branch__"

    (Pic below)

1.  Launch [__GitHub Desktop__](https://desktop.github.com/)

    Click "__File → Add Local Repository__"

    Select our downloaded __`nuttx`__ folder.

    Click "__Add Repository__"

1.  Check that the __Current Branch__ is our Working Branch for the Pull Request.

    (Like __`gic`__ branch)

1.  Click "__Fetch Origin__"

1.  Click "__Branch → Rebase Current Branch__"

    Select the __`master`__ branch

    Click "__Rebase__" and "__Begin Rebase__"

1.  Click "__Force Push Origin__" and "__I'm Sure__"

    [(See the official docs)](https://docs.github.com/en/desktop/contributing-and-collaborating-using-github-desktop/keeping-your-local-repository-in-sync-with-github/syncing-your-branch#rebasing-your-project-branch-onto-another-branch)

_Is it really OK to Rebase To The Latest Master?_

Normally when the GitHub Actions build fails (not due to our code), I take a [__peek at other recent Pull Requests__](https://github.com/apache/nuttx/pulls).

If they didn't fail the build, then it's probably OK to Rebase with Master to force the Rebuild.

![Pull Updates from NuttX Mainline](https://lupyuen.github.io/images/pr-update.png)

# Update Our Repositories

[__(Watch the Video on YouTube)__](https://youtu.be/p6ly3EBhtpo?si=ArHLgnu5JWLb5FtW)

After our Pull Request has been merged into NuttX Mainline, __pull the updates__ into our repositories...

1.  Browse to the __`master`__ branch of our __`nuttx`__ repository.

    Click "__Sync Fork → Update Branch__"

    (Pic above)

1.  Do the same for the __`master`__ branch of our __`apps`__ repository.

    Click "__Sync Fork → Update Branch__"

1.  Test the code from the __`master`__ branch of our __`nuttx`__ and __`apps`__ repositories.

When we're ready to add our __next awesome feature__...

1.  Pull the updates from NuttX Mainline into our __`nuttx`__ repository...

    Select the __`master`__ branch.

    Click "__Sync Fork → Update Branch__"

    (Pic above)

1.  Do the same for our __`apps`__ repository.

1.  In our __`nuttx`__ repository, click __`master`__.

    Enter the name of our new branch.
    
    Click "__Create Branch__"

    ![Create Branch](https://lupyuen.github.io/images/pr-branch.png)

1.  Do the same for our __`apps`__ repository.

1.  Modify the code in the new branch of our __`nuttx`__ and __`apps`__ repositories.

    Build, test and submit our new Pull Request.

And that's the Complete Lifecycle of a Pull Request for Apache NuttX RTOS!

One last thing: Please help to validate that the __NuttX Release works OK__! We really appreciate your help with this... 🙏

-   [__"Validate NuttX Release"__](https://lupyuen.github.io/articles/pr#appendix-validate-nuttx-release)

# What's Next

I hope this article will be helpful for folks contributing code to NuttX for the very first time.

In the next article we'll explain this complex Pull Request that adds a __new SoC and new Board__ to NuttX. Stay Tuned!

- [__"Add the NuttX Arch and Board"__](https://lupyuen.github.io/articles/release#add-the-nuttx-arch-and-board)

- [__"Update the NuttX Docs"__](https://lupyuen.github.io/articles/release#update-the-nuttx-docs)

Many Thanks to my [__GitHub Sponsors__](https://github.com/sponsors/lupyuen) for supporting my work! This article wouldn't have been possible without your support.

-   [__Sponsor me a coffee__](https://github.com/sponsors/lupyuen)

-   [__My Current Project: "Apache NuttX RTOS for Ox64 BL808"__](https://github.com/lupyuen/nuttx-ox64)

-   [__My Other Project: "NuttX for Star64 JH7110"__](https://github.com/lupyuen/nuttx-star64)

-   [__Older Project: "NuttX for PinePhone"__](https://github.com/lupyuen/pinephone-nuttx)

-   [__Check out my articles__](https://lupyuen.github.io)

-   [__RSS Feed__](https://lupyuen.github.io/rss.xml)

_Got a question, comment or suggestion? Create an Issue or submit a Pull Request here..._

[__lupyuen.github.io/src/pr.md__](https://github.com/lupyuen/lupyuen.github.io/blob/master/src/pr.md)

# Notes

1.  Here's an excellent guide for the __Git Command Line__...

    [__"Flight rules for Git"__](https://github.com/k88hudson/git-flight-rules)

1.  Converting our code to the [__"NuttX C Coding Standard"__](https://nuttx.apache.org/docs/latest/contributing/coding_style.html)...

    Can we automate this with a __VSCode Extension?__

    Maybe like the [__Linux checkpatch Extension__](https://marketplace.visualstudio.com/items?itemName=idanp.checkpatch), but it will actually auto-reformat our lines?

1.  [__QEMU Emulator__](https://www.qemu.org/docs/master/system/target-arm.html) is incredibly helpful for Regression Testing...

    Can we extend it to __emulate PinePhone__? Maybe just the UART Hardware? Check out the articles...

    [__"(Possibly) Emulate PinePhone with Unicorn Emulator"__](https://lupyuen.github.io/articles/unicorn)

    [__"(Clickable) Call Graph for Apache NuttX Real-Time Operating System"__](https://lupyuen.github.io/articles/unicorn2)

# Appendix: Build NuttX with CMake

_Why did our NuttX Pull Request fail with this CMake Error?_

```yaml
Cmake in present: rv-virt/smp
riscv-none-elf-gcc CMakeFiles/nuttx.dir/empty.c.obj -o nuttx ...
riscv_createstack.c: undefined reference to board_autoled_on
collect2: error: ld returned 1 exit status
ninja: build stopped: subcommand failed.
```

[(Source)](https://github.com/apache/nuttx/actions/runs/10093621571/job/27909785064?pr=12762)

That's because the NuttX Automated Check (Continuous Integration with GitHub Actions) runs __CMake on some NuttX Configurations!__

(Instead of the usual GNU Make)

_What's this CMake?_

CMake is the __Alternative Build System__ for NuttX. We build __NuttX with CMake__ like this...

- [__"Compiling with CMake"__](https://nuttx.apache.org/docs/latest/quickstart/compiling_cmake.html)

```bash
## For macOS: Install Build Tools
brew install pipenv cmake ninja

## For Ubuntu: Install Build Tools
sudo apt install pipenv cmake ninja-build

## Configure NuttX for QEMU RISC-V SMP `rv-virt:smp`
cd nuttx
pipenv install
pipenv shell
pip install kconfiglib
cmake \
  -B build \
  -DBOARD_CONFIG=rv-virt:smp \
  -GNinja

## Build NuttX for QEMU RISC-V SMP
## with CMake and Ninja
cmake --build build

## Note: No more `tools/configure.sh` and `make`!
```

_But why did CMake fail?_

Probably because the CMake Makefiles are __out of sync__ with the GNU Make Makefiles.

Suppose we have a __GNU Make Makefile__: [rv-virt/src/Makefile](https://github.com/apache/nuttx/blob/master/boards/risc-v/qemu-rv/rv-virt/src/Makefile)

```bash
## Always compile `qemu_rv_appinit.c`
CSRCS = qemu_rv_appinit.c

## If `CONFIG_ARCH_LEDS` is Enabled:
## Compile `qemu_rv_autoleds.c`
ifeq ($(CONFIG_ARCH_LEDS),y)
  CSRCS += qemu_rv_autoleds.c
endif

## If `CONFIG_USERLED` is Enabled:
## Compile `qemu_rv_userleds.c`
ifeq ($(CONFIG_USERLED),y)
  CSRCS += qemu_rv_userleds.c
endif
```

Note that it checks the Kconfig Options __CONFIG_ARCH_LEDS__ and __CONFIG_USERLED__. And optionally compiles __qemu_rv_autoleds.c__ and __qemu_rv_userleds.c__.

We need to __Sync the Build Rules__ into the CMake Makefile like so: [rv-virt/src/CMakeLists.txt](https://github.com/apache/nuttx/blob/master/boards/risc-v/qemu-rv/rv-virt/src/CMakeLists.txt)

```bash
## Always compile `qemu_rv_appinit.c`
set(SRCS qemu_rv_appinit.c)

## If `CONFIG_ARCH_LEDS` is Enabled:
## Compile `qemu_rv_autoleds.c`
if(CONFIG_ARCH_LEDS)
  list(APPEND SRCS qemu_rv_autoleds.c)
endif()

## If `CONFIG_USERLED` is Enabled:
## Compile `qemu_rv_userleds.c`
if(CONFIG_USERLED)
  list(APPEND SRCS qemu_rv_userleds.c)
endif()
```

This will ensure that the Build Rules are consistent across the GNU Make Makefiles and CMake Makefiles. [(Like this)](https://github.com/apache/nuttx/pull/12762/files#diff-bae223efa0f0ca8d345bef6373514be02c79bdfa8da568b2029fd54e3d268e34)

# Appendix: NuttX Configuration Files

_Why did our NuttX Pull Request fail with this defconfig error?_

```bash
Normalize rv-virt/leds64
Saving the new configuration file
HEAD detached at pull/12762/merge
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
  modified:   boards/risc-v/qemu-rv/rv-virt/configs/leds64/defconfig
  no changes added to commit (use "git add" and/or "git commit -a")
```

[(Source)](https://github.com/apache/nuttx/actions/runs/10093621571/job/27909785064?pr=12762)

This means that the NuttX Configuration File for __rv-virt:leds64__ has a problem, like a Missing Newline.

To fix it, run __tools/refresh.sh__: [nuttx/pull/12762](https://github.com/apache/nuttx/pull/12762#issuecomment-2250656302)

```bash
## Normalize the `defconfig` for `rv-virt:leds64`
$ cd nuttx
$ tools/refresh.sh --silent rv-virt:leds64
  Normalize rv-virt:leds64
  < CONFIG_USERLED_LOWER=y
  \ No newline at end of file
  ---
  > CONFIG_USERLED_LOWER=y
  Saving the new configuration file

## Remember to commit the updated `defconfig`!
$ git status
  modified: boards/risc-v/qemu-rv/rv-virt/configs/leds64/defconfig
```

__refresh.sh__ will also fix Kconfig Options that are misplaced...

```bash
## `CONFIG_USERLED_LOWER` is misplaced in `rv-virt:leds64`
## Remember to commit the updated `defconfig`!
$ tools/refresh.sh --silent rv-virt:leds64
  Normalize rv-virt:leds64
  74d73
  < CONFIG_USERLED_LOWER=y
  75a75
  > CONFIG_USERLED_LOWER=y
  Saving the new configuration file
```

_What else will cause defconfig errors?_

When we modify the __Kconfig__ Configuration Files, remember to update the __defconfig__ Configuration Files! [(Like this)](https://github.com/apache/nuttx/pull/9243#issuecomment-1542918859)

If we forget to update __defconfig__...

```text
Configuration/Tool: pinephone/sensor
Building NuttX...
Normalize pinephone/sensor
71d70
< CONFIG_UART1_SERIAL_CONSOLE=y
Saving the new configuration file
HEAD detached at pull/9243/merge
Changes not staged for commit:
(use "git add <file>..." to update what will be committed)
(use "git restore <file>..." to discard changes in working directory)
    modified:   boards/arm64/a64/pinephone/configs/sensor/defconfig
```

[(Source)](https://github.com/apache/nuttx/pull/9243#issuecomment-1542918859)

_How to create a new defconfig?_

To __Create or Update a defconfig__, do this...

```bash
## TODO: Change this to your
## `boards/<archname>/<chipname>/<boardname>/config/<configname>`
cd nuttx
mkdir -p boards/risc-v/bl808/ox64/configs/nsh

## Create or Update the NuttX Config
make menuconfig \
  && make savedefconfig \
  && grep -v CONFIG_HOST defconfig \
  >boards/risc-v/bl808/ox64/configs/nsh/defconfig
```

_Some Default Settings in .config are missing from defconfig. Can we copy them ourselves to defconfig?_

Sorry it won't work. Suppose we copy these Default UART3 Settings from __.config__ to __defconfig__ (to hard-code the UART3 Baud Rate)...

```text
CONFIG_UART3_BAUD=115200
CONFIG_UART3_BITS=8
CONFIG_UART3_PARITY=0
CONFIG_UART3_2STOP=0
```

The Auto-Build will fail with the error below. Thus we can't copy any Default Settings in __.config__ to __defconfig__.

```text
Configuration/Tool: pinephone/modem
Building NuttX...
Normalize pinephone/modem
69,72d68
< CONFIG_UART3_BAUD=115200
< CONFIG_UART3_BITS=8
< CONFIG_UART3_PARITY=0
< CONFIG_UART3_2STOP=0
Saving the new configuration file
HEAD detached at pull/9304/merge
Changes not staged for commit:
(use "git add <file>..." to update what will be committed)
(use "git restore <file>..." to discard changes in working directory)
    modified:   boards/arm64/a64/pinephone/configs/modem/defconfig
```

[(Source)](https://github.com/apache/nuttx/actions/runs/4997328093/jobs/8951602341)

_What if the auto-build fails with "Untracked etctmp"?_

```text
HEAD detached at pull/11379/merge
Untracked files: (use "git add <file>..." to include in what will be committed)
boards/risc-v/k230/canmv230/src/etctmp.c
boards/risc-v/k230/canmv230/src/etctmp/
```

[(Source)](https://github.com/apache/nuttx/actions/runs/7203675079/job/19625255417?pr=11379)

Check that we've added "__etctmp__" to the __Board-Specific Git Ignore__: [boards/risc-v/jh7110/star64/src/.gitignore](https://github.com/apache/nuttx/blob/master/boards/risc-v/jh7110/star64/src/.gitignore)

```text
etctmp
etctmp.c
```

# Appendix: Validate NuttX Release

_For each Official Release of NuttX, how do we check if it runs OK on all devices? Like PinePhone, ESP32, BL602, ..._

NuttX needs to be __tested on every device__, and we need your help! 🙏

Before every Official Release of NuttX, a __Validation Request__ will be broadcast on the [__NuttX Developers Mailing List__](https://nuttx.apache.org/community/)...

-   [__Validation Request for NuttX Release__](https://www.mail-archive.com/dev@nuttx.apache.org/msg09563.html)

Follow the instructions here to validate that the NuttX Release __builds correctly and runs OK__ on your device...

-   [__Validating a Staged Release__](https://cwiki.apache.org/confluence/display/NUTTX/Validating+a+staged+Release)

    (See below for the updates)

Here's the script I run to __validate NuttX on PinePhone__...

-   [__Validation Script for PinePhone: release.sh__](https://gist.github.com/lupyuen/a08d3d478beefc5a492ed2dae39438f3)

And here's the output of the __validation script__...

-   [__Validation Output for PinePhone: release.log__](https://gist.github.com/lupyuen/5760e0375d44a06b3c730a10614e4d24)

Boot NuttX on our device, run "__uname -a__" and "__free__"...

```text
NuttShell (NSH) NuttX-12.1.0
nsh> uname -a
NuttX 12.1.0 d40f4032fc Apr 12 2023 07:11:20 arm64 pinephone
nsh> free
      total     used   free      largest   nused nfree
Umem: 133414240 550768 132863472 132863376 56    2
```

The __NuttX Hash__ (like "d40f4032fc" above) should match the [__Validation Request__](https://www.mail-archive.com/dev@nuttx.apache.org/msg09563.html).

Copy the above into a __Validation Response__ email...

-   [__Validation Response for NuttX Release__](https://www.mail-archive.com/dev@nuttx.apache.org/msg09565.html)

And send back to the Mailing List. (Assuming all is hunky dory)

Since there are so many NuttX devices, we really appreciate your help with the NuttX Validation! 🙏

_What are the updates to the NuttX Validation Instructions?_

The [__NuttX Validation Instructions__](https://cwiki.apache.org/confluence/display/NUTTX/Validating+a+staged+Release) should be updated...

-   To verify the NuttX Signature, we need to __import the NuttX Keys__...

    ```bash
    wget https://dist.apache.org/repos/dist/dev/nuttx/KEYS
    gpg --import KEYS
    ```

-   We also need to __trust the NuttX Keys__...

    ```bash
    gpg --edit-key 9208D2E4B800D66F749AD4E94137A71698C5E4DB
    ```

    (That's the RSA Key Fingerprint from "__gpg --verify__")

    Then enter "__trust__" and "__5__"

-   The file "__DISCLAIMER-WIP__" no longer exists in the __nuttx__ and __apps__ folders
