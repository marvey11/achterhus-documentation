# External Hard Drive Health Checks on Ubuntu Server 26.04 LTS

## 1. Introduction

This document describes how to inspect, diagnose, and monitor the health of an external hard drive connected to an Ubuntu Server 26.04 LTS instance.

The examples use `/dev/sda` as the target device.

The procedures cover:

- Identifying physical disks and their partitions.
- Retrieving disk manufacturer, model, serial number, and capacity.
- Checking SMART (Self-Monitoring, Analysis and Reporting Technology) health information.
- Examining SMART attributes and historical error information.
- Running built-in disk self-tests.
- Checking kernel logs for disk-related errors.
- Detecting bad sectors and read errors.
- Checking filesystem consistency.
- Monitoring disk health continuously.
- Interpreting diagnostic results and identifying potential hardware failures.

The primary diagnostic utility used throughout this document is `smartctl`, provided by the `smartmontools` package.

### Important considerations

Before running diagnostic commands, consider the following:

1. Always verify that `/dev/sda` refers to the intended external hard drive.
2. Linux device names such as `/dev/sda` are not persistent. They can change after a reboot or when devices are reconnected.
3. Prefer persistent device identifiers under `/dev/disk/by-id/` for scripts and automated monitoring.
4. SMART information is not available through every USB-to-SATA bridge.
5. SMART health status alone cannot guarantee that a disk is healthy.
6. If a disk is suspected of failing, prioritize backing up its data before performing lengthy diagnostic operations.
7. Never run filesystem repair commands against a mounted filesystem.
8. Never run destructive write tests against a disk containing data that needs to be preserved.

---

## 2. Installing the diagnostic utilities

Update the package index:

```bash
sudo apt update
```

Install the SMART monitoring utilities:

```bash
sudo apt install smartmontools
```

This package provides two important commands:

| Command | Description |
| --- | --- |
| `smartctl` | Retrieves SMART information and runs disk self-tests. |
| `smartd` | Monitors disks continuously and reports detected problems. |

Check the installed version:

```bash
smartctl --version
```

Check the available options:

```bash
smartctl --help
```

Additional useful utilities can be installed as follows:

```bash
sudo apt install nvme-cli
```

This provides diagnostic commands for NVMe devices.

The `e2fsprogs` package provides filesystem checking and bad-block utilities. It is normally already installed on Ubuntu:

```bash
sudo apt install e2fsprogs
```

Other useful tools:

```bash
sudo apt install hdparm lsscsi
```

---

## 3. Identifying the external hard drive

Before performing any diagnostics, identify the correct physical disk.

### 3.1 List all block devices

```bash
lsblk
```

For more detailed information:

```bash
lsblk -o NAME,PATH,TYPE,SIZE,MODEL,SERIAL,TRAN,FSTYPE,MOUNTPOINTS
```

Example:

```text
NAME   PATH       TYPE   SIZE MODEL             SERIAL       TRAN FSTYPE MOUNTPOINTS
sda    /dev/sda   disk   4.0T External HDD      ABC123456    usb
├─sda1 /dev/sda1  part   4.0T
sdb    /dev/sdb   disk  512G Samsung SSD        XYZ987654    sata
├─sdb1 /dev/sdb1  part   512G
```

Important columns:

- `NAME`: Linux device name.
- `PATH`: Full device path.
- `TYPE`: Device type (`disk`, `part`, etc.).
- `SIZE`: Device capacity.
- `MODEL`: Manufacturer/model information.
- `SERIAL`: Device serial number.
- `TRAN`: Connection type, such as USB or SATA.
- `FSTYPE`: Filesystem type.
- `MOUNTPOINTS`: Current mount locations.

### 3.2 Display disk information

```bash
sudo fdisk -l /dev/sda
```

This displays the partition table, disk capacity, logical and physical sector sizes, and partition information.

### 3.3 Display detailed device information

```bash
sudo udevadm info --query=all --name=/dev/sda
```

This provides additional information about the device, including its hardware identifiers and connection properties.

### 3.4 Identify persistent device names

List available disk identifiers:

```bash
ls -l /dev/disk/by-id/
```

Example:

```text
ata-WDC_WD40... -> ../../sda
usb-Seagate_Expansion... -> ../../sda
```

Use a persistent identifier instead of `/dev/sda` whenever possible.

For example:

```bash
DISK="/dev/disk/by-id/usb-Seagate_Expansion_ABC123"
```

Verify its target:

```bash
readlink -f "$DISK"
```

This reduces the risk of accidentally diagnosing a different disk after a reboot or reconnection.

---

## 4. Checking whether SMART is supported

SMART is a technology built into most modern hard drives and SSDs.

It allows the operating system to retrieve internal health information, including:

- Temperature.
- Power-on hours.
- Reallocated sectors.
- Pending sectors.
- Uncorrectable errors.
- Error logs.
- Internal self-test results.

### 4.1 Detect SMART-compatible devices

```bash
sudo smartctl --scan
```

Alternatively, attempt to open detected devices:

```bash
sudo smartctl --scan-open
```

These commands identify devices that can be accessed by `smartctl`.

### 4.2 Check SMART support

```bash
sudo smartctl -i /dev/sda
```

Look for information similar to:

```text
Device Model:     External HDD
Serial Number:    ABC123456
Firmware Version: 1.00
User Capacity:    4,000,000,000,000 bytes

SMART support is: Available
SMART support is: Enabled
```

The important fields are:

- `SMART support is: Available`: The device supports SMART.
- `SMART support is: Enabled`: SMART monitoring is enabled.

If SMART is supported but disabled, it can sometimes be enabled using:

```bash
sudo smartctl -s on /dev/sda
```

Verify the result:

```bash
sudo smartctl -i /dev/sda
```

**Note:** SMART availability depends on the drive, controller, and connection interface. Some USB enclosures do not correctly forward SMART commands.

---

## 5. Checking the overall SMART health status

The simplest health check is:

```bash
sudo smartctl -H /dev/sda
```

Example output:

```text
=== START OF READ SMART DATA SECTION ===
SMART overall-health self-assessment test result: PASSED
```

Possible results:

| Result | Meaning |
| --- | --- |
| PASSED | The drive has not reported a failing overall SMART health status. |
| FAILED | The drive reports a failure condition. |
| UNKNOWN | The health status cannot be determined. |
| SMART unavailable | SMART information cannot be retrieved. |

### Important interpretation

A `PASSED` result does not necessarily mean that the disk is completely healthy.

For example, a disk can report `PASSED` while already having:

- Reallocated sectors.
- Pending sectors.
- Increasing read errors.
- A history of failed self-tests.
- Other warning indicators.

Consequently, the overall health status should always be considered together with the detailed SMART attributes and error logs.

---

## 6. Retrieving comprehensive SMART information

The most useful diagnostic command is:

```bash
sudo smartctl -x /dev/sda
```

This retrieves extensive diagnostic information supported by the device.

It can include:

1. Device identification.
2. SMART capabilities.
3. SMART health status.
4. SMART attributes.
5. Error logs.
6. Self-test logs.
7. Temperature information.
8. Device statistics.
9. Interface-specific diagnostic information.

For a slightly less comprehensive report:

```bash
sudo smartctl -a /dev/sda
```

For most troubleshooting purposes, `-x` is the recommended command.

### 6.1 Save the complete diagnostic output

It is useful to save the report for future comparison.

```bash
sudo smartctl -x /dev/sda | tee smart-report.txt
```

To include a timestamp:

```bash
sudo smartctl -x /dev/sda | tee "smart-report-$(date +%Y%m%d-%H%M%S).txt"
```

### 6.2 Generate JSON output

For automated monitoring or integration with scripts:

```bash
sudo smartctl -j -x /dev/sda
```

To save the JSON report:

```bash
sudo smartctl -j -x /dev/sda > smart-report.json
```

JSON output is useful for parsing individual SMART attributes and building automated health-check systems.

---

## 7. Examining individual SMART attributes

Retrieve the SMART attributes:

```bash
sudo smartctl -A /dev/sda
```

Example:

```text
ID# ATTRIBUTE_NAME          VALUE WORST THRESH TYPE      RAW_VALUE
  1 Raw_Read_Error_Rate     100   100   006     Pre-fail  0
  5 Reallocated_Sector_Ct   100   100   010     Pre-fail  0
  9 Power_On_Hours           95    95   000     Old_age   3500
 12 Power_Cycle_Count        99    99   000     Old_age   25
194 Temperature_Celsius      110   110   000     Old_age   32
197 Current_Pending_Sector    100   100   000     Old_age   0
198 Offline_Uncorrectable     100   100   000     Old_age   0
```

The exact attributes and their interpretation depend on the drive manufacturer and model.

### 7.1 Important SMART attributes

The following attributes are particularly useful when investigating conventional HDDs.

| ID | Attribute | Description | Warning indication |
| --- | --- | --- | --- |
| 5 | Reallocated_Sector_Ct | Number of sectors replaced with spare sectors. | Increasing value or any unexpected nonzero value. |
| 9 | Power_On_Hours | Total operating time. | Useful for determining drive age. |
| 12 | Power_Cycle_Count | Number of power cycles. | Unexpected increases may indicate power issues. |
| 187 | Reported_Uncorrect | Uncorrectable errors reported to the host. | Nonzero or increasing values. |
| 188 | Command_Timeout | Commands that timed out. | Increasing values. |
| 192 | Power-Off_Retract_Count | Emergency head retracts or unsafe power events. | Unexpected increases. |
| 193 | Load_Cycle_Count | Number of head load/unload cycles. | Excessively high values. |
| 194 | Temperature_Celsius | Current drive temperature. | Excessive operating temperature. |
| 196 | Reallocated_Event_Count | Number of sector reallocation events. | Increasing values. |
| 197 | Current_Pending_Sector | Sectors awaiting reallocation. | Any persistent nonzero value. |
| 198 | Offline_Uncorrectable | Uncorrectable sectors detected during offline scanning. | Any nonzero value. |
| 199 | UDMA_CRC_Error_Count | Interface transmission errors. | Increasing values. |

### 7.2 Understanding normalized and raw values

SMART attributes typically contain:

- `VALUE`: Current normalized value.
- `WORST`: Worst normalized value recorded.
- `THRESH`: Failure threshold.
- `TYPE`: Attribute classification.
- `RAW_VALUE`: Raw value reported by the device.

Normalized values generally range from 1 to 253, depending on the attribute and implementation.

A normalized value reaching or falling below its threshold can indicate an impending or actual failure condition.

However, raw values are frequently vendor-specific.

For example, a raw read error rate might be represented differently by different manufacturers. It should not automatically be interpreted as a literal count of defective sectors.

### 7.3 Attributes requiring particular attention

#### Reallocated sectors (ID 5)

```bash
sudo smartctl -A /dev/sda | grep -i Reallocated
```

A nonzero value indicates that the drive has already replaced defective sectors with spare sectors.

An increasing value is particularly concerning.

#### Current pending sectors (ID 197)

```bash
sudo smartctl -A /dev/sda | grep -i Pending
```

A nonzero value indicates sectors that the drive cannot currently read reliably and may need to reallocate.

This is an important warning indicator.

#### Offline uncorrectable sectors (ID 198)

```bash
sudo smartctl -A /dev/sda | grep -i Offline_Uncorrectable
```

Nonzero values indicate sectors that could not be corrected during offline testing.

#### Interface CRC errors (ID 199)

```bash
sudo smartctl -A /dev/sda | grep -i CRC
```

Increasing CRC errors can indicate problems with:

- USB cables.
- SATA cables.
- Connectors.
- Enclosures.
- Controllers.
- Electrical interference.

These errors do not necessarily indicate defective disk media.

---

## 8. Checking the disk temperature

Retrieve the complete SMART report:

```bash
sudo smartctl -x /dev/sda
```

Look for temperature information.

Alternatively:

```bash
sudo smartctl -A /dev/sda | grep -i Temperature
```

Some drives provide a temperature log:

```bash
sudo smartctl -l scttemp /dev/sda
```

Temperature is an important operational indicator, especially for external HDDs operating continuously.

General guidance:

| Temperature | Interpretation |
| --- | --- |
| 20–40 °C | Generally comfortable operating range. |
| 40–45 °C | Elevated; monitor cooling and enclosure ventilation. |
| 45–50 °C | High; investigate cooling and operating conditions. |
| Above 50 °C | Potentially problematic; consult the manufacturer's specifications. |

These are general monitoring guidelines, not universal manufacturer limits.

The recommended operating temperature varies by drive model.

---

## 9. Checking historical disk errors

SMART maintains diagnostic logs that can reveal previous hardware problems.

### 9.1 SMART error log

```bash
sudo smartctl -l error /dev/sda
```

This displays recorded disk errors.

Look for:

- Uncorrectable read errors.
- Write errors.
- Command timeouts.
- Interface errors.
- Repeated failures at particular logical block addresses (LBAs).

A disk with repeated errors should be investigated even if its overall SMART health status is `PASSED`.

### 9.2 SMART self-test log

```bash
sudo smartctl -l selftest /dev/sda
```

This displays previously performed SMART self-tests.

Example:

```text
Num  Test_Description    Status                  Remaining
# 1  Short offline       Completed without error      00%
# 2  Extended offline    Completed without error      00%
```

Possible results include:

- Completed without error.
- Completed: read failure.
- Completed: electrical failure.
- Completed: servo/seek failure.
- Aborted by host.
- Interrupted (host reset).

A historical failed test is important even if a subsequent test succeeds.

---

## 10. Running SMART self-tests

SMART supports internal drive tests.

These tests are executed by the drive firmware and generally do not require unmounting the filesystem.

Nevertheless, avoid running tests when the disk is already exhibiting severe failures. Back up important data first.

### 10.1 Short self-test

Start a short test:

```bash
sudo smartctl -t short /dev/sda
```

Example output:

```text
Please wait 2 minutes for test to complete.
```

The actual duration depends on the drive.

Check the test results:

```bash
sudo smartctl -l selftest /dev/sda
```

A short test typically checks:

- Electrical components.
- Basic mechanical operation.
- Selected read operations.
- Internal drive functionality.

It is a good first diagnostic test.

### 10.2 Extended or long self-test

Start an extended test:

```bash
sudo smartctl -t long /dev/sda
```

The extended test performs a more comprehensive internal examination of the drive's readable media.

Depending on capacity and drive characteristics, it can take several hours.

Check the estimated completion time:

```bash
sudo smartctl -c /dev/sda
```

This displays the supported SMART capabilities and estimated self-test durations.

Check the test status:

```bash
sudo smartctl -c /dev/sda
```

Check the final result:

```bash
sudo smartctl -l selftest /dev/sda
```

A successful result should indicate:

```text
Completed without error
```

### 10.3 Abort a running self-test

If necessary:

```bash
sudo smartctl -X /dev/sda
```

This requests that the running self-test be aborted.

### 10.4 Recommended test sequence

For a drive that does not show obvious failure symptoms:

1. Retrieve the complete SMART report.
2. Run a short self-test.
3. Review the self-test log.
4. Run an extended self-test.
5. Review the self-test log again.
6. Compare SMART attributes with their previous values.

---

## 11. Checking Linux kernel logs for disk errors

SMART is not the only source of diagnostic information.

Linux kernel logs can reveal hardware communication problems that may not appear in the SMART report.

### 11.1 Display recent kernel messages

```bash
sudo dmesg -T
```

### 11.2 Search for disk-related errors

```bash
sudo dmesg -T | grep -Ei 'sda|I/O error|medium error|uncorrect|reset|failed|timeout'
```

### 11.3 Inspect the system journal

```bash
sudo journalctl -k
```

Display kernel messages from the current boot:

```bash
sudo journalctl -k -b
```

Display recent messages:

```bash
sudo journalctl -k -n 200
```

Search for disk-related errors:

```bash
sudo journalctl -k | grep -Ei 'sda|I/O error|medium error|uncorrect|reset|timeout'
```

### 11.4 Important kernel messages

Pay particular attention to messages such as:

```text
I/O error, dev sda
```

Indicates that a disk operation failed.

```text
Buffer I/O error on dev sda
```

Indicates an I/O problem involving the device.

```text
ata1: hard resetting link
```

Indicates a SATA communication reset.

```text
device offline
```

Indicates that the operating system has lost access to the device.

```text
blk_update_request: I/O error
```

Indicates a failed block-device operation.

```text
uas_eh_abort_handler
```

Indicates an error-handling event involving USB Attached SCSI.

Repeated USB resets, disconnects, and command timeouts can indicate problems with the cable, USB enclosure, power supply, or drive itself.

**Important:** Do not automatically assume that every I/O error means the disk surface is defective. Connection and power problems can produce similar symptoms.

---

## 12. Checking filesystem consistency

SMART tests examine the physical device. Filesystem checks examine the logical filesystem structures stored on the disk.

These are different types of diagnostics.

First, identify the filesystem and partitions:

```bash
lsblk -f /dev/sda
```

Example:

```text
NAME   FSTYPE LABEL UUID
sda
└─sda1 ext4         1234-5678
```

### 12.1 Check whether the partition is mounted

```bash
findmnt
```

Or:

```bash
lsblk -o NAME,FSTYPE,MOUNTPOINTS /dev/sda
```

### 12.2 Check an ext4 filesystem

For an ext4 filesystem, use:

```bash
sudo e2fsck -fn /dev/sda1
```

Options:

- `-f`: Force a filesystem check.
- `-n`: Answer no to questions and open the filesystem read-only.

This is intended as a diagnostic check without applying repairs.

**Important:** The filesystem should be unmounted before running filesystem checks. Do not run `e2fsck` against a mounted filesystem.

If errors are detected, plan a proper repair procedure after securing the data.

### 12.3 Check other filesystem types

Different filesystems require different tools.

| Filesystem | Diagnostic utility |
| --- | --- |
| ext2/ext3/ext4 | `e2fsck` |
| XFS | `xfs_repair -n` |
| Btrfs | `btrfs scrub` |
| FAT | `fsck.fat -n` |
| NTFS | `ntfsfix -n` |

Filesystem-specific restrictions apply. In particular, do not assume that a filesystem can be safely checked while mounted.

For XFS, use the no-modification mode:

```bash
sudo xfs_repair -n /dev/sda1
```

For Btrfs, a scrub can be performed on a mounted filesystem:

```bash
sudo btrfs scrub start -B /mount/point
```

Check scrub results:

```bash
sudo btrfs scrub status /mount/point
```

A scrub checks data and metadata checksums. Its repair capabilities depend on the availability of redundant copies.

---

## 13. Detecting bad sectors using badblocks

The `badblocks` utility can scan a disk for unreadable blocks.

It is part of the `e2fsprogs` package.

### 13.1 Read-only bad-block scan

Run:

```bash
sudo badblocks -sv /dev/sda
```

Options:

- `-s`: Display progress.
- `-v`: Verbose output.

This performs a read-only scan.

It attempts to read blocks throughout the specified device and reports blocks that cannot be read.

For a large HDD, this can take many hours.

The scan should be performed only after considering the disk's current condition. If the drive is already failing or making unusual mechanical noises, prioritize data recovery instead.

### 13.2 Important warning about write-mode testing

The following command is destructive:

```bash
sudo badblocks -wsv /dev/sda
```

It writes test patterns across the device and destroys existing data.

**NEVER execute this command on a disk containing data that needs to be preserved.**

The `-n` option provides a non-destructive read/write test, but it is still a lengthy and potentially stressful operation.

For most production external drives, SMART self-tests and read-only diagnostics should be preferred.

---

## 14. Additional diagnostics for USB-connected hard drives

External hard drives commonly use USB-to-SATA bridges.

Some bridges forward SMART commands correctly; others do not.

If this command fails:

```bash
sudo smartctl -x /dev/sda
```

Try the SAT (SCSI/ATA Translation) interface:

```bash
sudo smartctl -d sat -x /dev/sda
```

Some bridges require a different device type or bridge-specific options.

To investigate available devices:

```bash
sudo smartctl --scan-open
```

The `-d sat` option is particularly useful for USB enclosures that expose an ATA disk through a SCSI-like interface.

If the bridge still does not provide SMART information, consider:

- Testing with another USB enclosure.
- Connecting the drive directly through SATA.
- Testing with another USB port.
- Replacing the USB cable.
- Checking whether the enclosure firmware supports SMART passthrough.

### 14.1 Inspect USB connection information

```bash
lsusb
```

Display USB topology:

```bash
lsusb -t
```

This can help identify the USB controller, connection speed, and driver used by the external enclosure.

### 14.2 Inspect USB-related kernel messages

```bash
sudo journalctl -k | grep -Ei 'usb|uas|reset|disconnect|sda'
```

Repeated USB disconnects or resets should be investigated.

### 14.3 Check device statistics

```bash
sudo smartctl -l devstat /dev/sda
```

If supported by the drive, this provides additional device statistics.

---

## 15. Additional diagnostics for NVMe drives

Although `/dev/sda` normally identifies a SATA or USB-connected block device, some external enclosures contain NVMe drives.

NVMe devices typically appear as:

```text
/dev/nvme0n1
```

Install the NVMe management tools:

```bash
sudo apt install nvme-cli
```

List NVMe devices:

```bash
sudo nvme list
```

Retrieve the SMART information:

```bash
sudo nvme smart-log /dev/nvme0
```

Display the error log:

```bash
sudo nvme error-log /dev/nvme0
```

Retrieve device identification:

```bash
sudo nvme id-ctrl /dev/nvme0
```

Important NVMe SMART fields include:

| Field | Description |
| --- | --- |
| critical_warning | Indicates critical health warnings. |
| temperature | Current temperature. |
| available_spare | Available spare capacity. |
| available_spare_threshold | Spare capacity warning threshold. |
| percentage_used | Estimated endurance consumed. |
| media_errors | Number of media and data integrity errors. |
| num_err_log_entries | Number of error-log entries. |
| power_on_hours | Total operating hours. |
| unsafe_shutdowns | Number of unsafe shutdowns. |

A nonzero `critical_warning`, increasing `media_errors`, or unexpectedly low available spare capacity warrants investigation.

---

## 16. Monitoring disk health continuously with smartd

For a server that operates continuously, periodic manual checks are not sufficient.

The `smartd` daemon can monitor SMART-compatible devices automatically.

### 16.1 Check the smartd service

```bash
sudo systemctl status smartmontools
```

Depending on the package configuration, the service is named `smartmontools.service`.

### 16.2 Enable automatic monitoring

```bash
sudo systemctl enable --now smartmontools
```

Verify:

```bash
sudo systemctl status smartmontools
```

### 16.3 Configure smartd

The configuration file is:

```text
/etc/smartd.conf
```

Edit it:

```bash
sudo nano /etc/smartd.conf
```

An example configuration for `/dev/sda`:

```text
/dev/sda -a -o on -S on -s (S/../.././02|L/../../7/03) -m root
```

Explanation:

| Option | Description |
| --- | --- |
| `/dev/sda` | Device to monitor. |
| `-a` | Monitor all supported SMART attributes and logs. |
| `-o on` | Enable automatic offline testing, if supported. |
| `-S on` | Enable SMART attribute autosave. |
| `-s` | Define scheduled self-tests. |
| `S/../.././02` | Run a short test daily between 02:00 and 03:00. |
| `L/../../7/03` | Run a long test on Sundays between 03:00 and 04:00. |
| `-m root` | Send warnings to the local root account, subject to mail configuration. |

**Note:** The monitoring configuration is an example. Adjust the test schedule according to the drive's workload and availability requirements.

The host must have a working local or external mail delivery configuration for email notifications to be delivered.

### 16.4 Restart the monitoring service

After modifying the configuration:

```bash
sudo systemctl restart smartmontools
```

Check its status:

```bash
sudo systemctl status smartmontools
```

Inspect monitoring logs:

```bash
sudo journalctl -u smartmontools
```

Follow the logs:

```bash
sudo journalctl -u smartmontools -f
```

### 16.5 Verify smartd configuration

Run:

```bash
sudo smartd -q onecheck
```

This performs a single monitoring check and exits.

Consult the installed manual for configuration details:

```bash
man smartd
man smartd.conf
```

---

## 17. Understanding smartctl exit codes

The `smartctl` command returns an exit status that can be used in shell scripts.

Check the exit status:

```bash
sudo smartctl -H /dev/sda
echo $?
```

A return value of `0` means that no monitored error conditions were reported by the command.

For ATA devices, the return status is a bitmask.

Important bits include:

| Bit | Value | Meaning |
| --- | --- :| --- |
| 0 | 1 | Command-line parsing error. |
| 1 | 2 | Device open or identification error. |
| 2 | 4 | SMART command or data checksum error. |
| 3 | 8 | SMART health status indicates disk failure. |
| 4 | 16 | A prefailure attribute is at or below its threshold. |
| 5 | 32 | An attribute has previously been at or below its threshold. |
| 6 | 64 | The device error log contains records. |
| 7 | 128 | The device self-test log contains errors. |

Multiple conditions can be combined.

For example, an exit code of `8` indicates that bit 3 is set.

An exit code of `40` means that bits 3 and 5 are set.

**Important:** A nonzero exit code does not necessarily mean that the disk is currently failing. It may indicate historical errors or problems accessing the device. Always inspect the complete output.

---

## 18. Example comprehensive health-check procedure

The following sequence is suitable for a manual inspection of an external HDD.

Assumptions:

- Ubuntu Server 26.04 LTS.
- External disk: `/dev/sda`.
- SMART support is available.
- The disk is not showing severe failure symptoms.

### Step 1: Identify the device

```bash
lsblk -o NAME,PATH,TYPE,SIZE,MODEL,SERIAL,TRAN,FSTYPE,MOUNTPOINTS
```

### Step 2: Retrieve device information

```bash
sudo smartctl -i /dev/sda
```

### Step 3: Check SMART health

```bash
sudo smartctl -H /dev/sda
```

### Step 4: Retrieve complete SMART information

```bash
sudo smartctl -x /dev/sda
```

### Step 5: Inspect important SMART attributes

```bash
sudo smartctl -A /dev/sda
```

Pay particular attention to:

- Reallocated sectors.
- Current pending sectors.
- Offline uncorrectable sectors.
- Reported uncorrectable errors.
- Temperature.
- Interface CRC errors.

### Step 6: Inspect historical errors

```bash
sudo smartctl -l error /dev/sda
```

### Step 7: Inspect previous self-tests

```bash
sudo smartctl -l selftest /dev/sda
```

### Step 8: Run a short self-test

```bash
sudo smartctl -t short /dev/sda
```

### Step 9: Inspect the short self-test result

```bash
sudo smartctl -l selftest /dev/sda
```

### Step 10: Run an extended self-test

```bash
sudo smartctl -t long /dev/sda
```

Allow the test to complete.

### Step 11: Inspect the extended self-test result

```bash
sudo smartctl -l selftest /dev/sda
```

### Step 12: Check kernel messages

```bash
sudo journalctl -k | grep -Ei 'sda|I/O error|medium error|uncorrect|reset|timeout'
```

### Step 13: Check filesystem consistency

After identifying the relevant filesystem and unmounting it, use the appropriate filesystem-specific diagnostic utility.

For ext4:

```bash
sudo e2fsck -fn /dev/sda1
```

### Step 14: Save a final diagnostic report

```bash
sudo smartctl -x /dev/sda > "disk-health-$(date +%Y%m%d-%H%M%S).txt"
```

This produces a report that can be archived and compared against future health checks.

---

## 19. Recommended health-check interpretation

The following table provides general guidance for interpreting diagnostic results.

| Diagnostic result | Assessment | Recommended action |
| --- | --- | --- |
| SMART PASSED, no suspicious attributes, no errors | Good | Continue regular monitoring. |
| SMART PASSED, small historical error count | Investigate | Compare historical values and monitor trends. |
| Reallocated sectors greater than zero | Warning | Back up important data and monitor closely. |
| Increasing reallocated sectors | Critical warning | Plan replacement and prioritize backup. |
| Pending sectors greater than zero | Warning | Back up data and investigate immediately. |
| Offline uncorrectable sectors greater than zero | Warning | Investigate possible media deterioration. |
| SMART overall health FAILED | Critical | Treat the drive as potentially failing; prioritize recovery. |
| Self-test reports read failure | Critical | Investigate affected LBAs and prioritize data recovery. |
| Repeated kernel I/O errors | Critical | Investigate drive, cabling, enclosure, and controller. |
| Increasing CRC errors | Warning | Investigate the connection and cabling. |
| Repeated USB resets or disconnects | Warning | Investigate USB cable, enclosure, and power supply. |
| Excessive temperature | Warning | Improve cooling and investigate operating conditions. |

### General recommendations

- A single nonzero SMART attribute should not automatically trigger a replacement decision without considering its meaning and the manufacturer's specifications.
- Increasing error counts are generally more concerning than stable historical counts.
- A failed SMART self-test should be taken seriously.
- A SMART PASSED result is not a substitute for regular backups.
- If important data is stored on a suspected failing drive, avoid unnecessary full-disk scans and repeated tests.

---

## 20. Example shell script for a basic disk health check

The following script performs a basic diagnostic check and saves the results to a timestamped report.

It does not start self-tests, repair filesystems, or perform destructive operations.

Save it as:

```text
disk-healthcheck.sh
```

Script:

```bash
#!/usr/bin/env bash

set -u

DISK="${1:-/dev/sda}"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
REPORT="disk-health-${TIMESTAMP}.txt"

if [[ ! -b "$DISK" ]]; then
    echo "ERROR: Device does not exist: $DISK"
    exit 1
fi

echo "Disk Health Check"
echo "Device: $DISK"
echo "Date: $(date)"
echo

{
    echo "========================================"
    echo "DISK HEALTH REPORT"
    echo "========================================"

    echo
    echo "Device identification:"
    lsblk -o NAME,PATH,TYPE,SIZE,MODEL,SERIAL,TRAN "$DISK"

    echo
    echo "SMART device information:"
    smartctl -i "$DISK"

    echo
    echo "SMART health status:"
    smartctl -H "$DISK"

    echo
    echo "SMART attributes:"
    smartctl -A "$DISK"

    echo
    echo "SMART error log:"
    smartctl -l error "$DISK"

    echo
    echo "SMART self-test log:"
    smartctl -l selftest "$DISK"

    echo
    echo "Complete SMART report:"
    smartctl -x "$DISK"

} 2>&1 | tee "$REPORT"

echo
echo "Report saved to: $REPORT"
```

Make the script executable:

```bash
chmod +x disk-healthcheck.sh
```

Run it:

```bash
sudo ./disk-healthcheck.sh /dev/sda
```

Alternatively, use a persistent device identifier:

```bash
sudo ./disk-healthcheck.sh /dev/disk/by-id/usb-Seagate_Expansion_ABC123
```

**Note:** The script collects several reports, including the complete SMART report. It may therefore repeat some diagnostic queries.

It is intended as a simple manual health-check utility, not a replacement for continuous monitoring or a dedicated monitoring system.

---

## 21. Recommended monitoring frequency

For an external HDD used as a backup or general-purpose storage device, the following schedule is a reasonable starting point.

| Diagnostic | Suggested frequency |
| --- | --- |
| SMART overall health | Daily |
| SMART attributes | Daily |
| Temperature | Daily |
| SMART error log | Weekly |
| Short self-test | Weekly |
| Extended self-test | Monthly |
| Kernel error review | Weekly |
| Filesystem consistency check | As needed, while unmounted |

These frequencies should be adapted to the drive's workload, operating environment, and manufacturer's recommendations.

For critical storage, continuous monitoring through `smartd` is recommended.

---

## 22. References and further documentation

The following references provide additional information about the tools and technologies described in this document.

### Ubuntu packages

- [Ubuntu 26.04 LTS - smartmontools](https://packages.ubuntu.com/resolute/smartmontools)
- [Ubuntu 26.04 LTS - nvme-cli](https://packages.ubuntu.com/resolute/nvme-cli)
- [Ubuntu 26.04 LTS - e2fsprogs](https://packages.ubuntu.com/resolute/e2fsprogs)

### SMART and disk diagnostics

- [smartmontools project](https://www.smartmontools.org/)
- [smartctl manual - Ubuntu 26.04](https://manpages.ubuntu.com/manpages/resolute/man8/smartctl.8.html)
- [smartd manual](https://manpages.debian.org/unstable/smartmontools/smartd.8.en.html)
- [smartd.conf documentation](https://manpages.debian.org/unstable/smartmontools/smartd.conf.5.en.html)

### Linux storage utilities

- `man lsblk`
- `man fdisk`
- `man udevadm`
- `man journalctl`
- `man e2fsck`
- `man badblocks`
- `man hdparm`

### NVMe diagnostics

- [nvme-cli project](https://github.com/linux-nvme/nvme-cli)

---

## 23. Conclusion

For checking the health of an external HDD on Ubuntu Server 26.04 LTS, the most important commands are:

| Purpose | Command |
| --- | --- |
| Identify the drive | `lsblk -o NAME,PATH,SIZE,MODEL,SERIAL,TRAN` |
| Retrieve SMART support information | `sudo smartctl -i /dev/sda` |
| Check overall health | `sudo smartctl -H /dev/sda` |
| Retrieve complete health information | `sudo smartctl -x /dev/sda` |
| Inspect SMART attributes | `sudo smartctl -A /dev/sda` |
| Inspect error history | `sudo smartctl -l error /dev/sda` |
| Inspect self-test history | `sudo smartctl -l selftest /dev/sda` |
| Start a short self-test | `sudo smartctl -t short /dev/sda` |
| Start an extended self-test | `sudo smartctl -t long /dev/sda` |
| Inspect kernel errors | `sudo journalctl -k` |
| Perform a read-only bad-block scan | `sudo badblocks -sv /dev/sda` |
| Monitor SMART continuously | `sudo systemctl enable --now smartmontools` |

**Recommended minimum health check:**

```bash
sudo smartctl -x /dev/sda
sudo smartctl -t short /dev/sda
sudo smartctl -l selftest /dev/sda
sudo journalctl -k | grep -Ei 'sda|I/O error|medium error|uncorrect|reset|timeout'
```

For a more thorough assessment, also run an extended SMART self-test and examine the filesystem while it is unmounted.

Ultimately, the combination of SMART attributes, internal self-tests, kernel logs, and filesystem diagnostics provides a much more reliable assessment than checking the overall SMART health status alone.

Regular backups remain essential, regardless of the reported disk health.
