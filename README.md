### The RefSpectacular flow
Whats the main goal of Source Control, to protect and share your code, to retain a history on your "source".

Main points of refspec push flow.

1. To eliminate managing a million and one merges. What do I mean by that, in typical git flow, you cut a release off of trunk, you test, you merge it into a master or production branch. You might have had a release fix, so now you also need to merge master/production or tag into trunk, to pick up the release fix. Thats a full time job for one or many people depending on how many repos you have. In the refspec world we only care about the history on trunk (develop) branch. All up stream branches are just tags or references.
2. This branching method is ment to mate with a containerized infrastructure. If you think about the container world we build and certify an image and then just pass that golden image/images along our environments. Using git flow in container world is not recommeneded because you could certify a image in acceptance but as soon as the release branch is cut and a new thing is added to that branch it has changed the state, causing re-certification. We this Refspec branching method to pass the golden image/images up through our environments.
3. The Trunk MUST always be in working order.
4. Trunk (origin/develop) is the only branch that we care about history, its the only branch with infinite lifespan. This means all other branches become labels or references.
5. Another reason we keep Trunk in such good order is we consider every commit into trunk the next potential release.  
6. We tag ONLY signed off releases, we currently have tooling that is generating random release names. We have moved past the need for semantic versioning, we release too often for the relevance of a version to matter. Its only use to to anchor commits to release notes.
7. In the rare case you do need a hotfix and cant release the next trunk commits, All release fixes are cut from release, they are PR'd into release and rebased off develop, and PR'd back into develop. This ensures Code is never missed and no need for tag merge into trunk!


### The Main branches

| branch | purpose |
|--------|---------|
| develop | The main trunk of our codebase. HEAD of this branch is always deployed to the acceptance environment. This is done with circle ci. (can be any ci tool, I just like circle) |
| staging | This branch is used to pre-deploy the release to a pre-production (staging) environment. This is used for regression testing, automation runs, hot fixes and a chance to test the deploy itself. |
| performance | This branch is used to test performance in a performance environment, typically in alignment with acceptance but can point to anything. |
| production | The code that is currently deployed to Production. |


### Supporting branches

| branch | purpose |
|--------|---------|
| feature/chore/bug/so on | These branches are used to develop new code, this could be a feature/bug/chore/task whatever work type your project supports. These branches are merged into develop via strict peer review process. |
| release | These branches are used to release software. We use build tooling to auto cut and auto generate the release and release notes. These are typically only a few commits, we try to release often, at least once a day but can be many times in one day.
| hotfix | These branches are used to release burning issues, typically stemming from the last release. But are rare events in todays world, we typically just get commits into head and do a normal release since we keep so close to head. |

### The Magic, what is a refspec push
```sh
git push remote src:dst

example:
git push origin release/3.2.2:production
```
main thing to focus on:
1. src is head hash of your local branch your pushing from.
2. dst is name of the remote thing you want to update.

3. If dst exists it will try to fast forward, if no ff allowed, just force (minus develop branch), we dont care about history on anything but trunk (develop).

```sh
git push origin release/3.2.2:production -f
```

4. If dst doesn't exist then it will create the branch/tag referencing the src hash.

Other great sources of info on refspec
https://git-scm.com/docs/git-push
https://stackoverflow.com/questions/38494546/git-push-what-is-the-difference-between-headrefs-heads-branch-and-branch

### Hi Level flow

**Step 1**.
Create your repo and load your assets, begin development.


**Step 2**.
Cut your branch off develop for feature work

```sh
$> git checkout develop
$> git pull
$> git checkout -b feature/new-feature-jiraproject-1234567
```
write some Code
Submit PR for your code into develop. I wont go over Peer Review guidelines in this flow, I will address it later.


**Step 3**.
Fast forward 5 Develop merges and you finally have something to test. Refspec push develop branch to acceptance branch for testing.

```sh
$> git checkout develop
$> git pull
$> git push origin develop:acceptance
```
Circle triggers an acceptance deployment on successful acceptance branch build.

<p align="center">
  <img src="https://github.com/JNKRMN/RefSpectacular/blob/develop/images/Step%203.png" width="350"/>
</p>


**Step 4**.
Refspec push acceptance branch to performance branch

```sh
$> git checkout acceptance
$> git pull
$> git push origin acceptance:performance
```
Circle triggers an performance deployment on successful performance branch build. Post successfull deploy we auto trigger a performance test.

<p align="center">
  <img src="https://github.com/JNKRMN/RefSpectacular/blob/develop/images/Step%204.png" width="350"/>
</p>

**Step 5**.
We finally have a set of releasable items, lets cut a release branch.

This flow is cut off head
```sh
$> git checkout develop
$> git pull
$> git checkout -b release/1.xx.0
$> git push origin release/x.xx.x
```

This flow is reset back to a specific sha
```sh
$> git checkout develop
$> git pull
$> git checkout -b release/1.xx.0
$> git reset --hard <short sha>
$> git push origin release/x.xx.x
```
Once the release is green in circle, refspec push the release branch to staging branch

```
$> git push origin release/x.xx.x:staging
```

staging branch will deploy to staging environment via circle build

<p align="center">
  <img src="https://github.com/JNKRMN/RefSpectacular/blob/develop/images/step%205.png" width="350"/>
</p>

**Step 6**. cut regression fix branch off of release branch
```sh
$> git checkout release/3.xx.0
$> git pull origin release/3.xx.0
$> git checkout -b bug/fix-show-stopper-1234567

# write some code

$> git push origin bug/fix-show-stopper-1234567

# wait for some Peer Review

$> git checkout release/x.xx.0
$> git pull origin release/x.xx.0

# if any changes were pulled...
$> git checkout bug/fix-show-stopper-1234567
$> git rebase release/x.xx.0
$> git checkout release/x.xx.0

$> git merge --no-ff bug/fix-show-stopper-1234567
$> git push origin release/3.xx.0
```

Once the release is green in circle, refspec push the release branch to staging branch

```
$> git push origin release/x.xx.x:staging
```

staging branch will deploy to staging environment via circle build

<p align="center">
  <img src="https://github.com/JNKRMN/RefSpectacular/blob/develop/images/step%206.7.png" width="350"/>
</p>

**Step 7**. We follow a dual PR flow where release/hotfix regression is put into release/hotfix and develop.
Once your fix is in the release/hotfix, do an interactive rebase on your release fix branch then do a PR into develop.

```sh
$> git pull origin develop
$> git checkout bug/fix-show-stopper-WMMW-123
$> git rebase develop -i
```
Drop any commits that are not part of your fix, and then push your changes. Open a PR against develop


**Step 8**.
Once signed off by all parties, push release branch to production branch, this triggers production deployment.

```sh
$> git checkout release/x.xx.x
$> git pull
$> git push origin release/x.xx.x:production
```
<p align="center">
  <img src="https://github.com/JNKRMN/RefSpectacular/blob/develop/images/step%208.png" width="350"/>
</p>

**Step 9**.
Tag the release in github

Tag version:
x.xx.x

Target:
release/x.xx.x

Title:
x.xx.x

Description:
Intranet link to releasenotes

<p align="center">
  <img src="https://github.com/JNKRMN/RefSpectacular/blob/develop/images/step%209.png" width="350"/>
</p>



### Hotfixes
Hotfixes are Cut from last released Tag. Check for last released Tag.

```sh
git fetch
git tag
git checkout x.xx.x
```

You'll be in a detached head state

```sh
git checkout -b release/x.xx.x
git push origin release/x.xx.x
```

Follow steps 6 - 9 to finish out the Hotfix.

<p align="center">
  <img src="https://github.com/JNKRMN/RefSpectacular/blob/develop/images/Hotfix.png" width="350"/>
</p>

### Rollback
Rollbacks in Refspec are achieved by refspecing last successfully released tag to production branch.  
Check in github to see what the last released tag is. That will be your last good commit before your latest push that broke things. Verify that the last released tag matches the commit on production branch just before you pushed your broken code.

1. ```git checkout production branch; git fetch; git pull;```
2. Reset back to the SHA, its the one you researched, the last good commit.
```git reset --hard ${SHA}```
3. Now ensure your local copy is now showing the SHA you want to be in production, IE the last good commit
```git show-ref production```
4. Now push it up, the reason you have to force is its not a fast forward, in refspec we dont care about branch history past develop branch, so we dont mind overwriting production branch history with the force push.
```git push origin production -f```

This will push the last good code to Production branch and will deploy it Production Environment


### Peer Review
We put robust peer review in place.

1. Build in CI Tool (CircleCI).
2. Two code reviewers.
3. Unit tests.
4. Code coverage tools.
5. Also branch merging in must be up to date with trunk.

We have many stacks/apps and provisioning/deployment tools. Finding a little sanity in the way we manage our code is key. Watching shas progress in the RefSpec flow, is something to see. Your Network graphs will clean up, it will be easy to deploy and manage environments. The majority of the manual steps can be automated.
