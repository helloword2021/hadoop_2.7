<!---
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

HDFS Permissions Guide
======================

* [HDFS Permissions Guide](#HDFS_Permissions_Guide)
    * [Overview](#Overview)
    * [User Identity](#User_Identity)
    * [Group Mapping](#Group_Mapping)
    * [Understanding the Implementation](#Understanding_the_Implementation)
    * [Changes to the File System API](#Changes_to_the_File_System_API)
    * [Changes to the Application Shell](#Changes_to_the_Application_Shell)
    * [The Super-User](#The_Super-User)
    * [The Web Server](#The_Web_Server)
    * [ACLs (Access Control Lists)](#ACLs_Access_Control_Lists)
    * [ACLs File System API](#ACLs_File_System_API)
    * [ACLs Shell Commands](#ACLs_Shell_Commands)
    * [Configuration Parameters](#Configuration_Parameters)

Overview
--------

The Hadoop Distributed File System (HDFS) implements a permissions model for files and directories that shares much of the POSIX model. Each file and directory is associated with an owner and a group. The file or directory has separate permissions for the user that is the owner, for other users that are members of the group, and for all other users. For files, the r permission is required to read the file, and the w permission is required to write or append to the file. For directories, the r permission is required to list the contents of the directory, the w permission is required to create or delete files or directories, and the x permission is required to access a child of the directory.

In contrast to the POSIX model, there are no setuid or setgid bits for files as there is no notion of executable files. For directories, there are no setuid or setgid bits directory as a simplification. The Sticky bit can be set on directories, preventing anyone except the superuser, directory owner or file owner from deleting or moving the files within the directory. Setting the sticky bit for a file has no effect. Collectively, the permissions of a file or directory are its mode. In general, Unix customs for representing and displaying modes will be used, including the use of octal numbers in this description. When a file or directory is created, its owner is the user identity of the client process, and its group is the group of the parent directory (the BSD rule).

HDFS also provides optional support for POSIX ACLs (Access Control Lists) to augment file permissions with finer-grained rules for specific named users or named groups. ACLs are discussed in greater detail later in this document.

Each client process that accesses HDFS has a two-part identity composed of the user name, and groups list. Whenever HDFS must do a permissions check for a file or directory foo accessed by a client process,

* If the user name matches the owner of foo, then the owner permissions are tested;
* Else if the group of foo matches any of member of the groups list, then the group permissions are tested;
* Otherwise the other permissions of foo are tested.

If a permissions check fails, the client operation fails.

User Identity
-------------

As of Hadoop 0.22, Hadoop supports two different modes of operation to determine the user's identity, specified by the hadoop.security.authentication property:

*   **simple**

    In this mode of operation, the identity of a client process is
    determined by the host operating system. On Unix-like systems,
    the user name is the equivalent of \`whoami\`.

*   **kerberos**

    In Kerberized operation, the identity of a client process is
    determined by its Kerberos credentials. For example, in a
    Kerberized environment, a user may use the kinit utility to
    obtain a Kerberos ticket-granting-ticket (TGT) and use klist to
    determine their current principal. When mapping a Kerberos
    principal to an HDFS username, all components except for the
    primary are dropped. For example, a principal
    todd/foobar@CORP.COMPANY.COM will act as the simple username todd on HDFS.

Regardless of the mode of operation, the user identity mechanism is extrinsic to HDFS itself. There is no provision within HDFS for creating user identities, establishing groups, or processing user credentials.

Group Mapping
-------------

Once a username has been determined as described above, the list of groups is determined by a group mapping service, configured by the hadoop.security.group.mapping property. The default implementation, org.apache.hadoop.security.JniBasedUnixGroupsMappingWithFallback, will determine if the Java Native Interface (JNI) is available. If JNI is available, the implementation will use the API within hadoop to resolve a list of groups for a user. If JNI is not available then the shell implementation, org.apache.hadoop.security.ShellBasedUnixGroupsMapping, is used. This implementation shells out with the `bash -c groups` command (for a Linux/Unix environment) or the `net group` command (for a Windows environment) to resolve a list of groups for a user.

An alternate implementation, which connects directly to an LDAP server to resolve the list of groups, is available via org.apache.hadoop.security.LdapGroupsMapping. However, this provider should only be used if the required groups reside exclusively in LDAP, and are not materialized on the Unix servers. More information on configuring the group mapping service is available in the Javadocs.

For HDFS, the mapping of users to groups is performed on the NameNode. Thus, the host system configuration of the NameNode determines the group mappings for the users.

Note that HDFS stores the user and group of a file or directory as strings; there is no conversion from user and group identity numbers as is conventional in Unix.

Understanding the Implementation
--------------------------------

Each file or directory operation passes the full path name to the name node, and the permissions checks are applied along the path for each operation. The client framework will implicitly associate the user identity with the connection to the name node, reducing the need for changes to the existing client API. It has always been the case that when one operation on a file succeeds, the operation might fail when repeated because the file, or some directory on the path, no longer exists. For instance, when the client first begins reading a file, it makes a first request to the name node to discover the location of the first blocks of the file. A second request made to find additional blocks may fail. On the other hand, deleting a file does not revoke access by a client that already knows the blocks of the file. With the addition of permissions, a client's access to a file may be withdrawn between requests. Again, changing permissions does not revoke the access of a client that already knows the file's blocks.

Changes to the File System API
------------------------------

All methods that use a path parameter will throw `AccessControlException` if permission checking fails.

New methods:

* `public FSDataOutputStream create(Path f, FsPermission permission, boolean overwrite, int bufferSize, short replication, long
  blockSize, Progressable progress) throws IOException;`
* `public boolean mkdirs(Path f, FsPermission permission) throws IOException;`
* `public void setPermission(Path p, FsPermission permission) throws IOException;`
* `public void setOwner(Path p, String username, String groupname) throws IOException;`
* `public FileStatus getFileStatus(Path f) throws IOException;`will additionally return the user, group and mode associated with the path.

The mode of a new file or directory is restricted my the umask set as a configuration parameter. When the existing `create(path, ???)` method (without the permission parameter) is used, the mode of the new file is `0666 & ^umask`. When the new `create(path, permission, ???)` method (with the permission parameter P) is used, the mode of the new file is `P & ^umask & 0666`. When a new directory is created with the existing `mkdirs(path)` method (without the permission parameter), the mode of the new directory is `0777 & ^umask`. When the new `mkdirs(path, permission)` method (with the permission parameter P) is used, the mode of new directory is `P & ^umask & 0777`.

Changes to the Application Shell
--------------------------------

New operations:

*   `chmod [-R] mode file ...`

    Only the owner of a file or the super-user is permitted to change the mode of a file.

*   `chgrp [-R] group file ...`

    The user invoking chgrp must belong to the specified group and be
    the owner of the file, or be the super-user.

*   `chown [-R] [owner][:[group]] file ...`

    The owner of a file may only be altered by a super-user.

*   `ls file ...`

*   `lsr file ...`

    The output is reformatted to display the owner, group and mode.

The Super-User
--------------

The super-user is the user with the same identity as name node process itself. Loosely, if you started the name node, then you are the super-user. The super-user can do anything in that permissions checks never fail for the super-user. There is no persistent notion of who was the super-user; when the name node is started the process identity determines who is the super-user for now. The HDFS super-user does not have to be the super-user of the name node host, nor is it necessary that all clusters have the same super-user. Also, an experimenter running HDFS on a personal workstation, conveniently becomes that installation's super-user without any configuration.

In addition, the administrator my identify a distinguished group using a configuration parameter. If set, members of this group are also super-users.

The Web Server
--------------

By default, the identity of the web server is a configuration parameter. That is, the name node has no notion of the identity of the real user, but the web server behaves as if it has the identity (user and groups) of a user chosen by the administrator. Unless the chosen identity matches the super-user, parts of the name space may be inaccessible to the web server.

ACLs (Access Control Lists)
---------------------------

In addition to the traditional POSIX permissions model, HDFS also supports POSIX ACLs (Access Control Lists). ACLs are useful for implementing permission requirements that differ from the natural organizational hierarchy of users and groups. An ACL provides a way to set different permissions for specific named users or named groups, not only the file's owner and the file's group.

By default, support for ACLs is disabled, and the NameNode disallows creation of ACLs. To enable support for ACLs, set `dfs.namenode.acls.enabled` to true in the NameNode configuration.

An ACL consists of a set of ACL entries. Each ACL entry names a specific user or group and grants or denies read, write and execute permissions for that specific user or group. For example:

       user::rw-
       user:bruce:rwx                  #effective:r--
       group::r-x                      #effective:r--
       group:sales:rwx                 #effective:r--
       mask::r--
       other::r--

ACL entries consist of a type, an optional name and a permission string. For display purposes, ':' is used as the delimiter between each field. In this example ACL, the file owner has read-write access, the file group has read-execute access and others have read access. So far, this is equivalent to setting the file's permission bits to 654.

Additionally, there are 2 extended ACL entries for the named user bruce and the named group sales, both granted full access. The mask is a special ACL entry that filters the permissions granted to all named user entries and named group entries, and also the unnamed group entry. In the example, the mask has only read permissions, and we can see that the effective permissions of several ACL entries have been filtered accordingly.

Every ACL must have a mask. If the user doesn't supply a mask while setting an ACL, then a mask is inserted automatically by calculating the union of permissions on all entries that would be filtered by the mask.

Running `chmod` on a file that has an ACL actually changes the permissions of the mask. Since the mask acts as a filter, this effectively constrains the permissions of all extended ACL entries instead of changing just the group entry and possibly missing other extended ACL entries.

The model also differentiates between an "access ACL", which defines the rules to enforce during permission checks, and a "default ACL", which defines the ACL entries that new child files or sub-directories receive automatically during creation. For example:

       user::rwx
       group::r-x
       other::r-x
       default:user::rwx
       default:user:bruce:rwx          #effective:r-x
       default:group::r-x
       default:group:sales:rwx         #effective:r-x
       default:mask::r-x
       default:other::r-x

Only directories may have a default ACL. When a new file or sub-directory is created, it automatically copies the default ACL of its parent into its own access ACL. A new sub-directory also copies it to its own default ACL. In this way, the default ACL will be copied down through arbitrarily deep levels of the file system tree as new sub-directories get created.

The exact permission values in the new child's access ACL are subject to filtering by the mode parameter. Considering the default umask of 022, this is typically 755 for new directories and 644 for new files. The mode parameter filters the copied permission values for the unnamed user (file owner), the mask and other. Using this particular example ACL, and creating a new sub-directory with 755 for the mode, this mode filtering has no effect on the final result. However, if we consider creation of a file with 644 for the mode, then mode filtering causes the new file's ACL to receive read-write for the unnamed user (file owner), read for the mask and read for others. This mask also means that effective permissions for named user bruce and named group sales are only read.

Note that the copy occurs at time of creation of the new file or sub-directory. Subsequent changes to the parent's default ACL do not change existing children.

The default ACL must have all minimum required ACL entries, including the unnamed user (file owner), unnamed group (file group) and other entries. If the user doesn't supply one of these entries while setting a default ACL, then the entries are inserted automatically by copying the corresponding permissions from the access ACL, or permission bits if there is no access ACL. The default ACL also must have mask. As described above, if the mask is unspecified, then a mask is inserted automatically by calculating the union of permissions on all entries that would be filtered by the mask.

When considering a file that has an ACL, the algorithm for permission checks changes to:

* If the user name matches the owner of file, then the owner
  permissions are tested;

* Else if the user name matches the name in one of the named user entries,
  then these permissions are tested, filtered by the mask permissions;

* Else if the group of file matches any member of the groups list,
  and if these permissions filtered by the mask grant access, then these
  permissions are used;

* Else if there is a named group entry matching a member of the groups list,
  and if these permissions filtered by the mask grant access, then these
  permissions are used;

* Else if the file group or any named group entry matches a member of the
  groups list, but access was not granted by any of those permissions, then
  access is denied;

* Otherwise the other permissions of file are tested.

Best practice is to rely on traditional permission bits to implement most permission requirements, and define a smaller number of ACLs to augment the permission bits with a few exceptional rules. A file with an ACL incurs an additional cost in memory in the NameNode compared to a file that has only permission bits.

ACLs File System API
--------------------

New methods:

* `public void modifyAclEntries(Path path, List<AclEntry> aclSpec) throws IOException;`
* `public void removeAclEntries(Path path, List<AclEntry> aclSpec) throws IOException;`
* `public void public void removeDefaultAcl(Path path) throws IOException;`
* `public void removeAcl(Path path) throws IOException;`
* `public void setAcl(Path path, List<AclEntry> aclSpec) throws IOException;`
* `public AclStatus getAclStatus(Path path) throws IOException;`

ACLs Shell Commands
-------------------

*   `hdfs dfs -getfacl [-R] <path>`

    Displays the Access Control Lists (ACLs) of files and directories. If a
    directory has a default ACL, then getfacl also displays the default ACL.

*   `hdfs dfs -setfacl [-R] [-b |-k -m |-x <acl_spec> <path>] |[--set <acl_spec> <path>] `

    Sets Access Control Lists (ACLs) of files and directories.

*   `hdfs dfs -ls <args>`

    The output of `ls` will append a '+' character to the permissions
    string of any file or directory that has an ACL.

    See the [File System Shell](../hadoop-common/FileSystemShell.html)
    documentation for full coverage of these commands.

Configuration Parameters
------------------------

*   `dfs.permissions.enabled = true`

    If yes use the permissions system as described here. If no,
    permission checking is turned off, but all other behavior is
    unchanged. Switching from one parameter value to the other does not
    change the mode, owner or group of files or directories.
    Regardless of whether permissions are on or off, chmod, chgrp, chown and
    setfacl always check permissions. These functions are only useful in
    the permissions context, and so there is no backwards compatibility
    issue. Furthermore, this allows administrators to reliably set owners and permissions
    in advance of turning on regular permissions checking.

*   `dfs.web.ugi = webuser,webgroup`

    The user name to be used by the web server. Setting this to the
    name of the super-user allows any web client to see everything.
    Changing this to an otherwise unused identity allows web clients to
    see only those things visible using "other" permissions. Additional
    groups may be added to the comma-separated list.

*   `dfs.permissions.superusergroup = supergroup`

    The name of the group of super-users.

*   `fs.permissions.umask-mode = 0022`

    The umask used when creating files and directories. For
    configuration files, the decimal value 18 may be used.

*   `dfs.cluster.administrators = ACL-for-admins`

    The administrators for the cluster specified as an ACL. This
    controls who can access the default servlets, etc. in the HDFS.

*   `dfs.namenode.acls.enabled = true`

    Set to true to enable support for HDFS ACLs (Access Control Lists). By
    default, ACLs are disabled. When ACLs are disabled, the NameNode rejects
    all attempts to set an ACL.


