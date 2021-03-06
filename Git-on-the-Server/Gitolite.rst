Gitolite
==========================

Note: the latest copy of this section of the ProGit book is always available within the gitolite documentation. The author would also like to humbly state that, while this section is accurate, and can (and often has) been used to install gitolite without reading any other documentation, it is of necessity not complete, and cannot completely replace the enormous amount of documentation that gitolite comes with.

Git has started to become very popular in corporate environments, which tend to have some additional requirements in terms of access control. Gitolite was originally created to help with those requirements, but it turns out that it's equally useful in the open source world: the Fedora Project controls access to their package management repositories (over 10,000 of them!) using gitolite, and this is probably the largest gitolite installation anywhere too.

Gitolite allows you to specify permissions not just by repository, but also by branch or tag names within each repository. That is, you can specify that certain people (or groups of people) can only push certain "refs" (branches or tags) but not others.

安装
----------------------------

Installing Gitolite is very easy, even if you don't read the extensive documentation that comes with it. You need an account on a Unix server of some kind; various Linux flavours, and Solaris 10, have been tested. You do not need root access, assuming git, perl, and an openssh compatible ssh server are already installed. In the examples below, we will use the gitolite account on a host called gitserver.

Gitolite is somewhat unusual as far as "server" software goes -- access is via ssh, and so every userid on the server is a potential "gitolite host". As a result, there is a notion of "installing" the software itself, and then "setting up" a user as a "gitolite host".

Gitolite has 4 methods of installation. People using Fedora or Debian systems can obtain an RPM or a DEB and install that. People with root access can install it manually. In these two methods, any user on the system can then become a "gitolite host".

People without root access can install it within their own userids. And finally, gitolite can be installed by running a script on the workstation, from a bash shell. (Even the bash that comes with msysgit will do, in case you're wondering.)

We will describe this last method in this article; for the other methods please see the documentation.

You start by obtaining public key based access to your server, so that you can log in from your workstation to the server without getting a password prompt. The following method works on Linux; for other workstation OSs you may have to do this manually. We assume you already had a key pair generated using ssh-keygen::

 $ ssh-copy-id -i ~/.ssh/id_rsa gitolite@gitserver

This will ask you for the password to the gitolite account, and then set up public key access. This is essential for the install script, so check to make sure you can run a command without getting a password prompt::

 $ ssh gitolite@gitserver pwd
 /home/gitolite

Next, you clone Gitolite from the project's main site and run the "easy install" script (the third argument is your name as you would like it to appear in the resulting gitolite-admin repository)::

 $ git clone git://github.com/sitaramc/gitolite
 $ cd gitolite/src
 $ ./gl-easy-install -q gitolite gitserver sitaram

And you're done! Gitolite has now been installed on the server, and you now have a brand new repository called gitolite-admin in the home directory of your workstation. You administer your gitolite setup by making changes to this repository and pushing.

That last command does produce a fair amount of output, which might be interesting to read. Also, the first time you run this, a new keypair is created; you will have to choose a passphrase or hit enter for none. Why a second keypair is needed, and how it is used, is explained in the "ssh troubleshooting" document that comes with Gitolite. (Hey the documentation has to be good for something!)

Repos named gitolite-admin and testing are created on the server by default. If you wish to clone either of these locally (from an account that has SSH console access to the gitolite account via authorized_keys), type:

 $ git clone gitolite:gitolite-admin
 $ git clone gitolite:testing

To clone these same repos from any other account::

 $ git clone gitolite@servername:gitolite-admin
 $ git clone gitolite@servername:testing

自定义安装
-------------------------------------------

While the default, quick, install works for most people, there are some ways to customise the install if you need to. If you omit the -q argument, you get a "verbose" mode install -- detailed information on what the install is doing at each step. The verbose mode also allows you to change certain server-side parameters, such as the location of the actual repositories, by editing an "rc" file that the server uses. This "rc" file is liberally commented so you should be able to make any changes you need quite easily, save it, and continue. This file also contains various settings that you can change to enable or disable some of gitolite's advanced features.

配置文件和存取控制规则
-------------------------------------------------

Once the install is done, you switch to the gitolite-admin repository (placed in your HOME directory) and poke around to see what you got::

 $ cd ~/gitolite-admin/
 $ ls
 conf/  keydir/
 $ find conf keydir -type f
 conf/gitolite.conf
 keydir/sitaram.pub
 $ cat conf/gitolite.conf
 #gitolite conf
 # please see conf/example.conf for details on syntax and features
 
 repo gitolite-admin
     RW+                 = sitaram
 
 repo testing
     RW+                 = @all

Notice that "sitaram" (the last argument in the gl-easy-install command you gave earlier) has read-write permissions on the gitolite-admin repository as well as a public key file of the same name.

The config file syntax for gitolite is liberally documented in conf/example.conf, so we'll only mention some highlights here.

You can group users or repos for convenience. The group names are just like macros; when defining them, it doesn't even matter whether they are projects or users; that distinction is only made when you use the "macro"::

 @oss_repos      = linux perl rakudo git gitolite
 @secret_repos   = fenestra pear
 
 @admins         = scott     # Adams, not Chacon, sorry :)
 @interns        = ashok     # get the spelling right, Scott!
 @engineers      = sitaram dilbert wally alice
 @staff          = @admins @engineers @interns

You can control permissions at the "ref" level. In the following example, interns can only push the "int" branch. Engineers can push any branch whose name starts with "eng-", and tags that start with "rc" followed by a digit. And the admins can do anything (including rewind) to any ref::

 repo @oss_repos
     RW  int$                = @interns
     RW  eng-                = @engineers
     RW  refs/tags/rc[0-9]   = @engineers
     RW+                     = @admins

The expression after the RW or RW+ is a regular expression (regex) that the refname (ref) being pushed is matched against. So we call it a "refex"! Of course, a refex can be far more powerful than shown here, so don't overdo it if you're not comfortable with perl regexes.

Also, as you probably guessed, Gitolite prefixes refs/heads/ as a syntactic convenience if the refex does not begin with refs/.

An important feature of the config file's syntax is that all the rules for a repository need not be in one place. You can keep all the common stuff together, like the rules for all oss_repos shown above, then add specific rules for specific cases later on, like so::

 repo gitolite
     RW+                     = sitaram

That rule will just get added to the ruleset for the gitolite repository.

At this point you might be wondering how the access control rules are actually applied, so let's go over that briefly.

There are two levels of access control in gitolite. The first is at the repository level; if you have read (or write) access to any ref in the repository, then you have read (or write) access to the repository.

The second level, applicable only to "write" access, is by branch or tag within a repository. The username, the access being attempted (W or +), and the refname being updated are known. The access rules are checked in order of appearance in the config file, looking for a match for this combination (but remember that the refname is regex-matched, not merely string-matched). If a match is found, the push succeeds. A fallthrough results in access being denied.

带"deny"规则高级访问控制
--------------------------------------------------

So far, we've only seen permissions to be one or R, RW, or RW+. However, gitolite allows another permission: -, standing for "deny". This gives you a lot more power, at the expense of some complexity, because now fallthrough is not the only way for access to be denied, so the order of the rules now matters!

Let us say, in the situation above, we want engineers to be able to rewind any branch except master and integ. Here's how to do that::

     RW  master integ    = @engineers
     -   master integ    = @engineers
     RW+                 = @engineers

Again, you simply follow the rules top down until you hit a match for your access mode, or a deny. Non-rewind push to master or integ is allowed by the first rule. A rewind push to those refs does not match the first rule, drops down to the second, and is therefore denied. Any push (rewind or non-rewind) to refs other than master or integ won't match the first two rules anyway, and the third rule allows it.

Restricting pushes by files changed
---------------------------------------------------

In addition to restricting what branches a user can push changes to, you can also restrict what files they are allowed to touch. For example, perhaps the Makefile (or some other program) is really not supposed to be changed by just anyone, because a lot of things depend on it or would break if the changes are not done just right. You can tell gitolite::

 repo foo
     RW                  =   @junior_devs @senior_devs 
 
     RW  NAME/           =   @senior_devs
     -   NAME/Makefile   =   @junior_devs
     RW  NAME/           =   @junior_devs

This powerful feature is documented in conf/example.conf.

个人分支
----------------------------------

Gitolite also has a feature called "personal branches" (or rather, "personal branch namespace") that can be very useful in a corporate environment.

A lot of code exchange in the git world happens by "please pull" requests. In a corporate environment, however, unauthenticated access is a no-no, and a developer workstation cannot do authentication, so you have to push to the central server and ask someone to pull from there.

This would normally cause the same branch name clutter as in a centralised VCS, plus setting up permissions for this becomes a chore for the admin.

Gitolite lets you define a "personal" or "scratch" namespace prefix for each developer (for example, refs/personal/<devname>/*); see the "personal branches" section in doc/3-faq-tips-etc.mkd for details.

"Wildcard（通配符）"库
--------------------------------------

Gitolite allows you to specify repositories with wildcards (actually perl regexes), like, for example assignments/s[0-9][0-9]/a[0-9][0-9], to pick a random example. This is a very powerful feature, which has to be enabled by setting $GL_WILDREPOS = 1; in the rc file. It allows you to assign a new permission mode ("C") which allows users to create repositories based on such wild cards, automatically assigns ownership to the specific user who created it, allows him/her to hand out R and RW permissions to other users to collaborate, etc. This feature is documented in doc/4-wildcard-repositories.mkd.

其他功能
----------------------------

We'll round off this discussion with a sampling of other features, all of which, and many more, are described in great detail in the "faqs, tips, etc" and other documents.

Logging: Gitolite logs all successful accesses. If you were somewhat relaxed about giving people rewind permissions (RW+) and some kid blew away "master", the log file is a life saver, in terms of easily and quickly finding the SHA that got hosed.

Git outside normal PATH: One extremely useful convenience feature in gitolite is support for git installed outside the normal $PATH (this is more common than you think; some corporate environments or even some hosting providers refuse to install things system-wide and you end up putting them in your own directories). Normally, you are forced to make the client-side git aware of this non-standard location of the git binaries in some way. With gitolite, just choose a verbose install and set $GIT_PATH in the "rc" files. No client-side changes are required after that :-)

Access rights reporting: Another convenient feature is what happens when you try and just ssh to the server. Gitolite shows you what repos you have access to, and what that access may be. Here's an example::

     hello sitaram, the gitolite version here is v1.5.4-19-ga3397d4
     the gitolite config gives you the following access:
          R     anu-wsd
          R     entrans
          R  W  git-notes
          R  W  gitolite
          R  W  gitolite-admin
          R     indic_web_input
          R     shreelipi_converter

Delegation: For really large installations, you can delegate responsibility for groups of repositories to various people and have them manage those pieces independently. This reduces the load on the main admin, and makes him less of a bottleneck. This feature has its own documentation file in the doc/ directory.

Gitweb support: Gitolite supports gitweb in several ways. You can specify which repos are visible via gitweb. You can set the "owner" and "description" for gitweb from the gitolite config file. Gitweb has a mechanism for you to implement access control based on HTTP authentication, so you can make it use the "compiled" config file that gitolite produces, which means the same access control rules (for read access) apply for gitweb and gitolite.

Mirroring: Gitolite can help you maintain multiple mirrors, and switch between them easily if the primary server goes down.