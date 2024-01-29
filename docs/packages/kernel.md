# SIG/Cloud Kernel Package
## Requirements
* RESF Account: https://accounts.rockylinux.org/
* srpmproc: https://github.com/rocky-linux/srpmproc
* sideline: https://github.com/rocky-linux/sideline

## General Process
The RESF primary purpose is to rebuild and debrand a 1:1 bug replica of RHEL and have built tools that support this process. The SIGs are modifications to that initial purpose while reusing the tooling already developed for the base dist-git process.  The SIG dist-gits are controlled by automation that executes the `srpmproc` command so to modify the dist-git in any manner a developer needs to integrate their changes to the `sig/cloud/patch` configs related to that SIG. [SIG/CLOUD - SRPMPROC Kernel Patches](https://git.rockylinux.org/sig/cloud/kernel) -> [SIG/CLOUD - Kernel dist-git](https://git.rockylinux.org/sig/cloud/rpms/kernel)
`srpmproc` is designed to only modify the `dist-git` and if a SIG wants a particular driver branch, that cleanly applies, they need to use another program called `sideline` to create a monolithic patch against a point in time source code base.  That is then added to the `SIDELINE/_supporting` directory of the `sig/cloud/patch/kernel` directory.  `srpmproc` knows that these patches are to be included into `SOURCES/` and added to `applypatch` directives in the `kernel.spec` with appropriate config settings for `srpmproc`.

## Diagram Production
```
                |--------------------------|
                |      : SRPMPRROC :       |
                | Pulls RESF dist-git      |
                | Alters dist-git files    |
                | Injects SIDELINE patches |
     |---Pull---|  into soruce/ and .spec  |---Automation:--|
     |          |--------------------------|      Push      |
     v                                                      v
|----------|                                          |-----------|
| : RESF : |                                          |  : RESF : |
| dist-git |                                          | SIG/CLOUD |
|----------|                                          |  dist-git |
                                                      |-----------|
```

## Developer Process
The general process by the RESF was not initially created with ad-hoc development in mind but rather pulling entire branches from upstream into the current RESF base releases.  The following is to provide a general guide on how to develop with `srpmproc` as the `SIG/CLOUD` dist-git is not to be modified directly for development support.  
Basic Process
1. Clone `sig/cloud/patch/kernel` repo
2. Run Local `srpmproc`
3. Build local git repo for kernel source
4. Make Changes, local builds and testing (not covered in doc)
5. Source changes from local build to `sig/cloud/patch/kernel`
6. Regenerate `dist-git` with srpmproc
7. Mock test rpm builds.

The following sections are going to be based off real work that was ultimately abandoned for Rocky8.

### Prep for SRPMPROC: Cloning patch/kernel
This is the general prep of the directories used.  Note some directions for `srpmproc` use a temp directory, that is not used here as depending on how long development can last you could lose important work, specifically around the `srpmproc-cache`

Create Needed Directories
```
mkdir -p SIG/cloud/
cd SIG/cloud/
mkdir -p {rpms,patch,src,srpmproc-cache,src-git}
```

Clone the configuration Directives and Patches repo
```
git clone ssh://git@git.rockylinux.org:22220/sig/cloud/patch/kernel.git patch/kernel.git
```

Script example used to manage running `srpmproc` during 
```
#!/bin/bash
set -x

MYHOME=`pwd`
~/bin/srpmproc --cdn rocky8 --version 8 \
    --upstream-prefix "file://${MYHOME}/" \
    --storage-addr file:///${MYHOME}/srpmproc-cache \
    --import-branch-prefix 'r' --strict-branch-mode \
    --rpm-prefix "https://git.rockylinux.org/staging/rpms" \
    --ssh-key-location ~/.ssh/resfgit \
    --source-rpm kernel --tmpfs-mode kernel
```

### `srpmproc` 1st run
This step will be missing a lot of output just so that we can focus on the results and where we source building a local source git tree.

RUN SRPMPROC with script or by hand.
```
./sprmproc.sh
```

After `srpmproc` runs you'll have a a new directory (`kernel`) in your current path that will have a working copy of the `sig/cloud` `dist-git` used to build the RPMs for `sig/cloud` 

Example:
```
$ ls -a kernel/r8/
.  ..  .gitignore  .kernel.checksum  .kernel.metadata  SOURCES  SPECS
```

### Build Local git tree repo
At this point the dist-git can extracted and modified for change however the individual wants.
Below is how the author did this.

Extract the TarBall note this was the original 8.9 change and the exact version has incremented.
```
tar xvf kernel/r8/SOURCES/linux-4.18.0-513.9.1.el8_9.tar.xz -C src-git/
cd src-git/linux-4.18.0-513.9.1.el8_9/
```

Commit to a local branch so we have a starting point.
```
git init .
git add .
git commit
```
Results:
```
$ git log
commit 4be223b7a767b0d9ac27525ab26fc37ebdea52bb (HEAD -> mainline)
Author: Jonathan Maple <jmaple@ciq.com>
Date:   Tue Jan 2 15:44:17 2024 -0500

    Initial Commit from RESF Source
```

Find and apply patches from `kernel.spec`
```
egrep -i "apply.*\.patch" ../../kernel/r8/SPECS/kernel.spec
```
Results:
```
ApplyOptionalPatch linux-kernel-test.patch
ApplyPatch 4-backport-changes-from-https-kernel-googlesource-com-pub-scm-linux-kernel-git-torvalds-linux-tag-v6-5.patch
```
Note: `linux-kernel-test.patch` is empty.
Note2: `4-backport-changes-from-https-kernel-googlesource-com-pub-scm-linux-kernel-git-torvalds-linux-tag-v6-5.patch` is a massive monopatch.

Apply Patches
```
patch -p1  < ../../patch/kernel.git/SIDELINE/_supporting/4-backport-changes-from-https-kernel-googlesource-com-pub-scm-linux-kernel-git-torvalds-linux-tag-v6-5.patch
```
Results:
```
patching file drivers/net/ethernet/google/Kconfig
patching file drivers/net/ethernet/google/gve/gve.h
patching file drivers/net/ethernet/google/gve/gve_adminq.c
patching file drivers/net/ethernet/google/gve/gve_adminq.h
patching file drivers/net/ethernet/google/gve/gve_desc_dqo.h
patching file drivers/net/ethernet/google/gve/gve_ethtool.c
patching file drivers/net/ethernet/google/gve/gve_main.c
patching file drivers/net/ethernet/google/gve/gve_rx.c
patching file drivers/net/ethernet/google/gve/gve_rx_dqo.c
patching file drivers/net/ethernet/google/gve/gve_tx.c
patching file drivers/net/ethernet/google/gve/gve_tx_dqo.c
patching file drivers/net/ethernet/google/gve/gve_utils.c
patching file drivers/net/ethernet/google/gve/gve_utils.h
```

Commit changes:
```
git commit .
```
Results:
```
$ git log
commit 26523a2078fe35af623eb7c885be23c9d1078c32 (HEAD -> mainline)
Author: Jonathan Maple <jmaple@ciq.com>
Date:   Tue Jan 2 15:50:54 2024 -0500

    Add RESF Google Patch

    4-backport-changes-from-https-kernel-googlesource-com-pub-scm-linux-kernel-git-torvalds-linux-tag-v6-5.patch

```

### Backporting process (cherry-picks mostly)
This is ultimately up to the back porter but due to the massive monolithic patch that is preferred by `srpmproc` and its current configuration changes its best to leave some markers in the backport commits.  In the following example we used these markers to make sure we could reproduce the upstream change log for future cherry-picks
```
<subject unaltered >

<ticket_system> <ticket>
<opt: CVE> <CVE-NUMBER>
commit <sha1 of original commit>
<opt: upstream-diff> <diff commtent on why it differs from upstream>

<original commit message>
\t<original signoffs> (indented as we don't nessicarily want to notify everyone for this)
(cherry picked from commit: <sha1>)
Signed-off-by: Author <email>
```
Note the last two items are auto added by `git cherry-pick -nsx <sha1>`

EXAMPLE of `cherry-pick` with output:
```
$ git cherry-pick -nsx 4775bc63f880
Auto-merging arch/arm/include/asm/arch_timer.h
Auto-merging arch/arm64/include/asm/arch_timer.h
Auto-merging drivers/clocksource/arm_arch_timer.c

$ git status
On branch ampereone
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	modified:   arch/arm/include/asm/arch_timer.h
	modified:   arch/arm64/include/asm/arch_timer.h
	modified:   drivers/clocksource/arm_arch_timer.c

$ git add .

$ git commit

[ampereone 8be2a2c2ce27] clocksource/arm_arch_timer: Add build-time guards for unhandled register accesses
 3 files changed, 32 insertions(+), 1 deletion(-)

$ git log
commit 8be2a2c2ce27af0fd99ecc72da1977685021afd1 (HEAD -> ampereone)
Author: Jonathan Maple <jmaple@ciq.com>
Date:   Wed Jan 10 16:10:14 2024 -0500

    clocksource/arm_arch_timer: Add build-time guards for unhandled register accesses

    jira ROC-2809
    commit 4775bc63f880001ee4fbd6456b12ab04674149e3

    As we are about to change the registers that are used by the driver,
    start by adding build-time checks to ensure that we always handle
    all registers and access modes.

        Suggested-by: Mark Rutland <mark.rutland@arm.com>
        Signed-off-by: Marc Zyngier <maz@kernel.org>
        Link: https://lore.kernel.org/r/20211017124225.3018098-2-maz@kernel.org
        Signed-off-by: Daniel Lezcano <daniel.lezcano@linaro.org>

    (cherry picked from commit 4775bc63f880001ee4fbd6456b12ab04674149e3)
    Signed-off-by: Jonathan Maple <jmaple@ciq.com>

ommit 26523a2078fe35af623eb7c885be23c9d1078c32 (mainline)
Author: Jonathan Maple <jmaple@ciq.com>
Date:   Tue Jan 2 15:50:54 2024 -0500

    Add RESF Google Patch

    4-backport-changes-from-https-kernel-googlesource-com-pub-scm-linux-kernel-git-torvalds-linux-tag-v6-5.patch

commit 4be223b7a767b0d9ac27525ab26fc37ebdea52bb
Author: Jonathan Maple <jmaple@ciq.com>
Date:   Tue Jan 2 15:44:17 2024 -0500

    Initial Commit from RESF Source
```

#### Local Git Tree state after desired patches + known bug fixes:
```
 git log --oneline
c92984ca64bb (HEAD -> ampereone) clocksource/drivers/arm_arch_timer: Remove arch_timer_rate1
1a4edf30d579 clocksource/drivers/arm_arch_timer: Fix CNTPCT_LO and CNTVCT_LO value
c50e0c82b622 clocksource/drivers/arm_arch_timer: Fix XGene-1 TVAL register math error
c4b9ba6322bd arm64: Fix bit-shifting UB in the MIDR_CPU_MODEL() macro
d6c3b12badb6 clocksource/drivers/arm_arch_timer: Force inlining of erratum_set_next_event_generic()
612170156e8e clocksource/drivers/arm_arch_timer: Fix handling of ARM erratum 858921
ecd5b3bc64e6 clocksource/drivers/arm_arch_timer: Disable timer before programming CVAL
d0736a304c83 clocksource/drivers/arm_arch_timer: limit XGene-1 workaround
c44180277750 clocksource/drivers/arch_arm_timer: Move workaround synchronisation around
4245d37d01b5 clocksource/drivers/arm_arch_timer: Fix masking for high freq counters
c9a41f6ca50a clocksource/drivers/arm_arch_timer: Drop unnecessary ISB on CVAL programming
79cbc595435b clocksource/drivers/arm_arch_timer: Remove any trace of the TVAL programming interface
5ce552b7c335 clocksource/drivers/arm_arch_timer: Work around broken CVAL implementations
e616b69d5656 clocksource/drivers/arm_arch_timer: Advertise 56bit timer to the core code
7d8301d23b2d clocksource/drivers/arm_arch_timer: Move MMIO timer programming over to CVAL
96f3d866ac46 clocksource/drivers/arm_arch_timer: Fix MMIO base address vs callback ordering issue
405b16cdf7ea clocksource/drivers/arm_arch_timer: Add __ro_after_init and __init
a674842337c0 clocksource/drivers/arm_arch_timer: Move drop _tval from erratum function names
7362b3b1507d clocksource/drivers/arm_arch_timer: Move system register timer programming over to CVAL
fa7e8c083474 clocksource/drivers/arm_arch_timer: Extend write side of timer register accessors to u64
39df82354a3b clocksource/drivers/arm_arch_timer: Drop CNT*_TVAL read accessors
8be2a2c2ce27 clocksource/arm_arch_timer: Add build-time guards for unhandled register accesses
26523a2078fe (mainline) Add RESF Google Patch
4be223b7a767 Initial Commit from RESF Source
```

#### Local Kernel Builds
Compile the kernel in place (outside the scope of this document).
You could also export all these patches and manually add them to the `dist-git` like by many other `dist-git` builds however they can not be submitted to the SIG from this state. So this would be exclusively for your local testing.  If you choose to do this and are not sure how to use `mock` builds, [Skip's Blog](https://skip.linuxdn.org/blog.html#005_Rocky5_BuildLab_Part1) as a relevant primer tutorial for Rocky

### Prepare changes for srpmproc integration
This is the part deviates pretty drastically from normal developer workflows.

#### Generate Monolithic Patch
In this case you'll want to take the commit you're branched off to the HEAD of your branch and create a single patch.  Using the local commits in the above output we'll want `26523a2078fe` to be the first sha and `c92984ca64bb` to be the final

```
git diff 26523a2078fe c92984ca64bb > ../ampereone.patch
```

#### Generate cherry-pick patch and changelog entries (a basic method)
This segment is a manual way of producing the needed changelog entries and a nicer patch header than a standard `git diff`. And is being used to show examples of how and where to contribute.
```
echo -e "Backport AmpereOne CPU clocksource and related bug fixes\n\nThis is just a monolithic diff.\nThe Commits backported are in git log order.\n" > ../patch_msg.resf.txt;
echo -n "" > ../srpmproc_changelog.resf.txt;

git log --pretty="format:%b" \
    | grep ^commit | awk '{print $2}' \
    | sed 's/)$//' | \
    while read commit; do \
      git log -1 --pretty="format:%h %s" ${commit} >> ../patch_msg.resf.txt; \
      echo "\n" >> ../patch_msg.resf.txt; \
      git log -1 --pretty="format:message:    \"%h - %an - %cs - %s\"" ${commit} >> ../srpmproc_changelog.resf.txt; \
      echo "\n" >> ../srpmproc_changelog.resf.txt; \
    done;

echo -e "\n\nBackport-By: Jonathan Maple (jmaple@ciq.com)\n\n" >> ../patch_msg.resf.txt;
cat ../patch_msg.resf.txt > ../ampereone.resf.patch;
cat ../ampereone.patch >> ../ampereone.resf.patch;
```
##### Example Results
For `srpmproc` specfile changelog section
```
$ cat ../srpmproc_changelog.resf.txt
message:    "4f9f4f0f6261 - Jisheng Zhang - 2021-06-03 - clocksource/drivers/arm_arch_timer: Remove arch_timer_rate1"\n
message:    "af246cc6d0ed - Yang Guo - 2022-09-27 - clocksource/drivers/arm_arch_timer: Fix CNTPCT_LO and CNTVCT_LO value"\n
[19 lines ommited for readability]
message:    "4775bc63f880 - Marc Zyngier - 2021-10-17 - clocksource/arm_arch_timer: Add build-time guards for unhandled register accesses"\n
```

Patch File so there is local reference to what was done.  Its important to show how the monolithic patch is generated in the event the changes need to be re-played in the event of a code correction:
```
$ cat ../ampereone.resf.patch
Backport AmpereOne CPU clocksource and related bug fixes

This is just a monolithic diff.
The Commits backported are in git log order.

4f9f4f0f6261 clocksource/drivers/arm_arch_timer: Remove arch_timer_rate1\n
af246cc6d0ed clocksource/drivers/arm_arch_timer: Fix CNTPCT_LO and CNTVCT_LO value\n
[19 lines ommited for readability]
4775bc63f880 clocksource/arm_arch_timer: Add build-time guards for unhandled register accesses\n


Backport-By: Jonathan Maple (jmaple@ciq.com)


diff --git a/arch/arm/include/asm/arch_timer.h b/arch/arm/include/asm/arch_timer.h
index 4b66ecd6be99..5063cdaad21d 100644
--- a/arch/arm/include/asm/arch_timer.h
+++ b/arch/arm/include/asm/arch_timer.h
```

#### Copy Monolithic Patch
```
cp ../ampereone.resf.patch ../../patch/kernel.git/SIDELINE/_supporting/
```

### Modify SRPMPROC directives
This requires manual modification of directives file as `srpmproc` needs to be told what to do with the new file.

Some unique notes about `srpmproc`
* `patch` - directive applies a patch to the `dist-git` files themselves, NOT adding a patch the `.spec` file
* `add: {file: <path>}` is what will copy the file from `srpmproc` directories INTO the `dist-git` `SOURCES/` location.
* `spec_change` is where we configure changes to the specfile like would be done to a standard `dist-git`
  * This is processed in order, so `search_and_replace` will be processed in order for adding patches.  Make sure the patches are applied in correct order.
  * Since the file is processed in order the changelog items need to be placed in reverse order ie newest item at the top.

Example of a modification, NOTE `search_and_replace` were below the `chagnelog` entries.  They were moved on this abandoned work such that the `file` and `search_and_replace` were located generally in order relation to how they would be modified in a pure `dist-git` style repo.  In a hope to avoid jumping back and forth in the file.
```
$ git diff
diff --git a/ROCKY/CFG/directives.cfg b/ROCKY/CFG/directives.cfg
index 6c21c18..2889b25 100644
--- a/ROCKY/CFG/directives.cfg
+++ b/ROCKY/CFG/directives.cfg
@@ -1,12 +1,61 @@
 add:  {
   file:  "SIDELINE/_supporting/4-backport-changes-from-https-kernel-googlesource-com-pub-scm-linux-kernel-git-torvalds-linux-tag-v6-5.patch"
 }
+add: {
+  file:  "SIDELINE/_supporting/ampereone.patch
+}
 spec_change:  {
   file:  {
     name:  "4-backport-changes-from-https-kernel-googlesource-com-pub-scm-linux-kernel-git-torvalds-linux-tag-v6-5.patch"
     type:  Patch
     add:  true
   }
+  search_and_replace:  {
+      any:  true
+      find:  "ApplyOptionalPatch linux-kernel-test.patch"
+      replace:  "ApplyOptionalPatch linux-kernel-test.patch\nApplyPatch 4-backport-changes-from-https-kernel-googlesource-com-pub-scm-linux-kernel-git-torvalds-linux-tag-v6-5.patch"
+      n:  1
+  }
+
+  file: {
+      name: ampereone.patch
+      type: Patch
+      add: true
+  }
+  search_and_replace:  {
+      any:  true
+      find:  "ApplyPatch 4-backport-changes-from-https-kernel-googlesource-com-pub-scm-linux-kernel-git-torvalds-linux-tag-v6-5.patch"
+      replace:  "ApplyPatch 4-backport-changes-from-https-kernel-googlesource-com-pub-scm-linux-kernel-git-torvalds-linux-tag-v6-5.patch\nApplyPatch google_ampere_clocksource_diff.patch"
+      n: 1
+  }
+
+  changelog: {
+    author_name: "Jonathan Maple"
+    author_email: "jmaple@ciq.com"
+    message:    "4f9f4f0f6261 - Jisheng Zhang - 2021-06-03 - clocksource/drivers/arm_arch_timer: Remove arch_timer_rate1"
+    message:    "af246cc6d0ed - Yang Guo - 2022-09-27 - clocksource/drivers/arm_arch_timer: Fix CNTPCT_LO and CNTVCT_LO value"
+    message:    "45ae272a948a - Joe Korty - 2022-12-02 - clocksource/drivers/arm_arch_timer: Fix XGene-1 TVAL register math error"
+    message:    "8ec8490a1950 - D Scott Phillips - 2022-11-09 - arm64: Fix bit-shifting UB in the MIDR_CPU_MODEL() macro"
+    message:    "1edb7e74a7d3 - Marc Zyngier - 2021-12-10 - clocksource/drivers/arm_arch_timer: Force inlining of erratum_set_next_event_generic()"
+    message:    "6c3b62d93e19 - Kunkun Jiang - 2022-09-20 - clocksource/drivers/arm_arch_timer: Fix handling of ARM erratum 858921"
+    message:    "e7d65e40ab5a - Walter Chang - 2023-08-18 - clocksource/drivers/arm_arch_timer: Disable timer before programming CVAL"
+    message:    "851354cbd12b - Andre Przywara - 2023-10-18 - clocksource/drivers/arm_arch_timer: limit XGene-1 workaround"
+    message:    "db26f8f2da92 - Marc Zyngier - 2021-10-18 - clocksource/drivers/arch_arm_timer: Move workaround synchronisation around"
+    message:    "c1153d52c414 - Oliver Upton - 2021-10-18 - clocksource/drivers/arm_arch_timer: Fix masking for high freq counters"
+    message:    "ec8f7f3342c8 - Marc Zyngier - 2021-10-17 - clocksource/drivers/arm_arch_timer: Drop unnecessary ISB on CVAL programming"
+    message:    "41f8d02a6a55 - Marc Zyngier - 2021-10-17 - clocksource/drivers/arm_arch_timer: Remove any trace of the TVAL programming interface"
+    message:    "012f18850452 - Marc Zyngier - 2021-10-17 - clocksource/drivers/arm_arch_timer: Work around broken CVAL implementations"
+    message:    "30aa08da35e0 - Marc Zyngier - 2021-10-17 - clocksource/drivers/arm_arch_timer: Advertise 56bit timer to the core code"
+    message:    "8b82c4f883a7 - Marc Zyngier - 2021-10-17 - clocksource/drivers/arm_arch_timer: Move MMIO timer programming over to CVAL"
+    message:    "72f47a3f0ea4 - Marc Zyngier - 2021-10-17 - clocksource/drivers/arm_arch_timer: Fix MMIO base address vs callback ordering issue"
+    message:    "e2bf384d4329 - Jisheng Zhang - 2021-04-08 - clocksource/drivers/arm_arch_timer: Add __ro_after_init and __init"
+    message:    "ac9ef4f24cb2 - Marc Zyngier - 2021-10-17 - clocksource/drivers/arm_arch_timer: Move drop _tval from erratum function names"
+    message:    "a38b71b0833e - Marc Zyngier - 2021-10-17 - clocksource/drivers/arm_arch_timer: Move system register timer programming over to CVAL"
+    message:    "1e8d929231cf - Marc Zyngier - 2021-10-17 - clocksource/drivers/arm_arch_timer: Extend write side of timer register accessors to u64"
+    message:    "d72689988d67 - Marc Zyngier - 2021-10-17 - clocksource/drivers/arm_arch_timer: Drop CNT*_TVAL read accessors"
+    message:    "4775bc63f880 - Marc Zyngier - 2021-10-17 - clocksource/arm_arch_timer: Add build-time guards for unhandled register accesses"
+  }
+
   changelog:  {
     author_name:  "RESF Sideline (Backporter)"
     author_email:  "releng+sideline@rockylinux.org"
@@ -138,12 +187,6 @@ spec_change:  {
     message:  "f5cedc84a30d2d3d0e0a7f3eb53fbd66d9bf5517 - Catherine Sullivan - 2019-07-01 - gve: Add transmit and receive support"
     message:  "893ce44df56580fb878ca5af9c4a5fd87567da50 - Catherine Sullivan - 2019-07-01 - gve: Add basic driver framework for Compute Engine Virtual NIC"
   }
-  search_and_replace:  {
-    any:  true
-    find:  "ApplyOptionalPatch linux-kernel-test.patch"
-    replace:  "ApplyOptionalPatch linux-kernel-test.patch\nApplyPatch 4-backport-changes-from-https-kernel-googlesource-com-pub-scm-linux-kernel-git-torvalds-linux-tag-v6-5
.patch"
-    n:  1
-  }
   search_and_replace:  {
     any:  true
     find:  "ApplyOptionalPatch debrand-single-cpu.patch"
     
```

### Commit locally and rerun `srpmproc`
As of 2024.01.29 `srpmproc` cannot work off dev branches so this code will need to be on the main `r<version>` branch.  Once this is done re-running `srpmproc` using the above script or exact same command parameters used when creating the local `srpmproc` will use the new local HEAD.

```
$ git log
commit b32760fa1ed8a29145ca05d6bda391256a2aa70c (HEAD -> r8)
Author: Jonathan Maple <jmaple@ciq.com>
Date:   Thu Jan 11 14:54:14 2024 -0500

    Add AmpereOne CPU clocksource patches to r8

    There is a request in the cloud to support fixes for AmpereOne CPUs with
    clocksource fixes in the Rocky8 kernels.  This was a large manual
    backport with cherry-picks and picking up all known bugfixes for pulled
    patches.
```

Example using the new commit:
```
[jmaple@devbox cloud]$ ./srpmproc.sh
2024/01/12 13:25:15 Discovered --cdn distro: rocky8 .  Using override CDN URL Pattern: https://rocky-linux-sources-staging.a1.rockylinux.org/{{.Hash}}
2024/01/12 13:25:15 using tmpfs dir: kernel
2024/01/12 13:25:29 tag: imports/r8/kernel-4.18.0-513.11.1.el8_9
2024/01/12 13:25:29 using remote: file:///home/jmaple/workspace/code/r89builder/SIG/cloud//rpms/kernel.git
2024/01/12 13:25:29 using refspec: +refs/heads/r8:refs/remotes/origin/r8
2024/01/12 13:25:29 set reference to ref: refs/heads/r8
Found branchname that does not start w/ refs/heads ::  r8
[SNIP]
{"branch_versions":{"r8":{"version":"4.18.0","release":"513.11.1.el8_9"}}}

[jmaple@devbox cloud]$ find kernel/ -type f | xargs grep ampereone; find kernel/ -type f | grep ampereone
kernel/r8/SPECS/kernel.spec:Patch1002: ampereone.patch
kernel/r8/SPECS/kernel.spec:ApplyPatch ampereone.patch
kernel/r8/SOURCES/ampereone.patch
```

### Rebuild with local `dist-git`
Reuse methods above

### CREATE PR to RESF
This is following the developer guidelines.
