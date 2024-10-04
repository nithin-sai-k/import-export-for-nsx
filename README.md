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
      - [1.3.4.1 Token mode](#1341-token-mode)
      - [1.3.4.2 Local mode](#1342-local-mode)
      - [1.3.4.3 NSX endpoint type](#1343-nsx-endpoint-type)
    - [1.3.5. Update config.ini](#135-update-configini)
  - [1.4. Running the script](#14-running-the-script)
    - [1.4.1. Export](#141-export)
    - [1.4.2 Export history](#142-export-history)
    - [1.4.9. Running S3 export as a Lambda function](#149-running-s3-export-as-a-lambda-function)

<!-- /TOC -->
## 1.2. Overview

The Import/Export for NSX tool enable customers to save and restore their NSX-T network configurations. For the initial launch of the tool, only export is supported.

There are many situations when customers want to migrate from an existing NSX-T deployment to a different one. While HCX addresses the data migration challenge, this tool offers customers the ability to copy the configuration from a source to a destination NSX-T instance, without having to deal with restoring an entire NSX-T binary and the subsequent outage caused during a restore.

## 1.3. Getting Started

### 1.3.1. Install Python

This tool is dependent on Python3, you can find installation instructions for your operating system in the Python [documentation](https://wiki.python.org/moin/BeginnersGuide/Download).

### 1.3.2. Download code

If you know git, clone the repo with:

```bash
git clone https://github.com/awslabs/import-export-for-nsx.git
```

If you do not know git, you can download a zipfile from [Releases](https://github.com/awslabs/import-export-for-nsx/releases)

### 1.3.3. Install Python modules and packages

When you navigate to the nsx_import_export folder, you will find a requirements.txt file that list all your Python packages. They can all be installed by running the following command on Linux/Mac:

```bash
pip3 install -r requirements.txt
```

On Windows, run:

```powershell
python -m pip install -r requirements.txt
```

### 1.3.4. Update vmc.ini

There are two authentication modes set in `vmc.ini`: `auth_mode = token` and `auth_mode = local`. 

#### 1.3.4.1 Token mode

Token mode uses a VMware Cloud on AWS API token to authenticate over the Internet to the VMware Cloud on AWS service. It can only be used with VMware Cloud on AWS, and not any other NSX-T endpoint. If you use token mode, you must fill in the refresh token and org/SDDC ID fields in `vmc.ini`.

For token mode, access to the VMware Cloud on AWS API is dependent on a refresh token. To generate a token for your account, see the [Generate API Tokens](https://docs.vmware.com/en/VMware-Cloud-services/services/Using-VMware-Cloud-Services/GUID-E2A3B1C1-E9AD-4B00-A6B6-88D31FCDDF7C.html) help article.

Token mode supports the environment variable `EXP_source_refresh_token`.  If you set this environment variable, you do not need to save it in `vmc.ini`.

Windows:

```powershell
$env:EXP_source_refresh_token = "xxxxx"
```

Linux/Mac:

```bash
EXP_source_refresh_token="xxxxx"
export EXP_source_refresh_token
```

The Org ID and SDDC ID that are required for token mode can be found on the Support tab of your SDDC.

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

#### 1.3.4.2 Local mode

You can use local mode to authenticate directly against the NSX-T manager in VMware Cloud on AWS. If you have any other NSX-T deployment, you *must* use local mode. If you use local mode, you can fill in the `srcNSXmgrURL`, `srcNSXmgrUsername`, `srcNSXmgrPassword` fields in `vcenter.ini`. If you do not want credentials persisted in plaintext, you can use the methods shown below.

> Note: You must use the *private* URL for direct NSX manager access with local mode. The URL can be found under the `Access NSX Manager via internal network (Private) section of the NSX Information tab in VMware Cloud on AWS. The URL will be in the following format: https://nsxmanager.sddc-1.2.3.4.vmwarevmc.com. Local mode will not work if you use the public NSX URL.

Local mode supports environment variables `EXP_srcNSXmgrURL`, `EXP_srcNSXmgrUsername`, and `EXP_srcNSXmgrPassword`. If you set these environment variables, you do not need to save these values in `vcenter.ini`.

> Note: You can use the read-only NSX `audit` user for export operations when using local mode.

Windows:

```powershell
$env:EXP_srcNSXmgrURL = "https://nsxmgr.fqdn.com"
$env:EXP_srcNSXmgrUsername = "admin"
$env:EXP_srcNSXmgrPassword = "password-for-admin"
```

Linux/Mac:

```bash
EXP_srcNSXmgrURL="https://nsxmgr.fqdn.com"
export EXP_srcNSXmgrURL
EXP_srcNSXmgrUsername="admin"
export EXP_srcNSXmgrUsername
EXP_srcNSXmgrPassword="password-for-admin"
export EXP_srcNSXmgrPassword
```

If you do not want to use environment variables, you can leave them blank. If values are not found in vcenter.ini, and not found in environment variables, the program will then prompt you to enter values as shown below.

```bash
Current authentication mode: local
Source NSX manager URL was not found in the environment variables.
Enter source NSX manager URL: http://nsxmgr.fqdn.com
Source NSX manager username was not found in the environment variables.
Enter source NSX manager username: admin
Source NSX password was not found in the environment variables.
Enter source NSX manager password: ******************************
```

If you use local mode, the only other setting in `vmc.ini` that you need to consider is `nsx_endpoint_type`

#### 1.3.4.3 NSX endpoint type

`vmc.ini` contains a variable named `nsx_endpoint_type`. It can be set to one of two values:

- vmc - Use this if your NSX-T endpoint is a VMware Cloud on AWS SDDC
- nsx - Use this for any other NSX-T endpoint

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

Export will export your existing configuration from your source SDDC/NSX manager to a set of files that can be subsequently used for import.

For a VMware Cloud on AWS endpoint, run the following command to export:

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

Export is read-only and will not make any changes to your source NSX.

### 1.4.2 Export history

A config.ini flag named 'export_history' allows for the JSON files to be zipped for archival purposes. A related configuration option named 'max_export_history_files' lets you control how many zipped archive files are retained. By default, this is enabled and set to a retention of 1. The zipped file will be in the `/json` folder and in the format `nsx_yyyy-mm-dd_hh-mm-ss_json-export.json`.

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
