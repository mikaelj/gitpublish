
==================
The Gitpublish Design
==================

A Git-like Interface for Publishing
-----------------------------------

Git provides a clean model for "pulling" work from other people
and places, and "pushing" to other repositories to publish your
work.  I would like to use this same basic interface, for extracting
text from other sources (e.g. MS Word, wikis like MoinMoin etc.)
and for publishing content to external channels like WordPress.
This would allow us to manage and edit our content under git
with its powerful branch, merge and history tools, while publishing
content to the world via best-of-breed tools like WordPress.  
Many publishing tools try to provide a "complete solution",
but are only mediocre at each specific function.  I
want the freedom to use the best tool for each distinct function:
git for version control; WordPress for publishing; Restructured
Text (and all its associated tools like docutils and Sphinx)
for cross-compiling my content to any desired target format.

Instead of expounding on Git's many wonderful capabilities, I
want to zero in on one specific feature: git *remotes*::

   git remote add upstream git://github.com/cjlee112/pygr.git

creates a new *remote* called ``upstream``, which points to a
git repository at github.com.  We can then immediately fetch
all the latest updates from that repository using::

   git fetch upstream

They show up as *remote-tracking branches* in your local repository
with names like ``remotes/upstream/master`` (which tracks the 
``master`` branch in the ``upstream`` repository).  We can view
all the commits on those branches, using tools like ``gitk``
and see exactly what was added or changed.  We can merge
content from a given remote branch into our current branch
as easily as::

   git merge remotes/upstream/master

and I can push content from my master branch to ``upstream`` by typing::

   git push upstream master

I want to do exactly the same thing with remote "containers"
that are *not* git repositories.  For example::

   gitpublish remote add my_wordpress wordpress:thinking.bioinformatics.ucla.edu


creates a new *remote* called ``my_wordpress``, which points to a
WordPress website at ``thinking.bioinformatics.ucla.edu``.  I can 
then push restructured text content from my master branch
to that WordPress blog as simply as::

   gitpublish push my_wordpress master

Of course, this requires some transparent magic to automatically
convert the restructured text to the style of HTML that WordPress uses,
and upload the content via XMLRPC.  Gitpublish is all about automating 
such standard conversion and transmission tasks, and integrating
them in a clean way with the power of git.


The Gitpublish Interface
---------------------

* *add a remote*::

    gitpublish remote add <remotename> <protocol>:<address>

* *fetch* updates from a remote as (new) git branches and commits::

    gitpublish fetch <remotename>

  New commits show up on branches named
  ``gpremotes/<remotename>/<branchname>``.  New files will show up
  by default in a directory called *<remotename>-import*.
  Content will be automatically
  converted to the local format(s) specified by your gitpublish
  config file (i.e. ``.rst``, ``.csv``, ``.opml`` etc.).

* Checkout a remote tracking branch, to prepare content for
  sending to remote repository::

    gitpublish checkout <remotename> [<branchname>]

  This command simply executes::

    git checkout gpremotes/<remotename>/<branchname>

* *add* a local file to be published on a particular remote::

    gitpublish add bigdoc.rst [--pubtype=page ...]

  Note that just like ``git add``, this simply *marks* the file
  for addition to the remote, by adding it to the :class:`DocMap`
  file that maps local files to this remote repository.
  This change won't actually be pushed
  to the remote until you *gitpublish push* (see below).  Specifically, it
  adds a mapping for the designated file ``bigdoc.rst`` to be
  mapped to the designated remote, in this case using a default
  mapping.  For WordPress, a *Post*.  Provide extra arguments to
  give a non-default mapping (e.g. a *Page*).  The default will
  obtain the title etc. from the restructured text document.
  Automatically adds the updated state of the remote mapping
  file to git (via ``git add``).

* *remove* a local file from publication on a particular remote::

    gitpublish rm bigdoc.rst

* *rename* a local file::

    gitpublish mv <oldpath> <newpath>

  This does two things:

    * it runs *git mv <oldpath> <newpath>*.

    * it updates the :class:`DocMap` entry to reflect the local file
      name change.

  Note that just as for *git mv* you must *commit* the change.

* *commit* changes including remote mappings.  Just a proxy for
  regular ``git commit``, which you can use equally well::

    gitpublish commit -m 'a message'

* *push* my changes to the remote (to publish them)::

    gitpublish push my_wordpress [<branchname>]

* *merge* a local branch into your remote tracking branch.  Use this
  to bring in changes in your local documents, as the first step to
  pushing those changes to the remote repository::

    gitpublish merge <localbranch>

  All this does is call *git merge <localbranch>*.  

  Note that you must be on a gitpublish remote tracking branch to 
  run this command (i.e. you must first run *gitpublish checkout*).


The Gitpublish Plug-in API
-----------------------

My goal is to make it easy for anyone to add a new kind of remote
by writing in "plug-in" code that conforms to a standard gitpublish API.
A plug-in can implement several different levels of functionality:

* *pull-only*: enables gitpublish to import content from the remote,
  but not to export content to the remote.

* *push-only*: enables gitpublish to export content to the remote,
  but not to import.

* *push/pull without remote version history*: treats the remote as a "snapshot"
  with no built-in version control.  In other words, the remote
  can be synched with one specific git commit.  This is reasonably
  consistent with how git itself tracks its "branches", which are
  actually just pointers to a specific commit (i.e. the HEAD of
  that branch).

A plug-in should be a python file named ``gitpublish/plugin/myname.py``
(where *myname* is the remote protocol name)
that implements the following class (for an example see
gitpublish/plugin/wordpress.py):


.. class:: Repo(*args, **kwargs)

   Create an instance object that provides an interface to the
   remote repository.  It will be passed keyword arguments.

.. method:: Repo.new_document(doc, gitpubHash=None, *args, **kwargs)

   Save *doc* as a new :class:`Document` in the remote repository in
   branch *branchname*, with optional arguments controlling
   how it should be stored.  Returns the new document's 
   unique ID in the remote repository.

   *gitpubHash* provides a hashcode for the current content state.
   The new_document() method should save it to the remote repository
   if possible, in a form that can be retrieved by get_document().
   This will enable gitpublish to see if any changes have been
   made independently to the document on the remote repository.

.. method:: Repo.list_documents(*args, **kwargs)

   Get a dict of remote document ID (as a string), whose associated value
   should be a dictionary of document attributes (whatever information
   is appropriate from this remote).

.. method:: Repo.get_document(doc_id)

   Get (rest, info) pair, where *rest* is the restructured text of
   the specified document, and *info* is a dictionary of document attributes
   (whatever information is appropriate from this remote).

.. method:: Repo.set_document(doc_id, doc, *args, **kwargs)

   Save the specified :class:`Document` to the specified document ID
   in the remote repository.

For remote repositories that support branches:

.. method:: Repo.list_branches()

   Get a list of branches in the remote repository, as a list of
   string branch names.  Returns *None* if the remote does not support
   multiple branches.

.. method:: Repo.get_branch(branchname)

   Get a branch object for the specified branch name.



.. method:: Branch.list_commits()

   Get a list of all commit objects in this branch in temporal order.

.. method:: Branch.new_commit(changed_docs, *args, **kwargs)

   Create a new remote commit on this branch containing the changed
   documents *changed_docs*, and return the commit ID.

.. method:: Commit.list_documents(changed=False, *args, **kwargs)

   Get a list of document IDs (as strings) contained in this
   commit.  If *changed=True* only return IDs of documents that
   changed in this commit.

.. method:: Commit.get_document(doc_id)

   Get a document object for the specified document ID, reflecting
   its state in this commit.



Gitpublish API Classes
-------------------

.. class:: Document(path)

.. attribute:: Document.id

   the document's unique ID.  For a local document, just its path 
   within the repository.  For a remote document, its remote repository ID.

.. attribute:: Document.title
   
   the document's title

.. attribute:: Document.rest

   get the restructured text of the document as a string.   

.. method:: Document.write(rest_text)

   save the restructured text string to the document file.

.. class:: RemoteMap(remotename, remote)

   Creates an empty map to the specified remote, provided
   as a remote object.  Adds its mapfile to git.

.. method:: RemoteMap.add(doc, *args, **kwargs)

   Add the specified doc object to the map for this repository,
   with optional arguments

   