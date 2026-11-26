# Disk & Server Health Check


It grades every spinning disk behind the RAID controller as HEALTHY, WATCH, WARNING, FAILING, DEAD or TAMPERED, and writes a text and HTML report.

What it checks:
- SMART and SCSI error logs: grown defects, pending sectors, read errors, failed self-tests
- Reset power-on hours (on Seagate disks it reads the FARM log, which keeps the real hours)
- Long self-tests and a full read of the array
- Controller and iDRAC hardware event history

It never creates, deletes or changes a RAID virtual disk. The only controller jobs it can start are Dell's built-in non-destructive checks (Patrol Read and Consistency Check).

## Running it

The program is encrypted and needs a password to run.

```bash
sudo python3 disk_health_check_secure.py
```
