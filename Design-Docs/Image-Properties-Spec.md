---
title: SCS Image Metadata Proposal
version: 2021-07-02-001
authors: Kurt Garloff, Christian Berendt
state: Draft
---

SCS Image Metadata Proposal (DRAFT) SCS

Please take note, that this is a DRAFT open for discussion (2021-07-02),
though it will be locked down for SCS R0 in a few days.

# Motivation

Many clouds offer standard Operating System images for their users' convenience.
To make them really useful, they should contain meta data (properties) to allow
users to understand what they can expect using these images.

The specification is targeting images that are managed by the service provider,
provided for public consumption. The spec or parts of it however might turn out
to be useful whenever someone manages images for somebody else's consumption.

We also suggest a standard set of images to be available.

# Overview

We categorize the image properties into a few buckets

* Technical requirements and features
* Image handling aspects
* Licensing/Maintenance/Support aspects

# Naming

We suggest plain OS images to be named "Distribution Version",
e.g. "openSUSE Leap 15.3" or "Ubuntu 20.04" or "CentOS 8.4", "Windows Server 2012R2".
We do not normally recommend to add more detailed patch levels into the name.

Special variants that include specific non-standard features should be named
"Distribution Version Feature1 Feature2".

There are several policies possible to provide updated images to include the latest
bug- and security fixes. This is outlined in update policy description below.

# Technical requirements and features

This is dependent on whether we deal with VM images or container images.

For VM images (OpenStack), we recommend to use the properties as described
in the [OpenStack Image documentation](https://docs.openstack.org/glance/latest//admin/useful-image-properties).

The following properties are considered mandatory:
* `architecture`, `hypervisor_type`
* `min_disk_size`, `min_ram`
* `os_version`, `os_distro`
* `hw_rng_model`, `hw_disk_bus` (`scsi` recommended, and then setting `hw_scsi_model` is also recommended)

The following properties are recommended (if the features are supported):
* `os_secure_boot`, `hw_firmware_type`
* `hw_watchdog_action`, `hw_mem_encryption`, `hw_pmu`, `hw_video_ram`, `hw_vif_multiqueue_enabled`

The `trait:XXX=required` property can be used to indicate that certain virtual hardware
features `XXX` are required.

# Image handling

## Image updating 

It is recommended that provider managed images are regularly updated.
This means that users referencing an image *by name* will always get the latest image for the
operating system indicated by that name (which includes a version number, but not the patch
level).

Technically, the thus updated image is a new image and will thus carry a new UUID.
It is recommended that the old image gets renamed (e.g. build date or patch level attached)
and hidden (`os_hidden=true`), but remains accessible via its (unchanged) UUID. 

The update handling by the provider is described via the properties `replace_frequency`,
`uuid_validity`, `provided_until`, and `hotfix_policy`.

The `replace_frequency`, `provided_until`, and `hotfix_policy` fields reference to the image name.

| `replace_frequency` | meaning              |
|---------------------|----------------------|
| `yearly`            | the image will get replaced *at least* once per year    |
| `quarterly`         | the image will get replaced *at least* once per quarter |
| `monthly`           | the image will get replaced *at least* once per month   |
| `weekly`            | the image will get replaced *at least* once per week    |
| `daily`             | the image will get replaced *at least* once per day     |
| `critical_bug`      | the image will get replaced for critical issues only    |
| `never`             | the image referenced by name will never change (until the date `provided_until`) |

Note the *at least* wording: Providers can replace images more often.
The frequency is starting from the first release; so an image published on 2021-04-14 with an
update frequency of `monthly`, should be replaced no later than 2021-05-14. Due to weekends
etc., up to 3 days later is not considered a violation of this policy. So the a valid sequence
from an image with `monthly` update frequency might be 2021-04-14, 2021-05-14, 2021-06-15,
2021-07-14, 2021-07-27 (hotfix), 2021-08-13 ...

The `hotfix_hours` field indicates how providers deal with critical security issues
that affect the images; it is an optional field that contains a numerical value, indicating
how quickly (in hours) a new image is provided *after the latter of the points in time that
the issue becomes public and a tested fix is available as maintenance update from the upstream
distribution*. A value of 0 indicates a best-effort approach without firm SLAs; the field not
being present indicates no commitment. A value of 48 would indicate that the provider
commits to a new image within 48hrs. A critical issue is defined as a security vulnerability
with a CVSS score of 9.0 or higher that affects a package that is included in the image.

The `provided_until` field is supposed to contain a date in `YYYY-MM-DD` format that
indicates until when an image under this name will be provided and (according to the
`replace_frequency`) updated at least. (Providers are free to provide updates for
longer or leave the non-updated image visible for longer.)
If this field is not set, no promises are made.

The `uuid_validity` field indicates how long the public image will be referencable
by it's UUID.

| `uuid_validity`  | meaning                |
|------------------|------------------------|
| `none`           | UUID will only be valid as long as the content does not change          |
| `last-N`         | The last N images for newer replacement will remain accessible via UUID |
| `YYYY-MM-DD`     | UUID will be valid until at least the date YYYY-MM-DD                   |
| `notice`         | UUID will remain valid until a deprecation notice will be published     |
| `forever`        | UUID will remain valid for as long as the cloud operates                |

Note that the old images must be hidden from the image catalogue or renamed (or both)
to avoid failing referencing by name. Note that `last-N` may be limited by the `provided_till`
date.

The three properties `uuid_validity`, `provided_until` and `update_frequency` are mandatory;
the field `hotfix_hours` is optional.

All dates are in UTC.

### Example:

Providing an image with name `OPSYS MAJ.MIN` with
`replace_frequency=monthly`, `provided_until=2022-09-30`, `uuid_validity=2022-12-31`,
`hotfix_hours=0`
means that we will have a new image with this name at least once per month (starting from
the initial release) until the end of September 2022. Old images will be hidden and/or
renamed, but remain accessible via their UUID until at least the end of 2022 (in Universal Time).
The provider makes an effort to replace images upon critical security issues out of order.

## Image Origin

* Mandatory: `image_source` needs to be a URL to point to a place from which the image can be downloaded.
  (Note: This may be set to the string "private" to indicate that the image can not be freely
   downloaded.)
* Mandatory: `image_description` needs to be a URL with release notes and other human readable data
  about the image.

* Recommended *tag*: `managed_by_VENDOR`

## Image build info

* Mandatory: `image_build_date` needs to be `YYYY-MM-DD` or `YYYY-MM-DD hh:mm[:ss]` (time in UTC,
  24hrs clock).
  All publicly released and generally recommended patches before this date must be included in the
  image build. If the cutoff date is earlier, this cutoff date needs to be set instead, even
  if the actual build happens significantly after the cutoff date. If not all patches can be
  included for a good reason, then the `patchlevel` field (see below) must be used to describe
  the patch status.
* Mandatory: `image_original_user` is the default login user for the operating system which can connect
  to the image via the injected SSH key or provided password. (This can be set to `none` if no default
  user name exists for the operating system.)
* Optional: `patchlevel` can be set to an operating specific patch level that describes the
  patch status -- typically we would expect the `image_build_date` to be sufficient.

* Recommended: `os_hash_algo` and `os_hash_value`: The sha256 or sha512 hash
  for the image file.  (This references the image file in the format it is stored in, we 
  recommend raw over qcow2 for systems that use ceph.) Note that these values are
  typically generated automatically upon image registration.
* Recommended: Digital image signature according to the [glance image
  specification](https://docs.openstack.org/glance/wallaby/user/signature.html),
  using `img_signature`, `img_signature_hash_method`, `img_signature_key_type`,
  `img_signature_certificate_uuid`.

* Recommended *tag*: `os:OPERATINGSYSTEM`

## Licensing / Maintenance subscription / Support 

Some images require a license; in many cases the cloud providers include the license cost
by a per-use (e.g. hourly) fee. However, it is also possible sometimes that customers
use their own license agreements with the OS vendor with a bring-your-own-license (BYOL)
program. These properties may be attached to the image. Note that free Linux images
might not use any of these properties, except maybe `maintained_until`. Note that
Windows images would typically require `license_included`, `subscription_included`.
A boolean property that is not present is considered to be `false`.

* Optional: `license_included` (boolean) indicates whether ot not the flavor fee
  includes the licenses required to use this image. This field is mandatory for
  images that contain software that requires commercial licenses.
* Optional: `license_required` (boolean) indicates whether or not a customer must bring
  its own license to be license compliant. This can not be true at the same time as the
  previous setting. This field is mandatory IF customers need to bring their own
  license to use the image.
* Optional: `subscription_included` (boolean) indicates that the image contains already
  a maintenance subscription which typically gives access to bug fixes, security
  fixes and (minor) function updates. If a subscription is included, the CSP should
  have prepared the image to also receive the provided maintenance updates from the
  vendor (optionally via a mirror).
* Optional: `subscription_required` (boolean) indicates that the customer requires
  a maintenance subscription from the OS vendor in order to receive fixes
  (which is often also a prerequisite to be eligible for support).
* Optional: `maintained_until: YYYY-MM-DD` promises maintenance from the OS vendor
  until at least this date (in UTC).
* Optional: `l1_support_contact` contains a URI that provides customer support
  contact for issues with this image. Note that this field must only be set if the
  service provider does provide support for this image included in the image/flavor
  pricing (but it might be provided by a contracted 3rd party, e.g. the OS vendor).


