#nfs

####Table of Contents

1. [Overview - What is the nfs module?](#overview)
2. [Module Description - What does this module do?](#module-description)
3. [Setup - The basics of getting started with nfs](#setup)
4. [Usage - The class and available configurations](#usage)
5. [Requirements](#requirements)
6. [Limitations - OS compatibility, etc.](#limitations)
7. [Contributing to the graphite module](#contributing)

##Overview

This module installs, configures and manages everything on NFS clients and servers.

Github Master: [![Build Status](https://secure.travis-ci.org/derdanne/puppet-nfs.png?branch=master)](https://travis-ci.org/derdanne/puppet-nfs)

##Module Description

This module can be used to simply mount nfs shares on a client or to configure your nfs servers.
It makes use of storeconfigs on the puppetmaster to get its resources.

##Setup

This Module depends on puppetlabs-stdlib >= 4.5.0 and puppetlabs-concat >= 1.1.2.

Examples
----------------------

### Simple NFSv3 server and client example

This will export /data/folder on the server and automagically mount it on client.

<pre>
  node server {
    class { '::nfs':
      server_enabled => true.
    }
    nfs::server::export{ '/data_folder':
      ensure  => 'mounted',
      clients => '10.0.0.0/24(rw,insecure,async,no_root_squash) localhost(rw)'
  }

  # By default, mounts are mounted in the same folder on the clients as
  # they were exported from on the server
  node client {
    class { '::nfs':
      client_enabled => true,
    }
    Nfs::Client::Mount &lt;&lt;| |&gt;&gt;
  }

</pre>


### NFSv3 multiple exports, servers and multiple node example


<pre>
  node server1 {
    class { '::nfs':
      server_enabled => true
    }
    nfs::server::export{
      '/data_folder':
        ensure  => 'mounted',
        clients => '10.0.0.0/24(rw,insecure,async,no_root_squash) localhost(rw)'
      # exports /homeexports and mounts them om /srv/home on the clients
      '/homeexport':
        ensure  => 'mounted',
        clients => '10.0.0.0/24(rw,insecure,async,root_squash)',
        mount   => '/srv/home'
  }

  node server2 {
    class { '::nfs':
      server_enabled => true
    }
    # ensure is passed to mount, which will make the client not mount it
    # the directory automatically, just add it to fstab
    nfs::server::export{
      '/media_library':
        ensure  => 'present',
        nfstag     => 'media'
        clients => '10.0.0.0/24(rw,insecure,async,no_root_squash) localhost(rw)'
  }

  node client {
    class { '::nfs':
      client_enabled => true,
    }
    Nfs::Client::Mount &lt;&lt;| |&gt;&gt;
  }

  # Using a storeconfig override, to change ensure option, so we mount
  # all shares
  node greedy_client {
    class { '::nfs':
      client_enabled => true,
    }
    Nfs::Client::Mount &lt;&lt;| |&gt;&gt; {
      ensure => 'mounted'
    }
  }


  # only the mount tagged as media
  # also override mount point
  node media_client {
    class { '::nfs':
      client_enabled => true,
    }
    Nfs::Client::Mount &lt;&lt;|nfstag == 'media' | &gt;&gt; {
      ensure => 'mounted',
      mount  => '/import/media'
    }
  }

  # All @@nfs::server::mount storeconfigs can be filtered by parameters
  # Also all parameters can be overridden (not that it's smart to do
  # so).
  # Check out the doc on exported resources for more info:
  # http://docs.puppetlabs.com/guides/exported_resources.html
  node single_server_client {
    class { '::nfs':
      client_enabled => true,
    }
    Nfs::Client::Mount &lt;&lt;| server == 'server1' |&gt;&gt; {
      ensure => 'absent',
    }
  }

</pre>

### NFSv4 Simple example


<pre>

  # We use the $::domain fact for the Domain setting in
  # /etc/idmapd.conf.
  # For NFSv4 to work this has to be equal on servers and clients
  # set it manually if unsure.
  #
  # All nfsv4 exports are bind mounted into /export/$mount_name
  # and mounted on /srv/$mount_name on the client.
  # Both values can be overridden through parameters both globally

  # and on individual nodes.
  node server {
    class { '::nfs':
      server_enabled => true,
      nfs_v4 => true,
      nfs_v4_export_root_clients =>
        '10.0.0.0/24(rw,fsid=root,insecure,no_subtree_check,async,no_root_squash)'
    }
    nfs::server::export{ '/data_folder':
      ensure  => 'mounted',
      clients => '10.0.0.0/24(rw,insecure,no_subtree_check,async,no_root_squash) localhost(rw)'
  }

  # By default, mounts are mounted in the same folder on the clients as
  # they were exported from on the server

  node client {
    class { '::nfs':
      client_enabled => true,
      nfs_v4 => true,
      nfs_v4_export_root_clients =>
        '10.0.0.0/24(rw,fsid=root,insecure,no_subtree_check,async,no_root_squash)'
    }
    Nfs::Client::Mount &lt;&lt;| |&gt;&gt;
  }

  # We can also mount the NFSv4 Root directly through nfs::client::mount::nfsv4::root.
  # By default /srv will be used for as mount point, but can be overriden through
  # the 'mounted' option.

  node client2 {
    $server = 'server'
    class { '::nfs':
      client_enabled => true,
      nfs_v4 = true,
    }
    Nfs::Client::Mount::Nfs_v4::Root &lt;&lt;| server == $server |&gt;&gt; {
      mount => "/srv/$server",
    }
  }

</pre>

### NFSv4 insanely overcomplicated reference example


<pre>

  # and on individual nodes.
  node server {
    class { 'nfs::server':
      nfs_v4              => true,
      # Below are defaults
      nfs_v4_idmap_domain => $::domain,
      nfs_v4_export_root  => '/export',
      # Default access settings of /export root
      nfs_v4_export_root_clients =>
        "*.${::domain}(ro,fsid=root,insecure,no_subtree_check,async,root_squash)"


    }
    nfs::server::export{ '/data_folder':
      # These are the defaults
      ensure  => 'mounted',
      # rbind or bind mounting of folders bindmounted into /export
      # google it
      bind    => 'rbind',
      # everything below here is propogated by to storeconfigs
      # to clients
      #
      # Directory where we want export mounted on client
      mount     => undef,
      remounts  => false,
      atboot    => false,
      #  Don't remove that option, but feel free to add more.
      options   => '_netdev',
      # If set will mount share inside /srv (or overridden mount_root)
      # and then bindmount to another directory elsewhere in the fs -
      # for fanatics.
      bindmount => undef,
      # Used to identify a catalog item for filtering by by
      # storeconfigs, kick ass.
      nfstag     => undef,
      # copied directly into /etc/exports as a string, for simplicity
      clients => '10.0.0.0/24(rw,insecure,no_subtree_check,async,no_root_squash)'
  }

  node client {
    class { 'nfs::server':
      nfs_v4              => true,
      nfs_v4_idmap_domain => $::domain
      nfs_v4_mount_root   => '/srv',
    }

    # We can as you by now know, override options set on the server
    # on the client node.
    # Be careful. Don't override mount points unless you are sure
    # that only one export will match your filter!

    Nfs::Client::Mount &lt;&lt;| # filter goes here # |&gt;&gt; {
      # Directory where we want export mounted on client
      mount     => undef,
      remounts  => false,
      atboot    => false,
      #  Don't remove that option, but feel free to add more.
      options   => '_netdev',
      # If set will mount share inside /srv (or overridden mount_root)
      # and then bindmount to another directory elsewhere in the fs -
      # for fanatics.
      bindmount => undef,
    }
  }

</pre>

##Requirements

###Modules needed:

stdlib by puppetlabs
concat by puppetlabs

###Software versions needed:

facter > 1.6.2
puppet > 2.6.2

##Limitations

##Contributing

Derdanne modules are open projects. So if you want to make this module even better, you can contribute to this module on [Github](https://github.com/derdanne/puppet-nfs).

This module based on Harald Skoglund <haraldsk@redpill-linpro.com> from https://github.com/haraldsk/puppet-module-nfs/ but has been fundementally refactored