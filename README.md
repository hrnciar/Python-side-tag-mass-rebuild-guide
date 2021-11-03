# Python-side-tag-mass-rebuild-guide

This document tries to document the process of rebuilding Python packages in Fedora rawhide with the next major version of Python (in this case 3.10) in Fedora side-tag.

## Prerequisities

### Fork repos with scrips

For Python 3.10 mass rebuild we were using @hrnciar's forks ([hrnciar/mini-mass-rebuild](https://github.com/hrnciar/mini-mass-rebuild/tree/python3.10) & [hrnciar/rpm-list-builder](https://github.com/hrnciar/rpm-list-builder/tree/python310)) of @hroncok's repos (shout-out to Miro, his scripts were super useful). Don't forget to switch to `python3.10` branch for latest data. 

### Be a proven packager or have one at your side
For a successful rebuild it is necessary to bump the release number of each package so it does not conflict with the latest version in rawhide. 

### Packages in Python bootstrap sequence build with the upcoming version of Python
Python packages are interdependent. For a successful rebuild, we have to start with a bootstrapped Python package and slowly build other packages upon it. We have [a sequence](https://github.com/hrnciar/rpm-list-builder/blob/python310/python310.yaml) with bootstrap options that need to be followed from the top to the bottom of the file. This list is changing, so be ready to rearrange the order as needed. Don't forget to remove packages that are not needed anymore. It's important to ensure that all packages in this sequence build with the upcoming version of Python. Otherwise, it can slow down the whole process, you cannot move on if the package fails (except the ones with a comment that they are non-blocking). To avoid this, we were reporting issues with said packages to the downstream and the upstream maintainers. Some responded, some not. There are three things you can do about it. Wait/urge them to fix it, fix it yourself or write PR to disable failing tests (proven packager can merge them right away without waiting for a response from maintainers).

Beware, this does not ensure everything will work smoothly. With every Python release or package update, there is a chance of potential breakage.


### 2nd beta is out
We were waiting for 2nd beta of Python 3.10 to start the rebuild. The 1st beta is the last release that can officially break things. CPython developers are usually pushing plenty of last-minute changes into 1st beta, so it is safer to wait for the 2nd one. 

### Request fXX-python side tag
Approximately 2 weeks before the planned start of rebuild ask relengs to create a named site tag. You can [get inspiration](https://pagure.io/releng/issue/10115) from the one we used. 

## Side tag mass rebuild
When the 2nd beta of the upcoming Python is out, do Python update [as usually](https://hackmd.io/9f64YNIZTCy0ZzKb5wKtqQ?view#Tracking-patches). Don't forget to update `python3-docs` package as well. 

You can try to do scratchbuild in the side tag using this command:
    
    fedpkg build --fail-fast --target=f35-python --scratch --nowait --srpm
    
To rebuild a package in the side tag one must commit release bump into dist git. Some packages need to be build more than one time. Thus we were doing the first commit "Bootstrap for Python 3.10" that typically disabled some part of dependencies (eg. test or documentation dependencies.). This followed by a second one such as "Rebuilt for Python 3.10" which again enabled them. In most cases just one commit with a release bump was sufficient.

To avoid mistakes we were using [dirty-rebuild-script.py](https://github.com/hrnciar/rpm-list-builder/blob/python310/dirty-rebuild-script.py) to generate a sequence (`build.sh`) that takes care of all necessary steps. It loads [python310.yaml](https://github.com/hrnciar/rpm-list-builder/blob/python310/python310.yaml), clones package repository and modifies spec file if any bconds are present in YAML file and pushes changes into dist git (proven packager needed). In the case of bconds, a patch is created to remind that it needs to be reverted and rebuilt again in future. We suggest to uncomment only part of [python310.yaml](https://github.com/hrnciar/rpm-list-builder/blob/python310/python310.yaml) so constructed `build.sh` is smaller. You can run more scripts in parallel if you believe packages in them are not depending on each other. Pay attention to the comments in the YAML file, special needs of some packages are noted there. Once all packages from the YAML file were rebuilt with Python 3.10 make sure all commits introducing bconds were removed, if so we can try to mass rebuild the rest of the Python packages. Also inform maintainers (devel-announce, devel, python-devel) that mass rebuild started.
~~~~
https://lists.fedoraproject.org/archives/list/devel@lists.fedoraproject.org/message/G47SGOYIQLRDTWGOSLSWERZSSHXDEDH5/
~~~~

Add Python side tag as a new repository into your system so it can be used in repoqueries.

    $ cat /etc/yum.repos.d/koji-python3.10.repo
    [koji-python3.10]
    name=koji-python3.10
    baseurl=http://kojipkgs.fedoraproject.org/repos/f35-python/latest/$basearch/
    enabled=0

Command to get a list of packages built with Python 3.10.

    $ repoquery --refresh --repo=koji-python3.10 --source --whatrequires 'libpython3.10.so.1.0()(64bit)' --whatrequires 'python(abi) = 3.10' --whatrequires 'python3.10dist(*)' | pkgname | env LANG=en_US.utf-8 sort | uniq > python310.pkgs
    
Command to get a list of packages built with Python 3.9.
    
    repoquery --refresh --repo=koji --source --whatrequires 'libpython3.9.so.1.0()(64bit)' --whatrequires 'python(abi) = 3.9' --whatrequires 'python3.9dist(*)' | pkgname | env LANG=en_US.utf-8 sort | uniq > python39.pkgs

 And command to create the todo list of remaining ones:

    $ repoquery --refresh --repo=koji --source --whatrequires 'libpython3.9.so.1.0()(64bit)' --whatrequires 'python(abi) = 3.9' --whatrequires 'python3.9dist(*)' | pkgname | env LANG=en_US.utf-8 sort | uniq > python39.pkgs && repoquery --refresh --repo=koji-python3.10 --source --whatrequires 'libpython3.10.so.1.0()(64bit)' --whatrequires 'python(abi) = 3.10' --whatrequires 'python3.10dist(*)' | pkgname | env LANG=en_US.utf-8 sort | uniq > python310.pkgs && env LANG=en_US.utf-8 comm -23 python39.pkgs python310.pkgs | grep -E -v '^(python3\.9)$' > todo.pkgs && wc -l python39.pkgs python310.pkgs todo.pkgs

- pythno3.9 is excluded because it will always be listed

Change directory to your fork of [hrnciar/mini-mass-rebuild](https://github.com/hrnciar/mini-mass-rebuild/tree/python3.10) and run the above command. With prepared `todo.pkgs` create an empty directory where logs will be stored and `cd` into it. From there we will run [fedpkg-build.sh](https://github.com/hrnciar/mini-mass-rebuild/blob/python3.10/fedpkg-build.sh) to rebuild all packages in `todo.pkgs`.

    parallel -j 12 bash ../fedpkg-build.sh -- (cat ../todo.pkgs)

`fedpkg-build.sh` simply clones package and triggers `fedpkg build`. It also logs failures so when the script finishes do `grep 'Could not' *.log`. 

Try to rebuild the package again when you see errors such as:
- Could not execute build: Could not login to
- Could not execute clone: Failed to execute command.

Check spec file, package could have been retired in the meantime.
- Could not execute build: Cannot continue without properly constructed NVR.

Someone probably built a package in rawhide with the same NEVRA. Bump release and built it again.
- Could not execute build: Package X has already been built

We were also sending an email to devel, python-devel and BCC to maintainers such as:
~~~~
https://lists.fedoraproject.org/archives/list/devel@lists.fedoraproject.org/message/K7B6FTUMHACHZXCNMPI36F657AS3Y4VE/
~~~~

You can monitor your active tasks at [this link](https://koji.fedoraproject.org/koji/tasks?start=0&owner=thrnciar&state=active&view=tree&method=all&order=-id), when there is no active task, regenerate `todo.pkgs` and compare the number of remaining packages with the previous run. If it's significantly lower rebuild all packages again. At some point, the number won't decrease anymore and we will have to wait for actual fixes. 

When the side tag was merged, send another email to mailing lists:
~~~~
https://lists.fedoraproject.org/archives/list/devel@lists.fedoraproject.org/message/OHZEB6OSWUNPO4QZZXXOZPSICCPRXW2X/
~~~~

Also [set new](https://src.fedoraproject.org/rpms/python3.10/pull-request/58) main Python and [remove](https://src.fedoraproject.org/rpms/python3.9/pull-request/71) the old one from Fedora CI.

mhronock opened bugzillas for all remaining packages that failed to build with Python 3.10. Using these scripts:

- https://pagure.io/releng/blob/main/f/scripts/ftbfs-fti/follow-policy.py
- https://github.com/hrnciar/mini-mass-rebuild/blob/python3.10/dupe_ftbfs_fti_bugzillas.ipynb

With [monitor_check.py](https://github.com/hrnciar/mini-mass-rebuild/blob/7a9a7c2ab4f7725d006643e90ef8e74288c2ec1d/monitor_check.py#L195) you can detect missing dependencies and set Bugzilla blockers.

With merged side tag use this command to generate `todo.pkgs`:
~~~~
repoquery --refresh --repo=koji --source --whatrequires 'libpython3.9.so.1.0()(64bit)' --whatrequires 'python(abi) = 3.9' --whatrequires 'python3.9dist(*)' | pkgname | env LANG=en_US.utf-8 sort | uniq > python39.pkgs && repoquery --repo=koji --source --whatrequires 'libpython3.10.so.1.0()(64bit)' --whatrequires 'python(abi) = 3.10' --whatrequires 'python3.10dist(*)' | pkgname | env LANG=en_US.utf-8 sort | uniq > python310.pkgs && env LANG=en_US.utf-8 comm -23 python39.pkgs python310.pkgs | grep -E -v '^python3\.9$' > todo.pkgs && wc -l python39.pkgs python310.pkgs todo.pkgs
~~~~

At the very last, disable webhook rebuilds in COPR - https://pagure.io/copr/copr/issue/1972.
