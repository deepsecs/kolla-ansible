.. _swift-guide:

==============
Swift in Kolla
==============

Overview
========
Kolla can deploy a full working Swift setup in either a **all-in-one** or
**multinode** setup.

Disks with a partition table (recommended)
==========================================

Swift requires block devices to be available for storage. To prepare a disk
for use as a Swift storage device, a special partition name and filesystem
label need to be added.

The following should be done on each storage node, the example is shown
for three disks:

::

    # <WARNING ALL DATA ON DISK will be LOST!>
    index=0
    for d in sdc sdd sde; do
        parted /dev/${d} -s -- mklabel gpt mkpart KOLLA_SWIFT_DATA 1 -1
        sudo mkfs.xfs -f -L d${index} /dev/${d}1
        (( index++ ))
    done

For evaluation, loopback devices can be used in lieu of real disks:

::

    index=0
    for d in sdc sdd sde; do
        free_device=$(losetup -f)
        fallocate -l 1G /tmp/$d
        losetup $free_device /tmp/$d
        parted $free_device -s -- mklabel gpt mkpart KOLLA_SWIFT_DATA 1 -1
        sudo mkfs.xfs -f -L d${index} ${free_device}p1
        (( index++ ))
    done

Disks without a partition table
===============================

Kolla also supports unpartitioned disk (filesystem on ``/dev/sdc`` instead of
``/dev/sdc1``) detection purely based on filesystem label. This is generally
not a recommended practice but can be helpful for Kolla to take over Swift
deployment already using disk like this.

Given hard disks with labels swd1, swd2, swd3, use the following settings in
``ansible/roles/swift/defaults/main.yml``.

::

    swift_devices_match_mode: "prefix"
    swift_devices_name: "swd"

Rings
=====

Before running Swift we need to generate **rings**, which are binary compressed
files that at a high level let the various Swift services know where data is in
the cluster. We hope to automate this process in a future release.

The following example commands should be run from the ``operator`` node to
generate rings for a demo setup.  The commands work with **disks with partition
table** example listed above.  Please modify accordingly if your setup is
different.

::

  STORAGE_NODES=(192.168.0.2 192.168.0.3 192.168.0.4)
  KOLLA_SWIFT_BASE_IMAGE="kolla/oraclelinux-source-swift-base:4.0.0"

  mkdir -p /etc/kolla/config/swift

  # Object ring
  docker run \
    --rm \
    -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
    $KOLLA_SWIFT_BASE_IMAGE \
    swift-ring-builder \
      /etc/kolla/config/swift/object.builder create 10 3 1

  for node in ${STORAGE_NODES[@]}; do
      for i in {0..2}; do
        docker run \
          --rm \
          -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
          $KOLLA_SWIFT_BASE_IMAGE \
          swift-ring-builder \
            /etc/kolla/config/swift/object.builder add r1z1-${node}:6000/d${i} 1;
      done
  done

  # Account ring
  docker run \
    --rm \
    -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
    $KOLLA_SWIFT_BASE_IMAGE \
    swift-ring-builder \
      /etc/kolla/config/swift/account.builder create 10 3 1

  for node in ${STORAGE_NODES[@]}; do
      for i in {0..2}; do
        docker run \
          --rm \
          -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
          $KOLLA_SWIFT_BASE_IMAGE \
          swift-ring-builder \
            /etc/kolla/config/swift/account.builder add r1z1-${node}:6001/d${i} 1;
      done
  done

  # Container ring
  docker run \
    --rm \
    -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
    $KOLLA_SWIFT_BASE_IMAGE \
    swift-ring-builder \
      /etc/kolla/config/swift/container.builder create 10 3 1

  for node in ${STORAGE_NODES[@]}; do
      for i in {0..2}; do
        docker run \
          --rm \
          -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
          $KOLLA_SWIFT_BASE_IMAGE \
          swift-ring-builder \
            /etc/kolla/config/swift/container.builder add r1z1-${node}:6002/d${i} 1;
      done
  done

  for ring in object account container; do
    docker run \
      --rm \
      -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
      $KOLLA_SWIFT_BASE_IMAGE \
      swift-ring-builder \
        /etc/kolla/config/swift/${ring}.builder rebalance;
  done

For more info, see
https://docs.openstack.org/project-install-guide/object-storage/ocata/initial-rings.html

Deploying
=========
Enable Swift in ``/etc/kolla/globals.yml``:

::

    enable_swift : "yes"

Once the rings are in place, deploying Swift is the same as any other Kolla
Ansible service:

::

  kolla-ansible deploy

Validation
==========
A very basic smoke test:

::

  $ swift stat
                            Account: AUTH_4c19d363b9cf432a80e34f06b1fa5749
                       Containers: 1
                          Objects: 0
                            Bytes: 0
  Containers in policy "policy-0": 1
     Objects in policy "policy-0": 0
       Bytes in policy "policy-0": 0
      X-Account-Project-Domain-Id: default
                      X-Timestamp: 1440168098.28319
                       X-Trans-Id: txf5a62b7d7fc541f087703-0055d73be7
                     Content-Type: text/plain; charset=utf-8
                    Accept-Ranges: bytes

  $ swift upload mycontainer README.rst
  README.md

  $ swift list
  mycontainer

  $ swift download mycontainer README.md
  README.md [auth 0.248s, headers 0.939s, total 0.939s, 0.006 MB/s]
