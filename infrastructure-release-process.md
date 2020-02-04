---
title: Release process
layout: main
categories: infrastructure
---

This is to document the release process of AliRoot / AliPhysics.

# Creating a release

Creation of a release can be done in several ways, however this is the preferred workflow:

* Create a branch in alidist called `AliPhysics-<aliroot-tag>-01-patches`.
* Replace all the occurences of the old `<aliroot-tag>` with the new ones and commit on the newly created patches branch.
* Create a release in alidist called `AliPhysics-<aliroot-tag>-01` using the `AliPhysics-<aliroot-tag>-01-patches` branch.
* Create a release in AliRoot called `<aliroot-tag>` using your branch of choice.
* Create a release in AliPhysics called `<aliroot-tag>-01` using your branch of choice.

The last step will trigget a build in Jenkins.

<script>
function update() {
  var targetMatcher = /(v[0-9]-[0-9][0-9]-[0-9][0-9])[a-z]*/;
  var release = document.getElementById("release").value;
  var match = release.match(targetMatcher);
  if (match === null) {
    return;
  }
  var target = match[1];
  
  document.getElementById("alidist_target").value = "AliPhysics-" + target + "-01-patches";
  document.getElementById("alidist_tag").value = "AliPhysics-" + release + "-01";
  document.getElementById("alidist_title").value = release;
  document.getElementById("alidist_button").innerHTML = "Create tag for alidist@" + "AliPhysics-" + release + "-01";
  document.getElementById("aliroot_target").value = target + "-patches";
  document.getElementById("aliroot_tag").value = release;
  document.getElementById("aliroot_title").value = release;
  document.getElementById("aliroot_button").innerHTML = "Create tag for AliRoot@" + release;
  document.getElementById("aliphysics_target").value = target + "-01-patches"
  document.getElementById("aliphysics_tag").value = release + "-01";
  document.getElementById("aliphysics_title").value = release + "-01";
  document.getElementById("aliphysics_button").innerHTML = "Create tag for AliPhysics@" + release + "-01";
}
</script>

<form>
  <input id="release" type="text" onkeyup="update()" value="v5-09-XX"></input>
</form>
<form target="_blank">
  <input id="alidist_target" type="hidden" name="target"></input>
  <input id="alidist_tag" type="hidden" name="tag"></input>
  <input id="alidist_title" type="hidden" name="title"></input>
  <button id="alidist_button" type="submit" method="get" target="_blank" formaction="https://github.com/alisw/alidist/releases/new">Create alidist release</button>
</form>
<br/>
<form target="_blank">
  <input id="aliroot_target" type="hidden" name="target"></input>
  <input id="aliroot_tag" type="hidden" name="tag"></input>
  <input id="aliroot_title" type="hidden" name="title"></input>
  <button id="aliroot_button" type="submit" method="get" target="_blank" formaction="https://github.com/alisw/AliRoot/releases/new">Create AliRoot release</button>
</form>
<br/>
<form target="_blank">
  <input id="aliphysics_target" type="hidden" name="target"></input>
  <input id="aliphysics_tag" type="hidden" name="tag"></input>
  <input id="aliphysics_title" type="hidden" name="title"></input>
  <button id="aliphysics_button" type="submit" method="get" target="_blank" formaction="https://github.com/alisw/AliRoot/releases/new">Create AliPhysics release</button>
</form>
<br/>

Once the release is built. You can create a pull request in `alidist` from the branch `AliPhysics-<aliroot-tag>-01-patches` so that master uses a specific release. You might also want to move the dailies to such a release.

# Patching old ( <= AliRoot-v5-08) releases

Around `AliRoot-v5-08` / `AliRoot-v5-09` transition there
was a cleanup of large files, which were removed from the history of the official Github repository and left in a CERN hosted gitlab repository:

https://gitlab.cern.ch/alisw/AliRoot-legacy

For this reason any patch release on top of an old release should be tagged from that repository. In particular you have the following branches:

 * *v5-09-18b-01* (SLC5, alidist: legacy/v5-09-18)
 * *v5-08-13zm-01-cookdedx* (SLC5, alidist: IB/v5-08/prod)
 * *v5-08-13q-p14-01-cookdedx* (SLC5, alidist: IB/v5-08/prod)
 * *v5-08-13q-p14-01* (SLC5, alidist: IB/v5-08/prod)
 
