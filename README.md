petardfs
========

PetardFS - a FUSE filessytem for injecting intentional errors (e.g. for testing)

Originally developed by Ben Martin (http://sourceforge.net/projects/witme/files/petardfs/0.0.2/).

With no configuration petardfs takes a base filesystem and exposes it through FUSE.

An XML configuration file is used to tell petardfs which files to report errors
for and what error code to use. 

For example, foo.txt can have an EIO error at bytes 34 to 37. There is explicit 
support for errors such as EAGAIN and EINTR where petardfs will only report such 
transient errors a nominated number of times, handy for testing applications support 
such IO conditions gracefully.

XML Configuration File
======================

An example configuration file can be found within the source tree at:  testsuite/sampledata/config-simple-filesystem-simple-error-in-read-file1.xml

A basic configuation might look like:
```xml
<!DOCTYPE petardfs-config [
<!ENTITY EPERM   "1">
<!ENTITY ENOENT   "2">
<!ENTITY ESRCH   "3">
<!ENTITY EINTR   "4">
<!ENTITY EIO   "5">
<!ENTITY ENXIO   "6">
<!ENTITY E2BIG   "7">
<!ENTITY ENOEXEC   "8">
<!ENTITY EBADF   "9">
<!ENTITY ECHILD  "10">
<!ENTITY EAGAIN  "11">
<!ENTITY ENOMEM  "12">
<!ENTITY EACCES  "13">
<!ENTITY EFAULT  "14">
<!ENTITY ENOTBLK  "15">
<!ENTITY EBUSY  "16">
<!ENTITY EEXIST  "17">
<!ENTITY EXDEV  "18">
<!ENTITY ENODEV  "19">
<!ENTITY ENOTDIR  "20">
<!ENTITY EISDIR  "21">
<!ENTITY EINVAL  "22">
<!ENTITY ENFILE  "23">
<!ENTITY EMFILE  "24">
<!ENTITY ENOTTY  "25">
<!ENTITY ETXTBSY  "26">
<!ENTITY EFBIG  "27">
<!ENTITY ENOSPC  "28">
<!ENTITY ESPIPE  "29">
<!ENTITY EROFS  "30">
<!ENTITY EMLINK  "31">
<!ENTITY EPIPE  "32">
<!ENTITY EDOM  "33">
<!ENTITY ERANGE  "34">
]>
<petardfs-config>
   <errors>
     <read>
       <error path="/read-eintr-10.txt">
         <n start-offset="1" end-offset="1" error-code="&EINTR;" times="10"/>
       </error>
       <error path="/read-eio.txt">
         <n start-offset="2" end-offset="2" error-code="&EIO;"/>
       </error>
     </read>
     <write>
       <error path="/write-eio.txt">
         <n start-offset="2" end-offset="2" error-code="&EIO;"/>
       </error>
     </write>
     <open>
       <error path="/open-eio.txt" error-code="&EIO;" />
     </open>
   </errors>
</petardfs-config>
```

The `read`, `write`, and `open` elements represent the filesystem operation on which the errors 
contained within should be injected. Some operations (e.g. `read` and `write`) require an `n` element 
within the `error` element to specify which `error-code` should be introduced at which ranges of bytes 
(from `start-offset` to `end-offset`), while others (e.g. `open`) do not require that and can have 
the `error-code` specified in the `error` element itself. 

The full list of operations which are supported for error injection are: `read`, `write`, `fsync`, 
`mkdir`, `symlink`, `unlink`, `rmdir`, `rename`, `link`, `chmod`, `chown`, `ftruncate`, `utime`, and 
`open`.

The list of error codes supported should be given as XML entity declarations at the top of each XML file. 
The full list is given in the above example. 

The `path` within each `error` element is relative to the mount point of the filesystem. 

Usage Example
=============

The following example tests whether the `diff` and `cat` programs handle the fatal EIO error by erroring out 
and the EINTR error by retrying or reporting the error. 

```bash
# module add hgi/xerces-c/latest hgi/petardfs/latest
# mkdir /tmp/petard-data
# seq 1 100000 > /tmp/petard-data/testfile-fatal
# seq 1 100000 > /tmp/petard-data/testfile-retry
# mkdir /tmp/petard-mnt
# echo '<!DOCTYPE petardfs-config [
<!ENTITY EPERM   "1">
<!ENTITY ENOENT   "2">
<!ENTITY ESRCH   "3">
<!ENTITY EINTR   "4">
<!ENTITY EIO   "5">
<!ENTITY ENXIO   "6">
<!ENTITY E2BIG   "7">
<!ENTITY ENOEXEC   "8">
<!ENTITY EBADF   "9">
<!ENTITY ECHILD  "10">
<!ENTITY EAGAIN  "11">
<!ENTITY ENOMEM  "12">
<!ENTITY EACCES  "13">
<!ENTITY EFAULT  "14">
<!ENTITY ENOTBLK  "15">
<!ENTITY EBUSY  "16">
<!ENTITY EEXIST  "17">
<!ENTITY EXDEV  "18">
<!ENTITY ENODEV  "19">
<!ENTITY ENOTDIR  "20">
<!ENTITY EISDIR  "21">
<!ENTITY EINVAL  "22">
<!ENTITY ENFILE  "23">
<!ENTITY EMFILE  "24">
<!ENTITY ENOTTY  "25">
<!ENTITY ETXTBSY  "26">
<!ENTITY EFBIG  "27">
<!ENTITY ENOSPC  "28">
<!ENTITY ESPIPE  "29">
<!ENTITY EROFS  "30">
<!ENTITY EMLINK  "31">
<!ENTITY EPIPE  "32">
<!ENTITY EDOM  "33">
<!ENTITY ERANGE  "34">
]>
<petardfs-config>
   <errors>
     <read>
       <error path="/testfile-retry">
         <n start-offset="10000" end-offset="10000" error-code="&EINTR;" times="10"/>
       </error>
       <error path="/testfile-fatal">
         <n start-offset="2" end-offset="2" error-code="&EIO;"/>
       </error>
     </read>
   </errors>
</petardfs-config>' > /tmp/petard-errors.xml
# petardfs -e /tmp/petard-errors.xml -u /tmp/petard-data /tmp/petard-mnt
t:/tmp/petard-mnt
# diff /tmp/petard-mnt/testfile-fatal /tmp/petard-data/testfile-fatal
diff: /tmp/petard-mnt/testfile-fatal: Input/output error
# cat /tmp/petard-mnt/testfile-fatal > /dev/null
cat: /tmp/petard-mnt/testfile-fatal: Input/output error
# diff /tmp/petard-mnt/testfile-retry /tmp/petard-data/testfile-retry
diff: /tmp/petard-mnt/testfile-retry: Interrupted system call
# cat /tmp/petard-mnt/testfile-retry > /tmp/petard-testfile-retry
# diff /tmp/petard-testfile-retry /tmp/petard-data/testfile-retry
# fusermount -u /tmp/petard-mnt
```

In the above example, we see that the `diff` program reports error conditions for both EIO and EINTR errors, while the `cat` program reports the EIO error but retries the read repeatedly when it gets the EINTR error until it succeeeds in successfully reading the data. We can use the `strace` command to verify that the error is, in fact, occurring:

```bash
# petardfs -e /tmp/petard-errors.xml -u /tmp/petard-data /tmp/petard-mnt
t:/tmp/petard-mnt
# (strace cat /tmp/petard-mnt/testfile-retry > /dev/null) 2>&1 | grep read | head -n 12
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\200\30\2\0\0\0\0\0"..., 832) = 832
read(3, "1\n2\n3\n4\n5\n6\n7\n8\n9\n10\n11\n12\n13\n14"..., 32768) = 8192
read(3, 0x15a5000, 32768)               = -1 EINTR (Interrupted system call)
read(3, 0x15a5000, 32768)               = -1 EINTR (Interrupted system call)
read(3, 0x15a5000, 32768)               = -1 EINTR (Interrupted system call)
read(3, 0x15a5000, 32768)               = -1 EINTR (Interrupted system call)
read(3, 0x15a5000, 32768)               = -1 EINTR (Interrupted system call)
read(3, 0x15a5000, 32768)               = -1 EINTR (Interrupted system call)
read(3, 0x15a5000, 32768)               = -1 EINTR (Interrupted system call)
read(3, 0x15a5000, 32768)               = -1 EINTR (Interrupted system call)
read(3, "\n1861\n1862\n1863\n1864\n1865\n1866\n1"..., 32768) = 32768
read(3, "14\n8415\n8416\n8417\n8418\n8419\n8420"..., 32768) = 32768
# fusermount -u /tmp/petard-mnt
```

