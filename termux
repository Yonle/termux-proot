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
	if [ ! -f "${TMPDIR:-/tmp}"/.android-base.tar.gz ]; then
		echo "[1/6] Downloading Android base system...."
		curl -Lo "${TMPDIR:-/tmp}"/.android-base.tar.gz https://github.com/Yonle/termux-proot/archive/refs/tags/system-$ARCH.tar.gz
	fi
	mkdir -p "$TERMUX_ANDROID_BASE" && cd "$TERMUX_ANDROID_BASE" || exit 1
	echo -n "[2/6] Extracting Android base system.... "
	if ! tar -xzf "${TMPDIR:-/tmp}"/.android-base.tar.gz ; then
		echo "Failed to extract Base system"
		rm -rf "$TERMUX_ANDROID_BASE"
		rm "${TMPDIR:-/tmp}"/.android-base.tar.gz
		exit 6
	fi
	mv termux-proot-system-$ARCH/* . && rm -rf termux-proot-system-$ARCH
	echo "Done"
fi

if [ ! -d "$TERMUX_SANDBOX_PATH" ] || [ -z "$(ls -A "$TERMUX_SANDBOX_PATH")" ]; then
	if [ ! -f "${TMPDIR:-/tmp}"/.termux-rootfs.zip ]; then
		echo "[3/6] Downloading Latest Termux Bootstrap...."
		curl -Lo "${TMPDIR:-/tmp}"/.termux-rootfs.zip https://github.com/termux/termux-packages/releases/download/"$(curl -s "https://api.github.com/repos/termux/termux-packages/releases/latest" | grep -Po '"tag_name": "\K.*?(?=")')"/bootstrap-"$(uname -m)".zip
	fi
	mkdir "$TERMUX_SANDBOX_PATH" && cd "$TERMUX_SANDBOX_PATH" || exit 1

	echo -n "[4/6] Extracting.... "
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
	echo -n "[5/6] Symlinking.... "
	while read -r p; do
		IFS="←" read -r FILE DEST <<< "$p"
		proot -0 ln -s "$FILE" "$DEST"
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

echo "[6/6] Updating Static hosts...."
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
