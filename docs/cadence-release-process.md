# Release Process for ''Cadence Releases''

Before cutting any releases, read the Apache's [Releases Policy](http://www.apache.org/dev/release)

This page describes the technical steps for releasing a `Cadence Release` (see: [versioning-and-release-strategy.md](versioning-and-release-strategy.md)).

TODO: We may want to be using [signed tags](http://git-scm.com/book/en/Git-Basics-Tagging), or at least annotated tags.

## Getting Buy-in & Assigning a Release Manager

 1. Email the dev mailing-list and see if anyone has reason to postpone the release.
   * If so, agree upon a branching date / time.
 1. Ask for a volunteer to be Release Manager for the release (or volunteer yourself)

## Creating JIRA issues

Create the release bug for the Release Candidate:

    ./cordova-coho/coho create-release-bug --version=3.0.0 --username=JiraUser --password=JiraPassword

This bug contains subtasks for each top-level step involved in creating a release.
***For each completed step, add a comment to the relevant JIRA issue.***

## Branch & Tag RC1 for: cordova-js, cordova-mobile-spec and cordova-app-hello-world

This should be done *before* creating branches on other repos.

This step involves:
 * Updating version numbers
 * Creating release branches
 * Creating git tags

Coho automates these steps:

    ./cordova-coho/coho prepare-release-branch --version 2.8.0-rc1 -r js -r app-hello-world -r mobile-spec
    ./cordova-coho/coho repo-status -r js -r app-hello-world -r mobile-spec -b master -b 2.8.x
    # If changes look right:
    ./cordova-coho/coho repo-push -r js -r app-hello-world -r mobile-spec -b master -b 2.8.x
    ./cordova-coho/coho tag-release --version 2.8.0-rc1 -r js -r app-hello-world -r mobile-spec

If the JS ever needs to be re-tagged, rerun the `tag-release` command, and then re-run the `prepare-release-branch` command for the platform repos.

## Branch & Tag RC1 for Platform Repositories

### Before creating the release branch:

 1. Run [Apache RAT](http://creadur.apache.org/rat/) to ensure copyright headers are present
   * `./cordova-coho/coho audit-license-headers -r android | less`
 2. Update the copy of app-hello-world (if there were any changes to it)
   * This usually lives within bin/templates somewhere
   * TODO: More details needed here
 3. For iOS only:
   * Update [CordovaLib/Classes/CDVAvailability.h](https://github.com/apache/incubator-cordova-ios/blob/master/CordovaLib/Classes/CDVAvailability.h)

by adding a new macro for the new version, e.g.

    #define __CORDOVA_2_1_0  20100


and update `CORDOVA_VERSION_MIN_REQUIRED` with the latest version macro, e.g.

    #ifndef CORDOVA_VERSION_MIN_REQUIRED
        #define CORDOVA_VERSION_MIN_REQUIRED __CORDOVA_2_1_0
    #endif

### Creating the release branch

This step involves:
 * Updating cordova.js snapshots
 * Updating version numbers
 * Creating release branches
 * Creating git tags

Coho automates these steps (replace android with your platform):

    ./cordova-coho/coho prepare-release-branch --version 2.8.0-rc1 -r android
    ./cordova-coho/coho repo-status -r android -b master -b 2.8.x
    # If changes look right:
    ./cordova-coho/coho repo-push -r android -b master -b 2.8.x
    ./cordova-coho/coho tag-release --version 2.8.0-rc1 -r android

## Tagging RC1 of cordova-cli

cordova-cli doesn't use a release branch. Follow the instructions at [tools-release-process.md](tools-release-process.md), but in addition:

Update the tool to point to the new repo versions (within `cordova-cli/platforms.js`)

Instead of the normal `npm publish` flow:

    npm publish --tag rc

** WATCH OUT! You may have to run `npm tag cordova@x.x.x latest` due to a bug in npm: https://github.com/npm/npm/issues/4837

## Testing & Documentation

Once all the repos are branched & tagged, we focus on testing & fixing all of the regressions we find.

When a regression is found:

 * Create a JIRA issue for it, and mark it as a blocker.

To submit a fix:

    git checkout master
    git commit -am 'Your commit message'
    git push origin master
    git log     # note the first five or six digits of the commit hash
    git checkout 2.7.x
    git cherry-pick -x commit_hash
    git push origin 2.7.x

### What to Test

 * Run [mobile-spec](http://git-wip-us.apache.org/repos/asf/cordova-mobile-spec.git)
   * Don't forget to set up your white-list
   * Don't forget to run through the manual tests in addition to the automatic tests
   * Test loading the app over HTTP (via "cordova serve" and setting the config.xml start page)
 * Run each platform's ./bin/create script
   * Ensure generated project builds & runs both through an IDE and through the cordova/* scripts
 * Test Project Upgrades (old-style):
   1. Create a project using the previous version of cordova
     * `./cordova-coho/coho foreach "git checkout 2.9.0"`
     * `./cordova-coho/coho foreach -r active-platform "./bin/create foo org.apache.foo foo"`
   2. Upgrade the project via the bin/update_project script:
     * `./cordova-coho/coho foreach "git checkout 3.0.x"`
     * `./cordova-coho/coho foreach -r active-platform "cd foo && ../bin/update_project"`
   3. Test the result:
     * Project should run
     * cordova/version should report the new version
 * Test Project Upgrades (new-style):
   1. Create a project using the previous version of cordova
     * `./cordova-coho/coho foreach "git checkout 2.9.0"`
     * `./cordova-mobile-spec/createmobilespec.sh`
   2. Upgrade the project via the update command:
     * `../cordova-cli/bin/cordova platform update PLATFORM`
   3. Test the result:
     * Project should run
     * cordova/version should report the new version
     * Mobile Spec should still run.

#### Android Extras

 * Unit tests in: [test](https://github.com/apache/incubator-cordova-android/tree/master/test) directory

#### iOS Extras

 * Unit tests in: [CordovaLibTests/CordovaTests.xcodeproj](https://git-wip-us.apache.org/repos/asf?p=cordova-ios.git;a=tree;f=CordovaLibTests;h=88ba8e3c286159151b378efb1b0c39ef26dac550;hb=HEAD)
 * Test the Makefile via `make`
 * Run `bin/diagnose_project` on a newly created project and ensure it reports no errors.

### Documentation To Update

For each repository:
 1. Update RELEASENOTES.md (if the file is missing, use the iOS one as a reference: [RELEASENOTES.md](https://github.com/apache/cordova-ios/blob/master/RELEASENOTES.md))

Grab changes from the previous release until now.

    # Changes:
    git log --pretty=format:'* %s' --topo-order --no-merges origin/3.2.x..origin/3.3.x
    # Commit count:
    git log --pretty=format:'* %s' --topo-order --no-merges origin/3.2.x..origin/3.3.x | wc -l
    # Author Count:
    git log --pretty=format:'%an' --topo-order --no-merges origin/3.2.x..origin/3.3.x | sort | uniq | wc -l

Edit the commit descriptions - don't add the commits verbatim, usually they are meaningless to the user. Only show the ones relevant for the user (fixes, new features)

 2. Update README.md (if necessary)
 3. Ensure the [Upgrade Guide](http://docs.phonegap.com/en/edge/guide_upgrading_index.md.html) for your platform is up-to-date
 4. Ensure the other guides listed in the sidebar are up-to-date for your platform

## Final Tagging (non-RC)

This is done for all repos once testing is complete, and documentation is up-to-date.

Use the same coho commands as for the RCs (it will update JS & VERSION):

    ./cordova-coho/coho prepare-release-branch --version 2.8.0 -r js -r app-hello-world -r mobile-spec
    ./cordova-coho/coho repo-status -r js -r app-hello-world -r mobile-spec -b master -b 2.8.x
    # If changes look right:
    ./cordova-coho/coho repo-push -r js -r app-hello-world -r mobile-spec -b master -b 2.8.x
    ./cordova-coho/coho tag-release --version 2.8.0 -r js -r app-hello-world -r mobile-spec

## Branching & Tagging cordova-docs

 1. Cherry pick relevant commits from master to 2.8.x branch
 2. Generate the docs for the release on the 2.8.x branch.
 3. Commit & tag on the 2.8.x branch.
 4. Cherry pick commit into master.


See [Generating a Version Release](https://git-wip-us.apache.org/repos/asf?p=cordova-docs.git;a=blob;f=README.md#l127) for more details.


## Final Details

### Update cordova.apache.org

 * Refer to [this commit](http://svn.apache.org/viewvc?view=revision&revision=r1478146) (also includes updating the DOAP file)
 * And the instructions at https://svn.apache.org/repos/asf/cordova/site/README.md

### Update the Docs
 1. Upload the new docs to http://cordova.apache.org/docs
   * Website README.md explains [How to update the docs](https://svn.apache.org/repos/asf/cordova/site/README.md)
   * Commit should look like [this one](http://svn.apache.org/viewvc?view=revision&revision=r1478171)
 1. Ask Michael Brooks to update the docs.cordova.io redirect.
   * Check out the branch `cordova-labs:redirect-docs-cordova-io`
   * Repository README.md explains [How to update the HTTP redirect](https://github.com/apache/cordova-labs/tree/redirect-docs-cordova-io#usage)
   * Nodejitsu is limited to one deployer, so Michael Brooks is currently the point of contact.

### Push CLI to npm

Refer to [tools-release-process.md](tools-release-process.md)

### Tell JIRA it's Released

 * Visit https://issues.apache.org/jira/plugins/servlet/project-config/CB/versions
 * Fill in the Release Date field and mark it as released.

### Announce It!
 1. Announce the release to the world!
   * Create a blog post for it (instructions on [sites page README](https://svn.apache.org/repos/asf/cordova/site/README.md))
   * Tweet it on https://twitter.com/apachecordova
   * Announce to [G+ Page](https://plus.google.com/u/0/113178331525415522084/posts)


# Additional Information
 * [IOSReleaseChecklist](https://wiki.apache.org/cordova/IOSReleaseChecklist)
 * [AndroidReleaseChecklist](https://wiki.apache.org/cordova/AndroidReleaseChecklist)

## Moving Tags

If you need to move a tag before the release, here is how to do that:

    $ git tag -d 3.1.0
    Deleted tag '3.1.0' (was 2a9bc20)
    $ git push origin :refs/tags/3.1.0
    To https://git-wip-us.apache.org/repos/asf/cordova-docs.git
     - [deleted]         3.1.0
    $ git tag 3.1.0 7cf9fea03d7d02a13aef97a09a459e8128bd3198
    $ git push origin 3.1.0 --tags
    Total 0 (delta 0), reused 0 (delta 0)
    To https://git-wip-us.apache.org/repos/asf/cordova-docs.git
     * [new tag]         3.1.0 -> 3.1.0

Then send a note to the mailing list:

    To verify you have the updated tag in your local clone, doing a "git rev-parse 3.1.0" in cordova-docs should reply with "7cf9fea03d7d02a13aef97a09a459e8128bd3198". If it is wrong, do "git fetch --tags".

# Official Apache Releases

An official source release contains the source code for the repositories of the Apache Cordova platform, the signing keys and various checks to prove the validity of the release.

Pre-3.0, official source releases were how end-users downloaded Cordova. Now that we use CLI, they are more for historical purposes and are done only for major releases.

A release contains:

    /
    |- KEYS .................................. signing keys
    |- cordova-VERSION-src.zip ............... zip file that contains the src of all platform repos
    |- .md5 .................................. md5 file containing the MD5 Checksum of the src zip
    |- .sha .................................. sha file containing the SHA Hash of the src zip
    |- .asc .................................. asc file that contains the ASCII Armoring of the zip


The `/cordova-VERSION-src.zip/` is the official release artifact and contains the source code for all the platforms, the top level documents concerning licences, notices, disclaimer, and as well the readme for the Apache Cordova project.

    /
    |-changelog
    |-DISCLAIMER
    |-cordova-$PLATFORM.zip  (per platform)
    |-cordova-app-hello-world.zip
    |-cordova-docs.zip
    |-cordova-js.zip
    |-cordova-mobile-spec.zip
    |-LICENSE
    |-NOTICE
    |-README.MD


## Uploading a Release

Create the release .zip with coho:

    ./cordova-coho/coho create-release-snapshot --prev-version 2.7.0 --new-version 2.8.0-rc1 -r release-repos

Upload it to: https://dist.apache.org/repos/dist/release/cordova/

    ./cordova-coho/coho upload-release --new-version 3.0.0 --prev-version 2.9.0

[Update the versions](https://wiki.apache.org/cordova/UpdatingVersionsOnTheCordovaWebsite) on the Cordova website

