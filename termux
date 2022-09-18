#!/usr/bin/env bash

# termux-proot - A sandboxed, 2nd termux, isolated or jailed termux environment with proot
# https://git.io/termux-proot

# Dependency check
DEPS=("curl" "unzip" "proot")
for DEP in "${DEPS[@]}"; do
	if ! hash "$DEP" 2>/dev/null; then
		echo "$DEP is not installed. Aborting." && exit 6
	fi
done

[ -z "$TERMUX_SANDBOX_PATH" ] && export TERMUX_SANDBOX_PATH=${HOME:-/home}/.termux-fs

[ -z "$TERMUX_ANDROID_BASE" ] && export TERMUX_ANDROID_BASE=${HOME:-/home}/.android-base

[ -z "$TERMUX_SANDBOX_APPPATH" ] && export TERMUX_SANDBOX_APPPATH=/data/data/com.termux

[ "$(id -u)" = "0" ] && exec echo -e "You shouldn't execute this script as root, don't you?\nSo you're trying to harm your own Device.\n\nAbility to use termux-proot with root (even fake) is disabled permanently.\nJust do it in your real termux, Or use chroot. But don't blame me for broken device OK?\n\nIf you mean to simulate fake root, add \"-0\" in TERMUX_SANDBOX_PROOT_OPTIONS variable."

# Installation

if [ ! -d "$TERMUX_ANDROID_BASE" ] || [ -z "$(ls -A $TERMUX_ANDROID_BASE)" ]; then
	# Figure the system architecture
	case $(uname -m) in
		x86_64|x86|i686)  ARCH=x86 ;;
		arm|aarch64|armv7l|armv8l)  ARCH=arm ;;
		*)  exec echo "Unsupported Architecture: $(uname -m)" ;;
	esac
	if [ ! -d "${TMPDIR:-/tmp}"/.termux-docker ]; then
		echo -n "[1/5] Downloading Android base system...."
		if ! git clone --depth=1 -q https://github.com/termux/termux-docker "${TMPDIR:-/tmp}"/.termux-docker; then
			echo "Fail"
			exit 6
		fi

		curl -sLo "${TMPDIR:-/tmp}"/.termux-docker/system/$ARCH/etc/static-dns-hosts.txt https://raw.githubusercontent.com/termux/termux-docker/master/static-dns-hosts.txt

		echo "Done"
	fi

	echo -n "[2/5] Preparing Android base system.... "

	current_dir=$(pwd)
	cd "${TMPDIR:-/tmp}"/.termux-docker/system/$ARCH/bin

	for i in "[" "[[" ar arch arp arping ascii ash awk base32 base64 basename bbconfig bc blkdiscard blkid blockdev brctl bunzip2 bzcat bzip2 cal cat chattr chgrp chmod chown chpst chroot chrt cksum clear cmp comm cp cpio crc32 crond crontab cut date dc dd diff dirname dmesg dnsdomainname dos2unix du echo ed egrep env envdir expand expr factor fallocate false fdisk fgrep find findfs flock fold free fstrim fsync ftpd ftpget ftpput fuser getopt grep groups gunzip gzip hd head hexdump hexedit hostname httpd hwclock id ifconfig inotifyd install ionice iostat ip ipaddr ipcalc iplink ipneigh iproute iprule iptunnel kill killall killall5 less link linux32 linux64 ln losetup ls lsattr lsof lspci lsscsi lsusb lzcat lzma lzop lzopcat makemime md5sum mkdir mkdosfs mke2fs mkfifo mkfs.ext2 mkfs.vfat mknod mkswap mktemp more mount mountpoint mpstat mv nc netcat netstat nice nl nmeter nohup nproc nsenter od paste patch pgrep pidof ping pipe_progress pivot_root pkill pmap popmaildir printenv printf ps pscan pstree pwd pwdx rdate readlink realpath reformime renice reset resize rev rfkill rm rmdir route run-parts runsv runsvdir rx script scriptreplay sed sendmail seq setarch setsid setuidgid sha1sum sha256sum sha3sum sha512sum shred shuf sleep smemcap softlimit sort split start-stop-daemon stat strings stty sum sv svlogd switch_root sync sysctl tac tail tar tc tcpsvd tee telnet telnetd test tftp tftpd time timeout top touch tr traceroute true truncate tty ttysize tunctl tune2fs udpsvd umount uname uncompress unexpand uniq unix2dos unlink unlzma unlzop unshare unxz unzip uptime usleep uudecode uuencode vconfig vi watch wc wget which whoami whois xargs xxd xz xzcat yes zcat; do
		ln -s busybox $i
	done

	rm -rf "$TERMUX_ANDROID_BASE"
	mv "${TMPDIR:-/tmp}"/.termux-docker/system/$ARCH ${HOME:-~}/.android-base
	cd "$current_dir"
	rm -rf "${TMPDIR:-/tmp}"/.termux-docker/

	echo "Done"
fi

if [ ! -d "$TERMUX_SANDBOX_PATH" ] || [ -z "$(ls -A "$TERMUX_SANDBOX_PATH")" ]; then
	if [ ! -f "${TMPDIR:-/tmp}"/.termux-rootfs.zip ]; then
		echo "[3/5] Downloading Latest Termux Bootstrap...."
		curl -Lo "${TMPDIR:-/tmp}"/.termux-rootfs.zip https://github.com/termux/termux-packages/releases/download/"$(curl -s "https://api.github.com/repos/termux/termux-packages/releases/latest" | grep -Po '"tag_name": "\K.*?(?=")')"/bootstrap-"$(uname -m)".zip
	fi
	mkdir "$TERMUX_SANDBOX_PATH" && cd "$TERMUX_SANDBOX_PATH" || exit 1

	echo -n "[4/5] Extracting.... "
	if ! unzip -q "${TMPDIR:-/tmp}"/.termux-rootfs.zip; then
		echo "Failed extracting bootstrap!"
		proot -0 rm -rf "$TERMUX_SANDBOX_PATH"
		rm "${TMPDIR:-/tmp}"/.termux-rootfs.zip
		exit 6
	fi
	echo "Done"
fi

if [ -f "$TERMUX_SANDBOX_PATH/SYMLINKS.txt" ]; then
	cd $TERMUX_SANDBOX_PATH
	echo -n "[5/5] Symlinking.... "
	while read -r p; do
		IFS="←" read -r FILE DEST <<< "$p"
		! [ -h "$DEST" ] && proot -0 ln -s "$FILE" "$DEST"
	done < SYMLINKS.txt && rm SYMLINKS.txt
	echo "Done"
fi

ARGS=("proot")
ARGS+=("-r $TERMUX_ANDROID_BASE $TERMUX_SANDBOX_PROOT_OPTIONS")
ARGS+=("-w $TERMUX_SANDBOX_APPPATH/files/home")

# Make sure that some common directory like /home is there.
# Otherwise, we recreate the directory
for dir in $TERMUX_SANDBOX_PATH/var/cache $TERMUX_SANDBOX_PATH/home $TERMUX_ANDROID_BASE/sdcard; do
	! [ -d "$dir" ] && mkdir -p "$dir"
done

# Bind some common path
for bind in /dev /proc /sys $TERMUX_ANDROID_BASE:/system $TERMUX_ANDROID_BASE:/vendor /apex /linkerconfig/ld.config.txt /property_context $TERMUX_SANDBOX_PATH:$TERMUX_SANDBOX_APPPATH/files/usr $TERMUX_SANDBOX_PATH/var/cache:$TERMUX_SANDBOX_APPPATH/cache $TERMUX_SANDBOX_PATH/home:$TERMUX_SANDBOX_APPPATH/files/home; do
	[ -d "$bind" ] || [ -f "$bind" ] || grep -q $TERMUX_SANDBOX_APPPATH <<< "$bind" || grep -q $TERMUX_ANDROID_BASE <<< "$bind" && ARGS+=("-b $bind")
done

ARGS+=("$TERMUX_SANDBOX_APPPATH/files/usr/bin/env -i")
ARGS+=("HOME=$TERMUX_SANDBOX_APPPATH/files/home")
ARGS+=("PATH=$TERMUX_SANDBOX_APPPATH/files/usr/bin")
ARGS+=("TERM=${TERM:-xterm-256color}")
ARGS+=("COLORTERM=${COLORTERM:-truecolor}")
ARGS+=("ANDROID_DATA=/data")
ARGS+=("ANDROID_ROOT=/system")
ARGS+=("EXTERNAL_STORAGE=/sdcard")
ARGS+=("LANG=${LANG:-en_US.UTF-8}")
ARGS+=("LD_LIBRARY_PATH=$TERMUX_SANDBOX_APPPATH/files/usr/lib")
[ -x "$TERMUX_SANDBOX_APPPATH/files"/usr/lib/libtermux-exec.so ] && ARGS+=("LD_PRELOAD=$TERMUX_SANDBOX_APPPATH/files/usr/lib/libtermux-exec.so")
ARGS+=("TERMUX_VERSION=${TERMUX_VERSION:-0.118}")
ARGS+=("PREFIX=$TERMUX_SANDBOX_APPPATH/files/usr")
ARGS+=("TMPDIR=$TERMUX_SANDBOX_APPPATH/files/usr/tmp")
ARGS+=("$TERMUX_SANDBOX_ENV")

cmd="$*"

# Unset Preload Library.
unset LD_PRELOAD

${ARGS[@]} /system/bin/update-static-dns
# shellcheck disable=SC2068
case "$cmd" in
"")
	exec ${ARGS[@]} login
	;;
*)
	exec ${ARGS[@]} login -c "$cmd"
	;;
esac
