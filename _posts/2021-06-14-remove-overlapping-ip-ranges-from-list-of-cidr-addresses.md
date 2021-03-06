---
layout: post
section-type: post
title: Remove Overlapping IP Address Ranges from a List of CIDR IP Addresses
category: 'powershell'
tags: [ 'azure', 'tutorial' ]
---

# Welcome Back Nerds!

Today, I wanted to share a problem I was faced with, as well as my scripted solution.

This week, my team needed to whitelist a large list of IP Address ranges (in CIDR notation) in an Azure Network Security Group (NSG).

AWS Provides a nice function to get these ranges. Once you've installed the `AWS.Tools.Common` module from the PSGallery, you will be able to run the following command:

```powershell
(Get-AWSPublicIpAddressRange | Where-Object IpAddressFormat -eq 'IPv4').IpPrefix
```

This will give you ALL of AWS's public-facing Infrastructure IP CIDR Ranges.

## The Problem

The issue is this, when pasting in the values, separated by commas, Azure doesn't like when there are overlapping entries.

"The subnet [X] overlaps the subnet [Y]"

![](/img/posts/azure_cidr_overlap_error.png)

But, this isn't even my fault! I got these IPs from AWS! Doesn't matter, Azure has no auto-filter for these duplicates / overlaps, and AWS provides them anyway.

What ever can we do to resolve this?

Write some PowerShell. 😊

## The solution

I've created a PowerShell function and published it in my [AdminToolkit](https://github.com/matthewjdegarmo/AdminToolkit) module called [Merge-CIDRIpRanges](https://github.com/matthewjdegarmo/AdminToolkit/blob/master/Functions/Public/Merge-CIDRIpRanges.ps1).

To install this module:

```powershell
Install-Module AdminToolkit
```

This function is meant to resolve these errors from Azure when trying to add many CIDR Ip Addresses, but there are duplicates. On top of that, removing these manually could take... well.. who knows how long, since you have no idea how many conflicts there are? You are doomed to `Ctrl + F` and delete the entries until the errors stop, that's it.

Simply provide your CIDR IP list to my function and poof, you will get a filtered, conflict-free list that Azure will love.

```PowerShell
Merge-CIDRIpRanges -CIDRAddresses (Get-AWSPublicIpAddressRange | ? IPAddressFormat -eq 'ipv4').IpPrefix
```

![](/img/posts/merge-cidripranges_progress.jpg)

You can then join these and copy them to your clipboard for Azure like this:

```powershell
$IPs = Merge-CIDRIpRanges -CIDRAddresses (Get-AWSPublicIpAddressRange -Region ca-central-1 | ? IPAddressFormat -eq 'ipv4').IpPrefix
$IPs -Join ',' | Set-Clipboard
```

This will combine the list of ranges into a single string, separated by commas, and then copy it to you clipboard, ready to `Ctrl + V`.

### How is this filtering

At a high level, when a conflict is detected, the winning range is the largest subnet.

Here is a quick example.
Given the following set of CIDR IPs:

```text
3.128.0.0/15
15.230.39.190/31
15.230.39.142/31
3.128.56.128/26
3.128.56.192/26
3.128.56.64/26
3.128.93.0/24
3.17.136.0/23
3.21.86.0/23
52.15.127.128/26
52.15.247.208/29
```

Would resolve down to..

```text
3.128.0.0/15
15.230.39.190/31
15.230.39.142/31
3.17.136.0/23
3.21.86.0/23
52.15.127.128/26
52.15.247.208/29
```

The following are all ranges that live inside of the `3.128.0.0/15` subnet, so they are all removed, and `3.128.0.0/15` is kept.

```text
3.128.93.0/24
3.128.56.64/26
3.128.56.128/26
3.128.56.192/26
```

## That's all for this one

Thanks for tuning in, and I hope you find use in this solution!
