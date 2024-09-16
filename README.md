# 1. Import/Export for NSX

## 1.1. Table of Contents
<!-- TOC -->

- [1. Import/Export for NSX](#1-importexport-for-nsx)
  - [1.1. Table of Contents](#11-table-of-contents)
  - [1.2. Overview](#12-overview)
  - [1.3. Getting Started](#13-getting-started)
    - [1.3.1. Install Python](#131-install-python)
    - [1.3.2. Download code](#132-download-code)
    - [1.3.3. Install Python modules and packages](#133-install-python-modules-and-packages)
    - [1.3.4. Update vmc.ini](#134-update-vmcini)
    - [1.3.5. Update config.ini](#135-update-configini)
  - [1.4. Running the script](#14-running-the-script)
    - [1.4.1. Export](#141-export)
    - [1.4.9. Running S3 export as a Lambda function](#149-running-s3-export-as-a-lambda-function)

<!-- /TOC -->
## 1.2. Overview

The Import/Export for NSX tool enable customers to save and restore their NSX-T network configurations.

There are many situations when customers want to migrate from an existing NSX-T deployment to a different one. While HCX addresses the data migration challenge, this tool offers customers the ability to copy the configuration from a source to a destination NSX-T instance, without having to deal with restoring an entire NSX-T binary and the subsequent outage caused during a restore.

## 1.3. Getting Started

### 1.3.1. Install Python

This tool is dependent on Python3, you can find installation instructions for your operating system in the Python [documentation](https://wiki.python.org/moin/BeginnersGuide/Download).

### 1.3.2. Download code

If you know git, clone the repo with

```bash
git clone https://github.com/awslabs/import-export-for-nsx.git
```

### 1.3.3. Install Python modules and packages

When you navigate to the nsx_import_export folder, you will find a requirements.txt file that list all your Python packages. They can all be installed by running the following command on Linux/Mac:

```bash
pip3 install -r requirements.txt
```

On Windows, use

```powershell
python -m pip install -r requirements.txt
```

### 1.3.4. Update vmc.ini

There are two authentication modes set in `vmc.ini`: `auth_mode = token` and `auth_mode = local`. Token mode uses a VMware Cloud on AWS API token to authenticate to the VMware Cloud on AWS service, and only works with VMC-A. If you have any other NSX-T deployment, you must use local mode. Local mode will authenticate directly from a local NSX-T manager. If you use token mode, you must fill in the refresh token and org/SDDC ID fields in `vmc.ini`. If you use local mode, you must fill in the `srcNSXmgrURL`, `srcNSXmgrUsername`, `srcNSXmgrPassword` fields in `vcenter.ini`

Local mode supports environment variables `EXP_srcNSXmgrURL`, `EXP_srcNSXmgrUsername`, and `EXP_srcNSXmgrPassword`. If you set these environment variables, you do not need to save these values in `vcenter.ini`

If you use local mode, you do not need any other settings in `vmc.ini`

Version 1.1 introduces the gov_cloud_urls flag in vmc.ini. The default value is False - change this value to True if you are using GovCloud.

Access to the VMware Cloud on AWS API is dependent on a refresh token. To generate a token for your account, see the [Generate API Tokens](https://docs.vmware.com/en/VMware-Cloud-services/services/Using-VMware-Cloud-Services/GUID-E2A3B1C1-E9AD-4B00-A6B6-88D31FCDDF7C.html) help article.

The Org ID and SDDC ID can be found on the Support tab of your SDDCs.

```bash
# Refresh tokens generated in the VMC console. Users have a separate token in each org
source_refresh_token    = XXXXXXXXXXXXXXX
dest_refresh_token      = XXXXXXXXXXXXXXX

# Organization and SDDC IDs are easily found in the support tab of any SDDC
source_org_id           = XXXXXXXXXXXXXXX
source_sddc_id          = XXXXXXXXXXXXXXX
dest_org_id             = XXXXXXXXXXXXXXX
dest_sddc_id            = XXXXXXXXXXXXXXX
```

### 1.3.5. Update config.ini

Config.ini contains configuration sections for import and export.

There are True/False flags that can be set for each configuration option. The default configuration enables all options.

For example, in this section of the configuration, the compute gateway networks would be exported, but the public IP and NAT associations would not be exported.

```bash
# Export the networks configured on the compute gateway?
network_export  = True
network_export_filename = cgw-networks.json

# Export the list of public IP addresses?
public_export = False
public_export_filename = public.json

# Export the NAT rules, including Public IP addresses?
nat_export = False
nat_export_filename = natrules.json
```

## 1.4. Running the script

### 1.4.1. Export

Export will export your existing SDDC configuration from your source SDDC to a set of files that can be subsequently used for import.

Run the following command to export:

Windows

```powershell
python ./nsx_import_export.py -o export
```

Linux/Mac

```bash
python3 nsx_import_export.py -o export
```

If all of the export options are enabled, this will export a set of files:

- Services.json
- cgw_groups.json
- cgw-networks.json
- cgw.json
- dfw_details.json
- dfw.json
- dhcp-static-bindings.json
- flex_seg_disc_prof.json
- flex_seg.json
- mcgw_fw.json
- mcgw_static_routes.json
- mcgw.json
- mgw_groups.json
- mgw.json
- mpl.json
- natrules.json
  nsx_adv_fw_policies.json
- nsx_adv_fw_profiles.json
- nsx_adv_fw_rules.json
- nsx_adv_fw_settings.json
- nsx_adv_fw_sigs.json
- public_ip_old_new.json
- public.json
- ral.json
- route_config.json
- s3-service_access.json
- sddc_info.json
- service_access.json
- services.json
- tags.json
- t1vpn.json
- t1vpn_service.json
- t1vpn_le.json
- vms.json
- vm-vifs.json
- vpn-bgp.json
- vpn-dpd.json
- vpn-ike.json
- vpn-l2.json
- vpn-l3.json
- vpn-local-bgp.json
- vpn-tunnel.json

A config.ini flag named 'export_history' allows for the JSON files to be zipped for archival purposes. A related configuration option named 'max_export_history_files' lets you control how many zipped archive files are retained.

Export is read-only and will not make any changes to your source NSX manager.

### 1.4.9. Running S3 export as a Lambda function

Install all required packages to a folder

```bash
mkdir python_req
cd python_req
pip3 install --target . -r ../requirements.txt
```

Zip python_req and upload it to a Lambda layer

Change export_folder in config.ini to /tmp, because /tmp is the only writable folder in Lambda

Ensure you have configured aws.ini with your S3 bucket settings

Ensure that you have granted the execution role write permissions to your S3 bucket

Add the following files individually to the function code, or zip them up and upload all at once:

- config_ini/*
- invoke_lambda.py
- nsx_import_export.py
- VMCImportExport.py

Change the Handler runtime settings to invoke_lambda.lambda_handler

Execute your Lambda function. Although it is possible to configure values in the config_ini files that you upload to the function code, it might be preferable to pass the required values via command line argument. See [invoke_lambda.py](invoke_lambda.py) for an example.
