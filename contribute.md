Contribute
==========

This project is a [GraphX](https://spark.apache.org/graphx/)-based open source effort. All kinds of contributions are always welcome!

Contributing Documentation
--------------------------

The iWCT-GraphX website is constructed with MDwiki and hosted on GitHub. You can just [fork iWCT-GraphX-webpage on GitHub](https://github.com/sjtu-iwct/graphx-webpage), modify the `develop` branch and send a pull request. Please [create an issue](https://github.com/sjtu-iwct/graphx-webpage/issues) whenever the modification is massive. There is no need to submit a CLA.

Contributing Sourcecode
-----------------------

If you are a developer, feel free to [fork iWCT-GraphX-algorithm on GitHub](https://github.com/sjtu-iwct/graphx-algorithm) and send in pull requests. Right now there are no strict coding guidelines, but please follow the basic Scala and C/C++ Language specifications, and annotate the Scala codes in a [Scala-doc style](http://docs.scala-lang.org/style/scaladoc.html). These can make the code more readable and help build the API docs automatically.

For code contributions, submission of a contributors license agreement is **not currently required**. However, the iWCT-GraphX-algorithm project is licensed [GNU General Public License v2](https://github.com/sjtu-iwct/graphx-algorithm/blob/master/LICENSE) now. Any users or developers should follow this license. The CLA might be changed later, then the contributing documentation will be updated.

How to Contribute
-----------------

As we will only keep the stable and well-tested documents and codes in the `master` branch, please remember to fork the `develop` branch instead (and send pull requests to `develop` too).

Note: If you are a newbie to Git, you can follow [a simple Git guide](tutorials/git.md) including several basic git operations.

### Available prefixes

Note: Please add one of the following prefix in your commit message.

* [docs] Fix a typo in README
* [hotfix] Fix a bug
* [feature] Add a new algorithm
* [issue-1234] Fix the bug mentioned in Issue-1234
* Other reasonable prefixes

When a bug or any necessary future work is found, you can post it in [Issues](https://github.com/sjtu-iwct/graphx-algorithm/issues) first and solve them in future.
