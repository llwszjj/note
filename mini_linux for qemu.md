制作mini_linux

1) 下载linux kernel源码
以linux-4.8.7为例，
```js
make menuconfig 选择你需要的modules
make -j64 
sudo make modules_install 会拷贝到lib/kernel-version/modules/
make install 生成system.map/vmlinux/intramfs，拷贝至/boot
修改grub
。。。
``` 
其实上面的步骤是针对在host上更新kernel的。

2) qemu启动
```js
/home/xxx/qemu/x86_64-softmmu/qemu-system-x86_64 \
    -machine q35,accel=kvm,usb=off \
    -cpu host \
    -m 1G \
    -object memory-backend-file,id=mem,size=1G,mem-path=/dev/hugepages/libvirt/qemu,share=on \
    -numa node,memdev=mem \
    -realtime mlock=off -smp 1,sockets=1,cores=1,threads=1 -nographic \
    -rtc base=utc,clock=vm,driftfix=slew -no-reboot -boot strict=on \
    -kernel ./linux-4.8.7/arch/x86/boot/bzImage \
    -initrd /boot/initramfs-4.8.7.img \
    -append 'console=ttyS0 panic=1 no_timer_check'
```  
尝试用上面命令启动这个kernel版本的vm，报错systemd，这是因为make install后生成的initramfs.img是针对host的，linux启动的时候init会由systemd接管。看来这种initramfs.img对于vm不可行，所以我们需要自己做一个initramfs

3) busybox制作rootfs
```js
mkdir rootfs
mkdir -p ./{bin,dev,etc,lib,lib64,mnt/root,proc,root,sbin,sys}
cp ${kerel_source}
make modules_install INSTALL_MOD_PATH=rootfs
```
下载busybox源码
```js
make menuconfig && make -j64
make install 会生成_install目录，里面包含了bin sbin等最小命令集，拷贝到./rootfs下面

ldd gcc查看gcc需要哪些so文件
        linux-vdso.so.1 =>  (0x00007fffc4f8f000)
        libm.so.6 => /lib64/libm.so.6 (0x00007fae5e629000)
        libc.so.6 => /lib64/libc.so.6 (0x00007fae5e267000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fae5e93b000)
需要将这些so文件拷贝到rootfs/lib64/

对于init

#cat init
#!/bin/busybox sh

# Mount the /proc and /sys filesystems.
mount -t proc none /proc
mount -t sysfs none /sys

echo '/sbin/mdev' > /proc/sys/kernel/hotplug
exec /bin/sh
``` 

echo '/sbin/mdev' > /proc/sys/kernel/hotplug 用于类似于udev来接管hotplug

```js
chmod +x init
```  
生成镜像文件
```js
find . -print0 | cpio --null -ov --format=newc | gzip -9 > /boot/custom-initramfs.cpio.gz
```  

4) qemu启动

```js
/home/xxx/qemu/x86_64-softmmu/qemu-system-x86_64 \
    -machine q35,accel=kvm,usb=off \
    -cpu host \
    -m 1G \
    -object memory-backend-file,id=mem,size=1G,mem-path=/dev/hugepages/libvirt/qemu,share=on \
    -numa node,memdev=mem \
    -realtime mlock=off -smp 1,sockets=1,cores=1,threads=1 -nographic \
    -rtc base=utc,clock=vm,driftfix=slew -no-reboot -boot strict=on \
    -kernel ./linux-4.8.7/arch/x86/boot/bzImage \
    -initrd /boot/custom-initramfs.cpio.gz \
    -append 'console=ttyS0 panic=1 no_timer_check'
```  
这样就可以启动vm，启动后lsmod为空，是因为当前我们没有加载任何驱动，可以写个脚本来insmod这些driver
