### The RefSpectacular flow
Whats the main goal of Source Control, to protect and share your code, to retain a history on your "source".

Main points of refspec push flow.

1. Controlled access to the baseline, we put robust Peer Review process in for merge into Trunk. This ensures Code going into pipeline is stable and ready to go to production. Potentially every Trunk merge could become next release.
2. The Trunk MUST always be in working order.
3. Trunk (origin/develop) is the only branch that we care about history, its the only branch with infinite lifespan. This means all other branches become labels.
4. We tag ONLY signed off releases (origin/release/x.xx.x).
5. We cut hotfixes from last released Tag.
6. All release fixes are cut from release, they are PR'd into release and rebased off develop, and PR'd back into develop. This ensures Code is never missed and no need for tag merge into trunk!


### The Main branches

| branch | purpose |
|--------|---------|
| develop | The main trunk and feature branch of our codebase. HEAD of this branch is always deployed to the develop environment. This is done with circle ci. |
| acceptance | This branch controls what QA is currently working on. It should be treated, not so much as it's own branch, but as a reference to the commit in develop (trunk) that is being tested |
| staging | This branch is used to pre-deploy the release to a pre-production (staging) environment. This is used for regression testing, automation runs, hot fixes and a chance to test the deploy itself. |
| performance | This branch is used to test performance in a performance environment, typically in alignment with acceptance but can point to anything. |
| production | The code that is currently deployed to Production. |


### Supporting branches

| branch | purpose |
|--------|---------|
| feature | These branches are used to develop new code, this could be a feature/bug/chore/task whatever work type your project supports. These branches are merged into develop via strict peer review process. |
| release | These branches are used to release software. These are typically a sprints worth of work but can be many sprints depending on your project. We follow a major.minor.patch flow. The release version should reflect what size of release it is. Weekly releases are merely minor version bumps. |
| hotfix | These branches are used to release burning issues, typically stemming from the last release. |

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
  <img src="RefSpectacular/images/Step 3.png" width="350"/>
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
  <img src="https://github.com/JNKRMN/RefSpectacular/blob/Chore/updating_documentation/images/step%205.png" width="350"/>
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
  <img src="https://github.com/JNKRMN/RefSpectacular/blob/Chore/updating_documentation/images/step%206.7.png" width="350"/>
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
  <img src="https://github.com/JNKRMN/RefSpectacular/blob/Chore/updating_documentation/images/step%208.png" width="350"/>
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
  <img src="https://github.com/JNKRMN/RefSpectacular/blob/Chore/updating_documentation/images/step%209.png" width="350"/>
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
  <img src="https://github.com/JNKRMN/RefSpectacular/blob/Chore/updating_documentation/images/Hotfix.png" width="350"/>
</p>



### Peer Review
We put robust peer review in place.

1. Build in CI Tool (CircleCI).
2. Two code reviewers.
3. Unit tests.
4. Code coverage tools.
5. Also branch merging in must be up to date with trunk.

We have many stacks/apps and provisioning/deployment tools. Finding a little sanity in the way we manage our code is key. Watching shas progress in the RefSpec flow, is something to see. Your Network graphs will clean up, it will be easy to deploy and manage environments. The majority of the manual steps can be automated.
