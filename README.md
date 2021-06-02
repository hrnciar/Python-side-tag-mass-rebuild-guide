# Python-side-tag-mass-rebuild-guide

This document tries to document the process of rebuilding Python packages in Fedora rawhide with the next major version of Python (in this case 3.10) in Fedora side-tag.

## Prerequisities

### Packages in Python bootstrap sequence build with the upcoming version of Python
Python packages are interdependent. For a successful rebuild, we have to start with a bootstrapped Python package and slowly build other packages upon it. We have a sequence with bootstrap options that need to be followed from the top to the bottom of the file. It's important to ensure that all packages in this sequence build with the upcoming version of Python. Otherwise, it can slow down the whole process, you cannot move on if the package fails (except the ones with a comment that they are non-blocking). To avoid this, we were reporting issues with said packages to the downstream and the upstream maintainers. Some responded, some not. There are basically three things you can do about it. Wait/urge them to fix it, fix it yourself or write PR to disable failing tests.


### 2nd beta is out
We were waiting for 2nd beta of Python 3.10 to start the rebuild. CPython developers are usually pushing plenty of last-minute changes into 1st beta, thus it is safer to wait for the 2nd one. 