# Release Process for ''Core Plugins''

Before cutting any releases, read the Apache's [Releases Policy](http://www.apache.org/dev/release)

If you have not done so already, create a GPG key (see: [setting-up-gpg.md](setting-up-gpg.md)).

Core Plugins are released at most weekly (see: [versioning-and-release-strategy.md](versioning-and-release-strategy.md)).

A plugins release is performed by a single person each week. We call this person the "Release Manager". How to select the Release Manager is still TDB.

TODO: add in RAT instruction (via coho) for next release

TODO: Use perl instead of sed in these commands so they work on Linux.

TODO: We may want to be using [signed tags](http://git-scm.com/book/en/Git-Basics-Tagging), or at least annotated tags.

TODO: Add step about ensuring plugman owner

TODO: Add step about releasing cordova-plugins

TODO: Should not mention testing other than checking medic

## Get Buy-in

 1. Email the dev mailing-list and see if anyone has reason to postpone the release.
   * If so, agree upon a branching date / time.

## Create JIRA issues

 * Create a JIRA issue to track the status of the release.
   * Make it of type "Task"
   * Title should be "Plugins Release _Feb 2, 2014_"
   * Description should be: "Following steps at https://github.com/apache/cordova-coho/blob/master/docs/plugins-release-process.md"
 * Comments should be added to this bug after each top-level step below is taken
 * Set a variable for use later on:


    JIRA="CB-????" # Set this to the release bug.


## Make sure you're up-to-date

    # Update your repos
    ./cordova-coho/coho repo-status -r plugins -b dev -b master
    ./cordova-coho/coho repo-update -r plugins
    ./cordova-coho/coho foreach -r plugins "git checkout dev"

    # Merge any commits mistakenly made to master into dev:
    (for l in cordova-plugin-*; do ( cd $l; git merge master ); done

    # Sanity check and push if needed:
    ./cordova-coho/coho repo-status -r plugins -b dev
    ./cordova-coho/coho repo-push -r plugins -b dev

## Identify which plugins have changes

    ./cordova-coho/coho repo-update -r plugins
    ./cordova-coho/coho foreach -r plugins "git checkout dev"
    ACTIVE=$(for l in cordova-plugin-*; do ( cd $l; git log --pretty=format:'* %s' --topo-order --no-merges master..dev | grep -v "Incremented plugin version" > /dev/null && echo $l); done | xargs echo)

## Ensure license headers are present everywhere:

    ./cordova-coho/coho audit-license-headers -r plugins | less

## Update RELEASENOTES.md & Version
Remove the ''-dev'' suffix on the version in plugin.xml.

    for l in $ACTIVE; do ( cd $l; v="$(grep version= plugin.xml | grep -v xml | head -n1 | cut -d'"' -f2)"; if [ $v = *-dev ); then v2="${v%-dev}"; echo "$l: Setting version to $v2"; sed -i '' -E s:"version=\"$v\":version=\"$v2\":" plugin.xml; fi) ; done

If the changes merit it, manually bump the major / minor version instead of the micro. Manual process, but list the changes via:

    for l in $ACTIVE; do ( cd $l; echo $l; git log --pretty=format:'* %s' --topo-order --no-merges master..dev | grep -v "Incremented plugin version" ); done

Update its RELEASENOTES.md file with changes

    # Add new heading to release notes with version and date
    DATE=$(date "+%h %d, %Y")
    for l in $ACTIVE; do ( cd $l; v="$(grep version= plugin.xml | grep -v xml | head -n1 | cut -d'"' -f2)"; echo -e "\n### $v ($DATE)" >> RELEASENOTES.md; git log --pretty=format:'* %s' --topo-order --no-merges master..dev | grep -v "Incremented plugin version" >> RELEASENOTES.md); done
    # Then curate:
    vim ${ACTIVE// //RELEASENOTES.md }/RELEASENOTES.md

Add a comment to the JIRA issue with the output from:

    for l in $ACTIVE; do ( cd $l; id="$(grep id= plugin.xml | grep -v xml | grep -v engine | grep -v param | head -1 | cut -d'"' -f2)"; v="$(grep version= plugin.xml | grep -v xml | head -n1 | cut -d'"' -f2)"; echo $id@$v; awk "{ if (p) print } /$DATE/ { p = 1 } " < RELEASENOTES.md; echo); done

Commit these two changes together to the `dev` branch

    for l in $ACTIVE; do ( cd $l; v="$(grep version= plugin.xml | grep -v xml | head -n1 | cut -d'"' -f2)"; git commit -am "$JIRA Updated version and RELEASENOTES.md for release $v"); done

## Tag on Dev Branch

    for l in $ACTIVE; do ( cd $l; v="r$(grep version= plugin.xml | grep -v xml | head -n1 | cut -d'"' -f2)"; echo "Tagging $l to $v"; git tag "$v" ); done

## Update dev branch's version

    for l in $ACTIVE; do ( cd $l; v="$(grep version= plugin.xml | grep -v xml | head -n1 | cut -d'"' -f2)"; if [ $v != *-dev ); then v2="$(echo $v|awk -F"." '{$NF+=1}{print $0RT}' OFS="." ORS="")-dev"; echo "$l: Setting version to $v2"; sed -i '' -E s:"version=\"$v\":version=\"$v2\":" plugin.xml; fi) ; done
    for l in $ACTIVE; do (cd $l; git commit -am "$JIRA Incremented plugin version on dev branch." ); done

## Test
 * Create mobilespec using the old versions of plugins (by checking them out to the previous tag)
 * Run through mobilespec, ensuring to do manual tests that relate to changes in the RELEASENOTES.md

## Push Dev Branch
    # Sanity check:
    ./cordova-coho/coho repo-status -r plugins -b dev
    # Push:
    for l in $ACTIVE; do ( cd $l; git push --tags https://git-wip-us.apache.org/repos/asf/$l.git dev); done

## Publish to dist/dev
Ensure you have the svn repos checked out:

    ./cordova-coho/coho repo-clone -r dist -r dist/dev

Create archives from your tags:

    ./cordova-coho/coho create-archive -r ${ACTIVE// / -r } --dest cordova-dist-dev/$JIRA

Sanity Check:

    # Manually double check version numbers are correct on the file names
    # Then run:
    ./cordova-coho/coho verify-archive cordova-dist-dev/$JIRA/*.zip

Upload:

    cd cordova-dist-dev && svn up && svn add $JIRA && svn commit -m "$JIRA Uploading release candidates for plugins release"

* Find your release here: https://dist.apache.org/repos/dist/dev/cordova/

## Prepare Blog Post
 * Combine highlights from RELEASENOTES.md into a Release Announcement blog post
   * Instructions on [sites page README](https://svn.apache.org/repos/asf/cordova/site/README.md)
 * Get blog post proof-read.

## Start VOTE Thread
Send an email to dev ML with:

__Subject:__

    [Vote] Plugins Release

__Body:__

    Please review and vote on the release of this plugins release.

    Release issue: https://issues.apache.org/jira/browse/CB-XXXX

    The plugins have been published to dist/dev:
    https://dist.apache.org/repos/dist/dev/cordova/CB-XXXX/

    The packages were published from their corresponding git tags:
    PASTE OUTPUT OF: ./cordova-coho/coho print-tags -r ${ACTIVE// / -r }

    Upon a successful vote I will upload the archives to dist/, upload them to the Plugins Registry, and post the corresponding blog post.

    Voting will go on for a minimum of 48 hours.

    I vote +1.


## Email the result of the vote
Respond to the vote thread with:

    The vote has now closed. The results are:

    Positive Binding Votes: (# of PMC members that +1'ed)

    .. names of all +1 PMC members ..

    Negative Binding Votes: (# of PMC members that -1'ed)

    .. names of all -1 PMC members ..

    The vote has passed.

_Note: list of PMC members: http://people.apache.org/committers-by-project.html#cordova-pmc_

## If the Vote does *not* Pass
* Revert adding of `-dev` on dev branch
* Address the concerns (on dev branch)
* Merge changes into master
* Re-tag release using `git tag -f`
* Add back `-dev`
* Start a new vote

## Otherwise: Merge & Push Master Branch

    for l in $ACTIVE; do ( cd $l; git checkout master ); done
    for l in $ACTIVE; do ( cd $l; v=$(git describe --tags --abbrev=0 dev); git merge $v ); done

    # Sanity check:
    ./cordova-coho/coho repo-status -r plugins -b master
    # Push:
    for l in $ACTIVE; do ( cd $l; git push --tags https://git-wip-us.apache.org/repos/asf/$l.git master); done


## Publish to dist/

    cd cordova-dist
    svn up
    for l in $ACTIVE; do ( svn rm plugins/$l* ); done
    cp ../cordova-dist-dev/$JIRA/* plugins/
    svn add plugins/*
    svn commit -m "$JIRA Published plugins release to dist"

    cd ../cordova-dist-dev
    svn up
    svn rm $JIRA
    svn commit -m "$JIRA Removing release candidates from dist/dev"
    cd ..

Find your release here: https://dist.apache.org/repos/dist/release/cordova/plugins/

## Publish to Plugins Registry

    for l in $ACTIVE; do ( cd $l; git checkout master ); done
    for l in $ACTIVE; do ( cd $l; echo -n "$l: "; plugman publish . ); done

## Post blog Post

    rake build
    svn st
    svn add NEW_FILES_HERE
    svn commit

## Close JIRA Issue
 * Double check that the issue has comments that record the steps you took
 * Mark it as fixed

## Finally:

 * Update *these instructions* if they were missing anything.

