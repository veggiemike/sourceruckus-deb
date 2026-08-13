SUBDIRS =
#SUBDIRS += src

top_docs = README COPYING AUTHORS TODO NOTES

EXTRA_DIST =
EXTRA_DIST += $(top_docs)

# The overall process is:
#
# - Build layer in sysroot
# - Create .sqsh of upper layer
# - Clean out sysroot
# - Prepare a new upper layer
# - Overlay-mount layers on sysroot for next image stage
#

CHROOT=chroot sysroot /usr/bin/env -i HOME=/root TERM=$$TERM PATH=$$PATH
DEBOOTSTRAP_DIR=$(abs_srcdir)/debootstrap
DEBOOTSTRAP=$(DEBOOTSTRAP_DIR)/debootstrap
#
# NOTE: We want apt install to be completely noninteractive with pre-seeded
#       values for all debconf questions.  My initial idea was to run through
#       interactively and then just copy the resulting debconf config.dat into
#       subsequent runs, but reading online kinda implies that that's the
#       "wrong" thing to do, and that I should use either DEBCONF_DB_FALLBACK
#       or DEBCONF_DB_OVERRIDE variables (set to either Pipe or File) to supply
#       values...  both of those methods were providing strange errors,
#       though...  so I tried just copying config.dat in directly as I'd
#       originally planned and it worked fine... so that's what I'm doing.
#
# NOTE: While the debconf-config.dat workaround does answer the
#       questions before they pop-up, it does NOT suppress other
#       interactive things (e.g., needrestart service restart dialog).
#       So we still also supply DEBIAN_FRONTEND=noninteractive.
#
APT_INSTALL=DEBIAN_FRONTEND=noninteractive apt install -y --no-install-recommends
APT_REMOVE=DEBIAN_FRONTEND=noninteractive apt remove -y --autoremove --purge
#
# FIXME: configure flag for this?  or maybe have configure check for
#        zstd support in mksquashfs?
#
#SQSH_COMP=-comp lz4 -Xhc
SQSH_COMP=-comp zstd


# NOTE: If --enable-devel was sepcified, we're building a devel-enabled base
#       image w/out 3rd party packages.
#
# FIXME: maybe for next time, we could rename es-devel to
#        es-bootstrap, then add a real finished es-devel (es
#        +compilation toolchain, maybe less stuff then bootstrap)
#
# FIXME: also, maybe later... we could rename workstation to
#        everything or desktop and add a workstation that has most of
#        everything but w/out 32bit multilib?
#
all: @IMG_LIST@

info:
	@echo PKGS_ES=\"$(PKGS_ES)\"
	@echo
	@echo PKGS_ES_3RDPARTY=\"$(PKGS_ES_3RDPARTY)\"
	@echo
	@echo PKGS_ESFILE=\"$(PKGS_ESFILE)\"
	@echo
	@echo PKGS_ESAUTH=\"$(PKGS_ESAUTH)\"
	@echo
	@echo PKGS_ESWEB=\"$(PKGS_ESWEB)\"
	@echo
	@echo PKGS_SERVER=\"$(PKGS_SERVER)\"
	@echo
	@echo PKGS_VMM=\"$(PKGS_VMM)\"
	@echo
	@echo PKGS_WORKSTATION=\"$(PKGS_WORKSTATION)\"
	@echo
	@echo PKGS_WORKSTATION_LOCAL\"$(PKGS_WORKSTATION_LOCAL)\"
	@echo
	@echo PKGS_FILESHARING=\"$(PKGS_FILESHARING)\"
	@echo
	@echo PKGS_ZFS=\"$(PKGS_ZFS)\"
	@echo
	@echo PKGS_COMPILATION=\"$(PKGS_COMPILATION)\"

sysroot/proc/self:
	mount proc -t proc $(dir $@)

sysroot/sys/block:
	mount sysfs -t sysfs $(dir $@)

sysroot/dev/block:
	mount --rbind /dev $(dir $@)
	mount --make-rslave $(dir $@)

# FIXME: i'm seeing /dev/pts missing errors here and
#        there... shouldn't that be getting pulled in via the /dev
#        bind mount?  seems to be happening only when playing around
#        in a built sysroot after building squashfs and doing things
#        like installing packages via apt.  Make sysroot-umount-vfs
#        sysroot-mount-vfs fixes it.
#
sysroot-mount-vfs: sysroot/proc/self sysroot/sys/block sysroot/dev/block #sysroot/dev/pts

sysroot-umount-vfs:
	umount -R sysroot/{proc,sys,dev} || echo already unmounted

sqsh_layer-es: es.sqsh
	$(MKDIR_P) $@
	mount es.sqsh $@

sqsh_layer-server: server.sqsh
	$(MKDIR_P) $@
	mount server.sqsh $@

sqsh_layer-vmm: vmm.sqsh
	$(MKDIR_P) $@
	mount vmm.sqsh $@

upper_layer:
	$(MKDIR_P) $@

# FIXME: i don't remember exactly why i do a umount -l here... and
#       it's causing problems occasionally with sysroot (when it's
#       overlay).  manually unmounting and trying again seems to work,
#       so i'm wonderingy if i should just drop the -l.
#
#       looks like i can't umount sysroot sometimes because gpg-agent
#       is still running?
#
sysroot-clean:
	$(MAKE) sysroot-umount-vfs
	umount sysroot || echo not mounted
	rm -rf --one-file-system sysroot
	umount upper_layer || echo not mounted
	rm -rf upper_layer
	umount sqsh_layer-vmm || echo not mounted
	rm -rf sqsh_layer-vmm
	umount sqsh_layer-server || echo not mounted
	rm -rf sqsh_layer-server
	umount sqsh_layer-es-web || echo not mounted
	rm -rf sqsh_layer-es-web
	umount sqsh_layer-es-auth || echo not mounted
	rm -rf sqsh_layer-es-auth
	umount sqsh_layer-es-file || echo not mounted
	rm -rf sqsh_layer-es-file
	umount sqsh_layer-es || echo not mounted
	rm -rf sqsh_layer-es

clean-local: sysroot-clean
	rm -rf .build


################################################################################
# PPA setup rules
#
# NOTE: These get actually applied to different layers via their build
#       rules, but they're all listed here for convenience.
#
# NOTE: We no longer use apt-key or the legacy trusted.gpg keyring
#       because they were deprecated in jammy, and for good reason.
#       See the DEPRECATION section in apt-key(8) for details.
#
#       https://askubuntu.com/questions/1286545/what-commands-exactly-should-replace-the-deprecated-apt-key
#
sysroot/etc/apt/sources.list.d/sourceruckus.list:
	$(MAKE) sysroot-mount-vfs
	wget -qO- https://sourceruckus.github.io/ppa/sourceruckus.key | $(CHROOT) tee /etc/apt/keyrings/sourceruckus.asc
	echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/sourceruckus.asc] https://sourceruckus.github.io/ppa resolute main" > $@

sysroot/etc/apt/sources.list.d/influxdb.list:
	$(MAKE) sysroot-mount-vfs
	wget -qO- https://repos.influxdata.com/influxdata-archive.key | $(CHROOT) tee /etc/apt/keyrings/influxdb.asc
	echo "deb [signed-by=/etc/apt/keyrings/influxdb.asc] https://repos.influxdata.com/debian stable main" > $@

sysroot/etc/apt/sources.list.d/grafana.list:
	$(MAKE) sysroot-mount-vfs
	wget -qO- https://packages.grafana.com/gpg.key | $(CHROOT) tee /etc/apt/keyrings/grafana.asc
	echo "deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main" > $@

sysroot/etc/apt/sources.list.d/wine.list:
	$(MAKE) sysroot-mount-vfs
	wget -qO- https://dl.winehq.org/wine-builds/winehq.key | $(CHROOT) tee /etc/apt/keyrings/wine.asc
	echo "deb [signed-by=/etc/apt/keyrings/wine.asc] https://dl.winehq.org/wine-builds/ubuntu resolute main" > $@

sysroot/etc/apt/sources.list.d/qgis.list:
	$(MAKE) sysroot-mount-vfs
	wget -qO- https://download.qgis.org/downloads/qgis-archive-keyring.gpg | $(CHROOT) tee /etc/apt/keyrings/qgis.gpg
	echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/qgis.gpg] https://qgis.org/ubuntu resolute main" > $@

# FIXME: This is a mess...  unifi requires mongodb >= 3.6.0 but <<
#        7.1.0.  I can only get an 8.0 ppa for noble, but I think I
#        can get a 7.0 ppa for jammy... let's try that for now.
#
sysroot/etc/apt/sources.list.d/mongodb.list:
	$(MAKE) sysroot-mount-vfs
#	wget -qO- https://www.mongodb.org/static/pgp/server-8.0.asc | $(CHROOT) tee /etc/apt/keyrings/mongodb.asc
#	echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/mongodb.asc] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" > $@
	wget -qO- https://www.mongodb.org/static/pgp/server-7.0.asc | $(CHROOT) tee /etc/apt/keyrings/mongodb.asc
	echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/mongodb.asc] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" > $@

sysroot/etc/apt/sources.list.d/ubiquiti.list:
	$(MAKE) sysroot-mount-vfs
	wget -qO- https://dl.ui.com/unifi/unifi-repo.gpg | $(CHROOT) tee /etc/apt/keyrings/unifi.gpg
	echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/unifi.gpg] https://www.ui.com/downloads/unifi/debian stable ubiquiti" > $@

# NOTE: We do this instead of using add-apt-repository to make things a bit
#	more consistent in here and to avoid multiple apt update calls.
sysroot/etc/apt/sources.list.d/linbit-drbd9.list:
	$(MAKE) sysroot-mount-vfs
	$(MKDIR_P) --mode=700 sysroot/root/.gnupg
	$(CHROOT) gpgconf --launch gpg-agent
	$(CHROOT) gpg --no-default-keyring --keyring /tmp/keybox.gpg --keyserver keyserver.ubuntu.com \
	  --recv-keys CC1B5A793C04BB3905AD837734893610CEAA9512
	$(CHROOT) gpg --no-default-keyring --keyring /tmp/keybox.gpg --export > sysroot/etc/apt/keyrings/linbit.gpg
	$(CHROOT) gpgconf --kill all
	rm sysroot/tmp/keybox.gpg
	echo "deb [signed-by=/etc/apt/keyrings/linbit.gpg] http://ppa.launchpad.net/linbit/linbit-drbd9-stack/ubuntu resolute main" > $@

sysroot/etc/apt/sources.list.d/openshot.list:
	$(MAKE) sysroot-mount-vfs
	$(MKDIR_P) --mode=700 sysroot/root/.gnupg
	$(CHROOT) gpgconf --launch gpg-agent
	$(CHROOT) gpg --no-default-keyring --keyring /tmp/keybox.gpg --keyserver keyserver.ubuntu.com \
	  --recv-keys B118F18F5C2AA998436F1603DC9E326890348C30
	$(CHROOT) gpg --no-default-keyring --keyring /tmp/keybox.gpg --export > sysroot/etc/apt/keyrings/openshot.gpg
	$(CHROOT) gpgconf --kill all
	rm sysroot/tmp/keybox.gpg
	echo "deb [signed-by=/etc/apt/keyrings/openshot.gpg] http://ppa.launchpad.net/openshot.developers/ppa/ubuntu resolute main" > $@

# FIXME: do i really need this one?  way back we got mbsystem from
#        here, along with other things like gmt, i thought...  but
#        after bionic (i think), mbsystem got dropped from Ubuntu-GIS
#        and I started building it myself with jammy... but by that
#        time i think all it's deps were in regular ubuntu...?
#
#        i think it still provides *newer* versions of packages,
#        including qgis
#
sysroot/etc/apt/sources.list.d/ubuntu-gis.list:
	$(MAKE) sysroot-mount-vfs
	$(MKDIR_P) --mode=700 sysroot/root/.gnupg
	$(CHROOT) gpgconf --launch gpg-agent
	$(CHROOT) gpg --no-default-keyring --keyring /tmp/keybox.gpg --keyserver keyserver.ubuntu.com \
	    --recv 6B827C12C2D425E227EDCA75089EBE08314DF160
	$(CHROOT) gpg --no-default-keyring --keyring /tmp/keybox.gpg --export > /sysroot/etc/apt/keyrings/ubuntugis.gpg
	$(CHROOT) gpgconf --kill all
	rm sysroot/tmp/keybox.gpg
	echo "deb [signed-by=/etc/apt/keyrings/ubuntugis.gpg] https://ppa.launchpadcontent.net/ubuntugis/ppa/ubuntu resolute main " > $@


################################################################################
# package groups
#
# Convenient groupings so we don't have to copy/paste when adding
# things to multiple layers
#
PKGS_FILESHARING =
PKGS_FILESHARING += nfs-kernel-server
PKGS_FILESHARING += samba samba-dsdb-modules samba-vfs-modules
PKGS_FILESHARING += wsdd
PKGS_FILESHARING += clamav

PKGS_ZFS =
PKGS_ZFS += zfs # our package, from sourceruckus ppa
PKGS_ZFS += zfs-auto-snapshot # ubuntu ppa, but now using our zfs package
# deps for sanoid, so we can use a development snapshot if desired
PKGS_ZFS += pv lzop mbuffer libcapture-tiny-perl libconfig-inifiles-perl
PKGS_ZFS += sanoid # ubuntu ppa, but now using our zfs package

# adds about 7G, but is pretty critical for building things ;-)
PKGS_COMPILATION =
PKGS_COMPILATION += build-essential manpages-dev
PKGS_COMPILATION += autoconf automake autopoint m4 libtool gettext pkg-config bison flex
PKGS_COMPILATION += cmake meson ccache bin86 gdb gperf help2man markdown nasm texinfo xsltproc
PKGS_COMPILATION += yajl-tools
PKGS_COMPILATION += dwarves # needed by kernel CONFIG_DEBUG_INFO_BTF
PKGS_COMPILATION += rdfind # needed by linux-firmware makefile
# add these in for ruckusrd subproject deps
PKGS_COMPILATION += libkmod-dev libblkid-dev # for eudev
PKGS_COMPILATION += libaio-dev # for lvm2
PKGS_COMPILATION += liblz4-dev liblzma-dev libzstd-dev zlib1g-dev liblzo2-dev # for squashfs-tools
PKGS_COMPILATION += libssl-dev libtirpc-dev libunwind-dev # for zfs
PKGS_COMPILATION += libacl1-dev libattr1-dev libext2fs-dev libpopt-dev libltdl-dev xxhash libxxhash-dev # for rsync
PKGS_COMPILATION += libefivar1t64 libefivar-dev libefiboot1t64 libefiboot-dev # for efibootmgr
PKGS_COMPILATION += netpbm # for groff
PKGS_COMPILATION += libbz2-dev libgcrypt20-dev # for fsarchiver
PKGS_COMPILATION += libslang2-dev # for newt
# add these for kernel compilation
PKGS_COMPILATION += libncurses-dev libelf-dev
# to build zfs deb packages
#
# NOTE: These things are required by the deb-utils target, which builds a bunch
#       of RPMs and then converts them into DEBs.  The resulting packages are
#       really only appropriate for a local apt install *.deb (because they
#       don't do ANY dependency tracking).  That's why I maintain special
#       Makefile rules (based on ruckusrd's 'dpkg' target) to build a single
#       zfs package with full dependency tracking.
#
#       In other words, we don't really need alien, and we probably don't
#       really need the two perl libs it required.  We probably don't need
#       fakeroot, either.  But we DO need the rest of them (in order to compile
#       OpenZFS later on), I haven't tested at all w/out those packages, we
#       might eventually want them, and it's not really that much more to add.
#
PKGS_COMPILATION += alien libarchive-cpio-perl libmail-sendmail-perl
PKGS_COMPILATION += libpython3-dev
#PKGS_COMPILATION += python3-distutils # no longer exists?
PKGS_COMPILATION += python3-setuptools
PKGS_COMPILATION += python3-cffi
PKGS_COMPILATION += python3-distlib python3-packaging
PKGS_COMPILATION += fakeroot
PKGS_COMPILATION += libudev-dev
# recommended at this point
PKGS_COMPILATION += libalgorithm-merge-perl libfl-dev libc6-dbg gdbserver
PKGS_COMPILATION += libfile-fcntllock-perl libwww-perl libxml-sax-expat-perl
PKGS_COMPILATION += ghostscript
PKGS_COMPILATION += libalgorithm-diff-xs-perl fonts-droid-fallback
PKGS_COMPILATION += libhtml-format-perl libpaper-utils libdata-dump-perl
PKGS_COMPILATION += libhtml-form-perl libhttp-daemon-perl libmailtools-perl
PKGS_COMPILATION += fonts-noto-mono libauthen-sasl-perl
# suggested at this point (not all of them, just a select few)
PKGS_COMPILATION += fonts-noto fonts-freefont-otf fonts-freefont-ttf
PKGS_COMPILATION += fonts-texgyre libdigest-hmac-perl
PKGS_COMPILATION += libgssapi-perl libcrypt-ssleay-perl libauthen-ntlm-perl
PKGS_COMPILATION += libxml-sax-expatxs-perl poppler-utils fonts-arphic-ukai
PKGS_COMPILATION += fonts-arphic-uming fonts-nanum fonts-noto-cjk
PKGS_COMPILATION += fonts-noto-cjk-extra fonts-noto-color-emoji fonts-noto-extra
PKGS_COMPILATION += fonts-noto-ui-core fonts-noto-ui-extra fonts-noto-unhinted
PKGS_COMPILATION += libmath-random-isaac-xs-perl
PKGS_COMPILATION += libasprintf-dev libgettextpo-dev elfutils
# to build ltfs
PKGS_COMPILATION += icu-devtools
PKGS_COMPILATION += libfuse3-dev libfuse-dev libnet-snmp-perl libcrypt-des-perl libsnmp-dev libsnmp-perl
PKGS_COMPILATION += libxml2-dev libxml2-utils libicu-dev
# to build xen
#
# NOTE: Some of these may not technically be needed for building xen,
#       but they were in my notes as being installed the last time I
#       did this.
#
PKGS_COMPILATION += python3-dev
PKGS_COMPILATION += bcc libcurl4-openssl-dev
PKGS_COMPILATION += fig2dev tgif
PKGS_COMPILATION += texlive-latex-base texlive-latex-recommended
PKGS_COMPILATION += texlive-fonts-extra texlive-fonts-recommended mercurial
PKGS_COMPILATION += python3-twisted
PKGS_COMPILATION += libvncserver-dev libsdl1.2-dev libsdl2-dev libjpeg-dev
PKGS_COMPILATION += libglib2.0-dev
PKGS_COMPILATION += libnl-3-dev libnl-cli-3-dev libnl-genl-3-dev libnl-route-3-dev
PKGS_COMPILATION += libnl-idiag-3-dev libnl-xfrm-3-dev
PKGS_COMPILATION += acpica-tools ocaml ocaml-findlib
PKGS_COMPILATION += ocamlbuild libx11-dev libyajl-dev
PKGS_COMPILATION += libpixman-1-dev pandoc figlet
PKGS_COMPILATION += libc6-dev-i386
PKGS_COMPILATION += lzma
PKGS_COMPILATION += libsystemd-dev
PKGS_COMPILATION += python3-pip python3-venv
PKGS_COMPILATION += gcc-13 # had to downgrade gcc to compile xen... :-/
# recommends from the xen packages
PKGS_COMPILATION += gcc-multilib libdecor-0-plugin-1-gtk #default-libdecor-0-plugin-1
#PKGS_COMPILATION += libgl1-amber-dri # FIXME: resolute really didn't like this one...
PKGS_COMPILATION += ocaml-man libtasn1-doc mesa-vulkan-drivers
PKGS_COMPILATION += libfindlib-ocaml-dev ledit python3-click lmodern dvisvgm
PKGS_COMPILATION += cm-super fonts-adf-accanthis fonts-adf-berenis fonts-adf-gillius
PKGS_COMPILATION += fonts-adf-universalis fonts-cabin fonts-cantarell fonts-clear-sans
PKGS_COMPILATION += fonts-comfortaa fonts-comic-neue fonts-croscore fonts-crosextra-caladea
PKGS_COMPILATION += fonts-crosextra-carlito fonts-dejavu-core fonts-dejavu-extra
PKGS_COMPILATION += fonts-ebgaramond-extra fonts-font-awesome fonts-gfs-artemisia
PKGS_COMPILATION += fonts-gfs-complutum fonts-gfs-didot fonts-gfs-neohellenic fonts-gfs-olga
PKGS_COMPILATION += fonts-gfs-solomos fonts-go fonts-inter fonts-lato fonts-linuxlibertine
PKGS_COMPILATION += fonts-lobstertwo fonts-oflb-asana-math fonts-open-sans fonts-paratype
PKGS_COMPILATION += fonts-roboto-slab fonts-roboto-unhinted fonts-sil-andika fonts-sil-charis
PKGS_COMPILATION += fonts-sil-gentium fonts-sil-gentium-basic fonts-sil-gentiumplus
PKGS_COMPILATION += fonts-sil-gentiumplus-compact fonts-stix texlive-fonts-extra-links
PKGS_COMPILATION += texlive-latex-extra tex-gyre tipa xfonts-75dpi libfile-mimeinfo-perl
PKGS_COMPILATION += libnet-dbus-perl libx11-protocol-perl x11-utils x11-xserver-utils
# to build mbsystem (generic compilation toolchain stuff)
#
# NOTE: Some of these may not technically be needed for building mbsystem,
#       but they were in my notes as being installed the last time I
#       did this.
#
PKGS_COMPILATION += autoconf-archive diffstat gfortran
PKGS_COMPILATION += gv libdevel-globaldestruction-perl
PKGS_COMPILATION += libfile-remove-perl libgfortran5 libhash-ordered-perl
PKGS_COMPILATION += libmail-box-perl libmail-message-perl libmail-transport-perl
PKGS_COMPILATION += libmime-types-perl libobject-realize-later-perl
PKGS_COMPILATION += libsub-exporter-progressive-perl libuser-identity-perl xaw3dg
PKGS_COMPILATION += libdraco-dev
# to build mbsystem (xorg stuff)
#
# NOTE: Some of these may not technically be needed for building mbsystem,
#       but they were in my notes as being installed the last time I
#       did this.
#
PKGS_COMPILATION += xorg-dev mesa-common-dev libmotif-common libmotif-dev
PKGS_COMPILATION += libsdl-image1.2-dev libsdl-mixer1.2-dev libsdl-ttf2.0-dev libsdl2-gfx-dev
PKGS_COMPILATION += libsdl2-image-dev libsdl2-mixer-dev libsdl2-ttf-dev
PKGS_COMPILATION += libgirepository1.0-dev
PKGS_COMPILATION += python3-pygments libpipewire-0.3-common
# to build mbsystem (gis stuff, some from ubuntu-gis ppa)
#
# FIXME: we need to actually prep the ubuntu-gis ppa before this will
#        work...
#
# FIXME: that's what my notes said, but it doesn't seem like any of
#        these are coming from the Ubuntu-GIS PPA...
#
PKGS_COMPILATION += gmt geotiff-bin gdal-bin libgeotiff-dev
PKGS_COMPILATION += odbc-postgresql tdsodbc ogdi-bin
PKGS_COMPILATION += gmt-gshhg libgmt-dev libgdal-plugin-grass libgnutls28-dev
PKGS_COMPILATION += libidn-dev libkrb5-dev libldap2-dev librtmp-dev libssh2-1-dev
PKGS_COMPILATION += netcdf-bin libfftw3-bin libfftw3-dev # libfftw3-3 # not in noble?
PKGS_COMPILATION += libfftw3-long3 libfftw3-quad3 libfftw3-double3 libfftw3-mpi-dev
PKGS_COMPILATION += libfftw3-mpi3 libfftw3-single3 libproj-dev proj-bin
PKGS_COMPILATION += libhdf4-dev hdf4-tools
PKGS_COMPILATION += grass-dev grass-gui e00compr avce00 gnuplot gpsbabel
PKGS_COMPILATION += gpstrans python3-rpy2 python3-termcolor grass krb5-k5tls
PKGS_COMPILATION += libmpfr-dev gnutls-bin libhwloc-contrib-plugins idn
PKGS_COMPILATION += setserial libitpp-dev elpa-ess libparallel-forkmanager-perl gmt-dcw
PKGS_COMPILATION += subversion python3-matplotlib libnamespace-clean-perl qttranslations5-l10n
PKGS_COMPILATION += libpackage-stash-xs-perl qt5-gtk-platformtheme libwacom-bin python3-bs4
PKGS_COMPILATION += python3-html5lib python3-chardet python3-olefile r-recommended
PKGS_COMPILATION += r-base-dev
# more packages needed to build mbsystem as of 5.8.3beta16.  well,
# probably needed prior to that version, but that's the one I was
# trying to build when I realized
#
PKGS_COMPILATION += libnetcdf-dev libgdal-dev
PKGS_COMPILATION += xfonts-100dpi libglu1-mesa-dev
PKGS_COMPILATION += libopencv-dev
# recommended packages from that last group
#
#PKGS_COMPILATION += libexif-doc
PKGS_COMPILATION += libgphoto2-l10n gstreamer1.0-plugins-base opencv-data
PKGS_COMPILATION += libopencv-java libraw1394-tools
#PKGS_COMPILATION += va-driver-all
#PKGS_COMPILATION += vdpau-driver-all


################################################################################
# ES (base)
#
# This is going to be used for most of our domUs.  Should include what's needed
# for srv-router, srv-net, srv-auth.
#

# stage1 - 220M
#
.build/RUCKUS-DEBOOTSTRAP:
	$(MAKE) sysroot-clean
	$(MKDIR_P) $(dir $@)
	DEBOOTSTRAP_DIR=$(DEBOOTSTRAP_DIR) $(DEBOOTSTRAP) --arch=amd64 resolute sysroot http://archive.ubuntu.com/ubuntu/
	touch $@

# after updates, sysroot is ~293M
.build/RUCKUS-ES-APT-PREP: .build/RUCKUS-DEBOOTSTRAP
	$(MAKE) sysroot-mount-vfs
	echo "deb http://archive.ubuntu.com/ubuntu/ resolute main restricted universe multiverse" > sysroot/etc/apt/sources.list
	echo "deb http://archive.ubuntu.com/ubuntu/ resolute-updates main restricted universe multiverse" >> sysroot/etc/apt/sources.list
	echo "deb http://archive.ubuntu.com/ubuntu/ resolute-backports main restricted universe multiverse" >> sysroot/etc/apt/sources.list
	echo "deb http://security.ubuntu.com/ubuntu/ resolute-security main restricted universe multiverse" >> sysroot/etc/apt/sources.list
	$(CHROOT) apt update
	$(CHROOT) apt upgrade -y --autoremove
	touch $@

# base list (pared down from ubuntu-server)
#
# This base package list is generated manually by reviewing the
# packages that would be installed by ubuntu-server, removing some
# stuff we don't want, and adding some stuff we do.  The resulting
# image should be tiny and awesome.
#
# 1. apt-get install ubuntu-server [n]
# 2. Review apps (skip over libs and python mods, remove initramfs
#    stuff, remove vm/cloud stuff).
# 3. apt install --no-install-recommends $PKGS_ES [n]
# 4. Compare and contrast, try again until you like the list.
#
# W/out removing things, ubuntu-server adds ~653M.  All our pruning
# reduces that to just an additional ~302M, then we add in things we
# actually want.
#
# Removed
# - apparmor
# - apport apport-*
# - appstream
# - bcache-tools
# - bolt (installed in es-server)
# - busybox-initramfs
# - byobu
# - cloud-*
# - cryptsetup-initramfs
# - dmidecode (installed in es-server)
# - dracut-*
# - fonts-ubuntu-console
# - fwupd fwupd-signed
# - initramfs-tools initramfs-tools-*
# - klibc-utils
# - landscape-common
# - lxd-*
# - multipath-tools (pulled in initramfs stuff)
# - open-iscsi (installed in es-server)
# - open-vm-tools
# - overlayroot
# - pastebinit
# - plymouth plymouth-*
# - sbsigntool secureboot-db
# - sg3-utils sg3-utils-udev (pulled in initramfs stuff)
# - snapd
# - sos
# - sosreport
# - tmux
# - tpm-udev
# - udisks2 (installed in es-server)
# - ubuntu-server
# - update-manager-core update-notifier-common
#
# NOTE: noble utils to play with: htop zerofree screen
#
# NOTE: resolute utils to play with: 3cpio
#
PKGS_ES =
PKGS_ES += 3cpio # new in resolute, replaces cpio (but not drop-in replacement)
PKGS_ES += bash-completion
PKGS_ES += bc
PKGS_ES += binutils
PKGS_ES += binutils-common
PKGS_ES += binutils-x86-64-linux-gnu
PKGS_ES += btrfs-progs
PKGS_ES += cpio
PKGS_ES += cryptsetup
PKGS_ES += cryptsetup-bin
PKGS_ES += curl
PKGS_ES += dirmngr
PKGS_ES += dmeventd
PKGS_ES += dosfstools
PKGS_ES += exfatprogs
PKGS_ES += finalrd
PKGS_ES += fuse3
PKGS_ES += gawk
PKGS_ES += gettext-base
PKGS_ES += gir1.2-packagekitglib-1.0
PKGS_ES += git
PKGS_ES += git-man
PKGS_ES += gnupg
PKGS_ES += gnupg-l10n
PKGS_ES += gnupg-utils
PKGS_ES += gpg
PKGS_ES += gpg-agent
PKGS_ES += gpg-wks-client
PKGS_ES += gpgconf
PKGS_ES += gpgsm
PKGS_ES += htop
PKGS_ES += jq
PKGS_ES += kpartx
PKGS_ES += linux-base
PKGS_ES += lshw
PKGS_ES += lsof
PKGS_ES += lvm2
PKGS_ES += mdadm
PKGS_ES += modemmanager
PKGS_ES += motd-news-config
PKGS_ES += needrestart
PKGS_ES += ntfs-3g
PKGS_ES += openssh-client
PKGS_ES += packagekit
PKGS_ES += parted
PKGS_ES += patch
PKGS_ES += pci.ids
PKGS_ES += pciutils
PKGS_ES += perl
PKGS_ES += perl-modules-5.40
PKGS_ES += pinentry-curses
PKGS_ES += polkitd
PKGS_ES += publicsuffix
PKGS_ES += screen # previously removed
PKGS_ES += sgml-base
PKGS_ES += software-properties-common
PKGS_ES += squashfs-tools
PKGS_ES += tcl
PKGS_ES += tcl8.6
PKGS_ES += thin-provisioning-tools
PKGS_ES += ubuntu-release-upgrader-core
PKGS_ES += unattended-upgrades
PKGS_ES += usb-modeswitch
PKGS_ES += usb-modeswitch-data
PKGS_ES += usb.ids
PKGS_ES += util-linux-extra # new in resolute
PKGS_ES += vim
PKGS_ES += vim-runtime
PKGS_ES += xauth
PKGS_ES += xfsprogs
PKGS_ES += xml-core
PKGS_ES += xz-utils
PKGS_ES += zerofree
PKGS_ES += zstd
# and that's all we want from ubuntu-server
#
# things that used to be in ubuntu-server that we still want (additional ~900K)
PKGS_ES += ethtool
PKGS_ES += fdisk
#PKGS_ES += gdisk # maybe?
#PKGS_ES += groff-base # maybe?
PKGS_ES += pollinate
PKGS_ES += run-one
#
# additional things we want (~742M last time I checked)
PKGS_ES += lz4
PKGS_ES += iptables
PKGS_ES += network-manager
PKGS_ES += iptables-persistent #(yes for saving rules, dpkg-reconfigure later?)
PKGS_ES += tshark #(yes for non-root capture, dpkg-reconfigure wireshark-common later?)
PKGS_ES += openssh-server
#PKGS_ES += ntp sntp # replaced in resolute by ntpsec and ntpsec-ntpdig
PKGS_ES += ntpsec ntpsec-ntpdig
PKGS_ES += ncurses-term
PKGS_ES += dnsmasq
PKGS_ES += cifs-utils smbclient
PKGS_ES += keyutils winbind libnss-winbind libpam-winbind
PKGS_ES += acl
PKGS_ES += attr
PKGS_ES += bind9-host bind9-dnsutils net-tools
PKGS_ES += vlan bridge-utils ifenslave
PKGS_ES += postfix #(no config, use dpkg-reconfigre to tweak later)
PKGS_ES += libsasl2-modules
PKGS_ES += mailutils
PKGS_ES += zsh gkrellmd
PKGS_ES += git-email
PKGS_ES += emacs-nox emacs-common-non-dfsg
PKGS_ES += aspell aspell-en
PKGS_ES += autofs nfs-common
PKGS_ES += ftp ncftp
# FIXME: is this wireguard comment still relevant?
PKGS_ES += wireguard-tools #(--no-install-recommends to avoid grub junk - or apt-mark hold wireguard-dkms wireguard-modules)
PKGS_ES += logwatch libdate-manip-perl
PKGS_ES += rsync
PKGS_ES += info
PKGS_ES += traceroute
PKGS_ES += tree
PKGS_ES += zip unzip unrar
PKGS_ES += xattr
# FIXME: krb5 didn't ask during initial resolute install?  had to dpkg-reconfigure -plow krb5-config
PKGS_ES += krb5-user #(specified default realm (SOURCERUCKUS.ORG), server (krb5), and pw server (krb5pw), use dpkg-reconfigure later?)
PKGS_ES += libpam-krb5
PKGS_ES += mosh
PKGS_ES += wget
PKGS_ES += man-db wamerican whois
PKGS_ES += psmisc
PKGS_ES += fio
PKGS_ES += iperf
PKGS_ES += iperf3 #(no for run as daemon automatically, dpkg-reconfigure later? no was the default)
PKGS_ES += iotop
PKGS_ES += sysstat
PKGS_ES += nmap
PKGS_ES += apt-file
PKGS_ES += fsarchiver
PKGS_ES += bzip2
PKGS_ES += manpages
#
# FIXME: would add nftables so I can play with it, but afraid it will
#        mess with iptables
#
# PKGS_ES += nftables iptables-nftables-compat
#


# compilation toolchain and additional development tools and libraries (~7G)
#
# NOTE: We normally leave all these development packages until VMM or
#       Workstation.  You only need them at this point if this is the first
#       ever bootstrap build of a new release and we need to actually compile
#       the Source Ruckus PPA packages.  If that's the case, add
#       PKGS_COMPILATION to PKGS_ES for a test build and do your thing.
#
if ES_DEVEL
PKGS_ES += $(PKGS_COMPILATION)
endif

.build/RUCKUS-ES-APT: .build/RUCKUS-ES-APT-PREP
	$(MAKE) sysroot-mount-vfs
	cp debconf-config.dat sysroot/var/cache/debconf/config.dat
	cat debconf-config.dat | $(CHROOT) $(APT_INSTALL) $(PKGS_ES)
	touch $@

# NOTE: Definition of "3rd Party" in here is "upstream != Ubuntu".  That means
#       our own Source Ruckus PPA is considered "3rd Party".
#
# NOTE: Regarding the chicken-and-the-egg of the Source Ruckus PPA while
#       preping a distro-upgrade...  The easiest thing to do is manually take a
#       detour and build required packages by hand instead of somehow detecting
#       that need and automagically doing it via the build system.  I'm gonna
#       have to re-investigate required packages and stuff by hand, there's no
#       point in the added complexity required to "automagically" build things
#       instead of installing.  So, after a 'make .build/RUCKUS-ES-3RDPARTY' (which
#       should fail because missing packages), manually chroot into the
#       resulting system, build packages and upload them to our PPA.
#
# NOTE: We prep the PPAs for the ES-file and ES-web additions now, so that we
#       can easily apt install things for testing and corner-cases
#       (e.g. combined file/web server).
#
# NOTE: If --enable-devel, we're building es-devel.sqsh, leave PKGS_ES_3RDPARTY
#       empty (because they'll probably/definately fail)
#
PKGS_ES_3RDPARTY =
if !ES_DEVEL
PKGS_ES_3RDPARTY += telegraf #(135M! didn't realize it was that large)
PKGS_ES_3RDPARTY += pingenator
PKGS_ES_3RDPARTY += doit
PKGS_ES_3RDPARTY += extractenator
# technically only the zfs package is 3rd-party, the rest is pure ubuntu
PKGS_ES_3RDPARTY += $(PKGS_ZFS)
endif

# NOTE: For --enable-devel, we need to empty this list out as well.  We need to
#       do these by hand during the bootstrap process and update the rules as
#       we go.  The Source Ruckus PPA could just be left out, but in all
#       likelyhood at least one more of these will be invalid on our first
#       pass.
#
# NOTE: This is defined as an additional rule (instead of just adding more
#       depends to RUCKUS-ES-3RDPARTY) to silence some automake warnings.
#
.PHONY: PPAS_ES
if !ES_DEVEL
PPAS_ES: sysroot/etc/apt/sources.list.d/sourceruckus.list
PPAS_ES: sysroot/etc/apt/sources.list.d/influxdb.list # for telegraf
endif

.build/RUCKUS-ES-3RDPARTY: .build/RUCKUS-ES-APT
	$(MAKE) sysroot-mount-vfs
	if [ -n "$(PKGS_ES_3RDPARTY)" ]; then \
	  $(MAKE) PPAS_ES; \
	  $(CHROOT) apt update; \
	  cat debconf-config.dat | $(CHROOT) $(APT_INSTALL) $(PKGS_ES_3RDPARTY); \
	fi
	touch $@

# the finalization step - set default boot target, disable some services, tweak
# some files, etc.
#
# NOTE: We clone awesome-powers and prep it's submodules in here, so that it's
#       available if we're installing w/out an internet connection later on.
#
# NOTE: We do NOT create local users here.  That's handled by awesome-powers
#       during installation/groom.
#
# NOTE: We create globally-managed-devices.conf because all interfaces end up
#       "unmanaged" in NetworkManager w/out it.  (This was definately needed
#       for focal, but I forgot to check if it's still needed with
#       jammy... doesn't hurt anything though)
#
# NOTE: We disable and mask the RPC security service for NFS here, because upon
#       joining an AD domain, even if you don't have nfs-kernel-server
#       installed, this service autostarts and fails.
#
# NOTE: We increase the minimum user id for PAM's krb5 module here to
#       better differentiate between local and domain users.  This
#       way, the krb5 lookups don't happen for users defined locally
#       (i.e., local users OVERRIDE domain ones) and more importantly,
#       if the krb server is unreachable we won't induce an
#       agonizingly long timeout when trying to login to a local
#       account (usally to fix the krb5 problem).
#
#       We used to do this via /etc/login.defs to avoid apt upgrade
#       "modified files" questions, but that file isn't consistently
#       used anymore (e.g., accountservice ignores it).
#
# FIXME: should really move awesome-powers to /opt
#
.build/RUCKUS-ES-FINAL: .build/RUCKUS-ES-3RDPARTY
	$(MAKE) sysroot-mount-vfs
	echo "rootfs / auto defaults 0 0" > sysroot/etc/fstab
	mkdir -p sysroot/usr/lib/modules
	mkdir -p sysroot/usr/lib/firmware
	mkdir -p sysroot/usr/src
	touch sysroot/etc/NetworkManager/conf.d/10-globally-managed-devices.conf
	$(MKDIR_P) sysroot/scrap && cd sysroot/scrap && \
	    git clone https://github.com/sourceruckus/awesome-powers.git && \
	    cd awesome-powers && ./autogen.sh
	$(CHROOT) apt clean
	$(CHROOT) systemctl set-default multi-user.target
	$(CHROOT) systemctl disable dnsmasq
	$(CHROOT) systemctl disable rpc-svcgssd.service
	$(CHROOT) systemctl mask rpc-svcgssd.service
	$(CHROOT) [ -e /sbin/zfs ] && \
	  systemctl enable zfs-import-cache.service zfs-import.target zfs-mount.service \
	  zfs-share.service zfs-volume-wait.service zfs-volumes.target zfs-zed.service \
	  zfs.target
	rm -rfv sysroot/tmp/*
	grep minimum_uid sysroot/etc/pam.d/*
	sed -i 's|pam_krb5.so minimum_uid=1000|pam_krb5.so minimum_uid=1500|' sysroot/etc/pam.d/*
	grep minimum_uid sysroot/etc/pam.d/*

	touch $@


# NOTE: If --enable-devel, just swap out the name of this rule.
#
if ES_DEVEL
es-devel.sqsh: .build/RUCKUS-ES-FINAL
else
es.sqsh: .build/RUCKUS-ES-FINAL
endif
	$(MAKE) sysroot-umount-vfs
	mksquashfs sysroot $@ $(SQSH_COMP) -noappend -wildcards


################################################################################
# ES-file
#
# Based on ES, adds file-server specific packages.
#
PKGS_ESFILE =
PKGS_ESFILE += $(PKGS_FILESHARING)
PKGS_ESFILE += minidlna libaacs0 libgdk-pixbuf2.0-bin pocketsphinx-en-us librsvg2-common libbdplus0
# FIXME: at this rate, i'm probably never going to play with
#        orthanc... so commenting it out
#
# FIXME: is the orthanc stuff going to need apache2?
#PKGS_ESFILE += orthanc
#PKGS_ESFILE += orthanc-dicomweb
#PKGS_ESFILE += orthanc-doc javascript-common
##PKGS_ESFILE += orthanc-imagej mesa-vulkan-drivers libatk-wrapper-java-jni fonts-dejavu-extra
##PKGS_ESFILE += orthanc-mysql
##PKGS_ESFILE += orthanc-postgresql
#PKGS_ESFILE += orthanc-webviewer
#PKGS_ESFILE += orthanc-wsi libopenslide0


# prepare an overlay mount on sysroot
.build/RUCKUS-ESFILE-OVL:
	$(MAKE) sysroot-clean
	$(MAKE) sqsh_layer-es upper_layer
	$(MKDIR_P) sysroot upper_layer/{upper,work}
	mount -t overlay -o lowerdir=sqsh_layer-es,upperdir=upper_layer/upper,workdir=upper_layer/work none sysroot
	touch $@

# apt install
.build/RUCKUS-ESFILE-APT: .build/RUCKUS-ESFILE-OVL
	$(MAKE) sysroot-mount-vfs
	cat debconf-config.dat | $(CHROOT) $(APT_INSTALL) $(PKGS_ESFILE)
	touch $@

# service enable/dislabe, tweaks, etc
#
# FIXME: audit services
#
.build/RUCKUS-ESFILE-FINAL: .build/RUCKUS-ESFILE-APT
	$(MAKE) sysroot-mount-vfs
	rm -rfv sysroot/tmp/*
	touch $@

# mksquashfs
es-file.sqsh: .build/RUCKUS-ESFILE-FINAL
	$(MAKE) sysroot-umount-vfs
	mksquashfs upper_layer/upper $@ $(SQSH_COMP) -noappend -wildcards


################################################################################
# ES-auth
#
# Based on ES, not ES-file, and adds auth-server stuff.
#
# NOTE: Currently just samba, for samba-active-directory, but we could
#       branch out and try other products.
#
PKGS_ESAUTH =
PKGS_ESAUTH += samba
PKGS_ESAUTH += python3-idna samba-dsdb-modules samba-vfs-modules samba-ad-provision

.build/RUCKUS-ESAUTH-OVL:
	$(MAKE) sysroot-clean
	$(MAKE) sqsh_layer-es upper_layer
	$(MKDIR_P) sysroot upper_layer/{upper,work}
	mount -t overlay -o lowerdir=sqsh_layer-es,upperdir=upper_layer/upper,workdir=upper_layer/work none sysroot
	touch $@

# apt install
.build/RUCKUS-ESAUTH-APT: .build/RUCKUS-ESAUTH-OVL
	$(MAKE) sysroot-mount-vfs
	cat debconf-config.dat | $(CHROOT) $(APT_INSTALL) $(PKGS_ESAUTH)
	touch $@

# service enable/disable, tweaks, etc
#
# FIXME: audit services
#
.build/RUCKUS-ESAUTH-FINAL: .build/RUCKUS-ESAUTH-APT
	$(MAKE) sysroot-mount-vfs
	rm -rfv sysroot/tmp/*
	touch $@

es-auth.sqsh: .build/RUCKUS-ESAUTH-FINAL
	$(MAKE) sysroot-umount-vfs
	mksquashfs upper_layer/upper $@ $(SQSH_COMP) -noappend -wildcards


################################################################################
# ES-web
#
# Based on ES, not ES-file, and adds web-server stuff.
#
# FIXME: double check influxdb version... at some point i ended up
#        with the wrong version (ubuntu, not upstream) installed in
#        here.  might have been a fluke, as I'd be walking through the
#        rules randomly by hand testing things out
#
.PHONY: PPAS_ESWEB
PPAS_ESWEB: sysroot/etc/apt/sources.list.d/grafana.list
PPAS_ESWEB: sysroot/etc/apt/sources.list.d/mongodb.list
PPAS_ESWEB: sysroot/etc/apt/sources.list.d/ubiquiti.list

PKGS_ESWEB =
PKGS_ESWEB += apache2 ssl-cert
PKGS_ESWEB += javascript-common
PKGS_ESWEB += influxdb #  upstream ppa added in ES
PKGS_ESWEB += grafana # upstream ppa
### NOT starting on installation, please execute the following statements to configure grafana to start automatically using systemd
# sudo /bin/systemctl daemon-reload
# sudo /bin/systemctl enable grafana-server
### You can start grafana-server by executing
# sudo /bin/systemctl start grafana-server
#
# FIXME: wtf... the 26.04 unifi ppa is empty?  commenting this out for now.
#
#PKGS_ESWEB += unifi # upstream ppa, also pulls in old version of mongodb

.build/RUCKUS-ESWEB-OVL:
	$(MAKE) sysroot-clean
	$(MAKE) sqsh_layer-es upper_layer
	$(MKDIR_P) sysroot upper_layer/{upper,work}
	mount -t overlay -o lowerdir=sqsh_layer-es,upperdir=upper_layer/upper,workdir=upper_layer/work none sysroot
	touch $@

.build/RUCKUS-ESWEB-PPAS: .build/RUCKUS-ESWEB-OVL
	$(MAKE) sysroot-mount-vfs
	$(MAKE) PPAS_ESWEB
	$(CHROOT) apt update
	touch $@

# apt install
.build/RUCKUS-ESWEB-APT: .build/RUCKUS-ESWEB-PPAS
	$(MAKE) sysroot-mount-vfs
	cat debconf-config.dat | $(CHROOT) $(APT_INSTALL) $(PKGS_ESWEB)
	touch $@

# service enable/disable, tweaks, etc
#
# FIXME: audit services
#
.build/RUCKUS-ESWEB-FINAL: .build/RUCKUS-ESWEB-APT
	$(MAKE) sysroot-mount-vfs
	rm -rfv sysroot/tmp/*
	touch $@

es-web.sqsh: .build/RUCKUS-ESWEB-FINAL
	$(MAKE) sysroot-umount-vfs
	mksquashfs upper_layer/upper $@ $(SQSH_COMP) -noappend -wildcards


################################################################################
# Server
#
# Based on ES, not ES-file, ES-web, or ES-auth, this is the first
# layer that is intended for use on physical hardware (i.e., not a
# virtual machine), so it's where we start to add in bootloader and
# more hardware-centric things.
#
.PHONY: PPAS_SERVER
PPAS_SERVER: sysroot/etc/apt/sources.list.d/linbit-drbd9.list

PKGS_SERVER =
PKGS_SERVER += drbd-utils # upstream ppa
PKGS_SERVER += $(PKGS_FILESHARING)
PKGS_SERVER += ruckusrd
PKGS_SERVER += firmwarenator
PKGS_SERVER += microcodenator
PKGS_SERVER += syslinux syslinux-common extlinux isolinux syslinux-efi syslinux-utils
PKGS_SERVER += efibootmgr
PKGS_SERVER += efitools efivar
PKGS_SERVER += secureboot-db
PKGS_SERVER += powermgmt-base
PKGS_SERVER += fwupd fwupd-signed
PKGS_SERVER += smartmontools
PKGS_SERVER += hdparm
PKGS_SERVER += sdparm
PKGS_SERVER += rfkill
PKGS_SERVER += bolt
PKGS_SERVER += dmidecode
PKGS_SERVER += open-iscsi
PKGS_SERVER += udisks2
PKGS_SERVER += lm-sensors fancontrol read-edid i2c-tools
PKGS_SERVER += ltfs

.build/RUCKUS-SERVER-OVL:
	$(MAKE) sysroot-clean
	$(MAKE) sqsh_layer-es upper_layer
	$(MKDIR_P) sysroot upper_layer/{upper,work}
	mount -t overlay -o lowerdir=sqsh_layer-es,upperdir=upper_layer/upper,workdir=upper_layer/work none sysroot
	touch $@

.build/RUCKUS-SERVER-PPAS: .build/RUCKUS-SERVER-OVL
	$(MAKE) sysroot-mount-vfs
	$(MAKE) PPAS_SERVER
	$(CHROOT) apt update
	touch $@

# apt install
.build/RUCKUS-SERVER-APT: .build/RUCKUS-SERVER-PPAS
	$(MAKE) sysroot-mount-vfs
	cat debconf-config.dat | $(CHROOT) $(APT_INSTALL) $(PKGS_SERVER)
	touch $@

# service enable/dislabe, tweaks, etc
#
# FIXME: audit services
#
.build/RUCKUS-SERVER-FINAL: .build/RUCKUS-SERVER-APT
	$(MAKE) sysroot-mount-vfs
	rm -rfv sysroot/tmp/*
	touch $@

server.sqsh: .build/RUCKUS-SERVER-FINAL
	$(MAKE) sysroot-umount-vfs
	mksquashfs upper_layer/upper $@ $(SQSH_COMP) -noappend -wildcards


################################################################################
# VMM (Xen Dom0)
#
# Based on Server, adds things needed for our VMM setup.
#
# NOTE: Currently just xen, but we could add other products here
#       later.  Maybe XCP-ng, XAPI, or Xen Orchestra for making things
#       prettier?
#
PKGS_VMM =
PKGS_VMM += xen
PKGS_VMM += qemu-system-x86 grub-xen-host qemu-utils seabios ovmf
PKGS_VMM += default-libdecor-0-plugin-1
PKGS_VMM += librsvg2-common os-prober at-spi2-core libgdk-pixbuf2.0-bin libgtk-3-bin
PKGS_VMM += ipxe-qemu-256k-compat-efi-roms qemu-system-gui qemu-system-modules-spice
PKGS_VMM += qemu-system-modules-opengl qemu-block-extra cpu-checker
#PKGS_VMM += libgl1-amber-dri # resolute really didn't like this one

.build/RUCKUS-VMM-OVL:
	$(MAKE) sysroot-clean
	$(MAKE) sqsh_layer-es sqsh_layer-server upper_layer
	$(MKDIR_P) sysroot upper_layer/{upper,work}
	mount -t overlay -o lowerdir=sqsh_layer-server:sqsh_layer-es,upperdir=upper_layer/upper,workdir=upper_layer/work none sysroot
	touch $@

# apt install
.build/RUCKUS-VMM-APT: .build/RUCKUS-VMM-OVL
	$(MAKE) sysroot-mount-vfs
	cat debconf-config.dat | $(CHROOT) $(APT_INSTALL) $(PKGS_VMM)
	touch $@

# service enable/disable, tweaks, etc
#
# FIXME: audit services
#
.build/RUCKUS-VMM-FINAL: .build/RUCKUS-VMM-APT
	$(MAKE) sysroot-mount-vfs
	$(CHROOT) systemctl enable xen-init-dom0 xen-qemu-dom0-disk-backend xen-watchdog xenconsoled xenstored
	rm -rfv sysroot/tmp/*
	touch $@

vmm.sqsh: .build/RUCKUS-VMM-FINAL
	$(MAKE) sysroot-umount-vfs
	mksquashfs upper_layer/upper $@ $(SQSH_COMP) -noappend -wildcards


################################################################################
# Workstation
#
# Based off Server (which exludes ES-file and ES-web additions), adds everything
# else we want in an awesome workstation of doom (including the kitchen sink).
#
# NOTE: This image includes upstream Wine (including full 32bit library
#       support) so you can run Windows junk.
#
# FIXME:
# - via apt
#   - jackd?
#   - dvd player stuff (maybe?)
#   - special font packages (maybe?)
#   - libglvnd-dev (what did i need this for...?)
# - 3rd party stuff
#   - ardour (wget the download, although I needed to pay for this didn't i... should i just install the ubuntu debs?)
#   - vmware player?
#   - virtualbox?
#
.PHONY: PPAS_WORKSTATION
PPAS_WORKSTATION: sysroot/etc/apt/sources.list.d/wine.list
PPAS_WORKSTATION: sysroot/etc/apt/sources.list.d/qgis.list
PPAS_WORKSTATION: sysroot/etc/apt/sources.list.d/openshot.list

PKGS_WORKSTATION =
PKGS_WORKSTATION += ubuntu-desktop
# development stuff
PKGS_WORKSTATION += $(PKGS_COMPILATION)
PKGS_WORKSTATION += kernel-builder
PKGS_WORKSTATION += gcc-multilib g++-multilib gfortran-multilib
PKGS_WORKSTATION += php php-xml
PKGS_WORKSTATION += default-jdk
# GIS stuff
PKGS_WORKSTATION += gmt gmt-dcw gmt-gshhg proj-bin
PKGS_WORKSTATION += mbsystem
PKGS_WORKSTATION += python3-geopandas
# everything else we want
PKGS_WORKSTATION += gimp gimp-help-en gimp-data-extras gsfonts libbluray-bdj mesa-opencl-icd
PKGS_WORKSTATION += libvdpau-va-gl1 libvdpau1 vdpauinfo
PKGS_WORKSTATION += groff ghostscript imagemagick libpaper2 netpbm psutils
PKGS_WORKSTATION += ffmpeg
PKGS_WORKSTATION += gnuplot
PKGS_WORKSTATION += inkscape xfig dia
PKGS_WORKSTATION += audacity audacious
PKGS_WORKSTATION += opus-tools librsvg2-bin speex vorbis-tools sox
PKGS_WORKSTATION += va-driver-all vdpau-driver-all libexif-doc mesa-utils
PKGS_WORKSTATION += mesa-vulkan-drivers
PKGS_WORKSTATION += wireshark gkrellm gitk git-gui
PKGS_WORKSTATION += vlc
PKGS_WORKSTATION += gnome-tweaks rdesktop
PKGS_WORKSTATION += emacs-gtk emacs # replaces emacs-nox
PKGS_WORKSTATION += wine-devel # upstream ppa # FIXME: this sometimes has upgrade dependency problems with wine-devel-amd64 and wine-devel-i386
PKGS_WORKSTATION += qgis # upstream-ppa
PKGS_WORKSTATION += openshot-qt # libopenshot libopenshot-audio # upstream ppa

PKGS_WORKSTATION_REMOVE =
PKGS_WORKSTATION_REMOVE += update-manager update-notifier
PKGS_WORKSTATION_REMOVE += gnome-initial-setup

.build/RUCKUS-WORKSTATION-OVL:
	$(MAKE) sysroot-clean
	$(MAKE) sqsh_layer-es sqsh_layer-server sqsh_layer-vmm upper_layer
	$(MKDIR_P) sysroot upper_layer/{upper,work}
	mount -t overlay -o lowerdir=sqsh_layer-vmm:sqsh_layer-server:sqsh_layer-es,upperdir=upper_layer/upper,workdir=upper_layer/work none sysroot
	touch $@

.build/RUCKUS-WORKSTATION-PPAS: .build/RUCKUS-WORKSTATION-OVL
	$(MAKE) sysroot-mount-vfs
	$(MAKE) PPAS_WORKSTATION
	touch $@

# add 32bit architecture support
.build/RUCKUS-WORKSTATION-ARCH: .build/RUCKUS-WORKSTATION-PPAS
	$(MAKE) sysroot-mount-vfs
	$(CHROOT) dpkg --add-architecture i386
	$(CHROOT) apt update
	touch $@

# apt install
#
# NOTE: We do --install-recommends for this one!  Bring on the kitchen
#       sink!
#
# FIXME: Is there something we can do to force the inclusion of snap
#        pacakges?  From a built system, if you apt install firefox or
#        thunderbird it automatically installs the snap (and any
#        requirements).
#
# FIXME: Also, and probably related, nothing is installing the base
#        snaps found on a fresh Ubuntu 24.04 install.  Here's what I
#        see on one of those (actualy this list is from a box that
#        went focal->jammy->noble so might have extra cruft):
#
#        bare			(installed by firefox)
#        core22			(installed by firefox)
#        firefox		(definately need this)
#        firmware-updater
#        gnome-42-2204		(installed by firefox)
#        gtk-common-themes	(installed by firefox)
#        snap-store
#        snapd			(installed by firefox)
#        snapd-desktop-integration
#
#        the noble install disk also had these installed:
#
#        ubuntu-desktop-bootstrap
#
#        Iteresting... I thought thunderbird was a part of
#        ubuntu-desktop, but it's not installed after a default
#        installation.  Oooh, it *is* a part of ubuntu-desktop, but
#        the default package list during graphical installation
#        doesn't install ubuntu-desktop.  It also doesn't installe
#        libreoffice, file-roller, gnome-calendar, gnome-snapshot,
#        gstreamer, remmina, rhythmbox, shotwell, totem, ure, or
#        xorriso.
#
#        The only additional snap installed by installing
#        ubuntu-desktop was thunderbird
#
# FIXME: looking at snaps intstalled by default for 26.04 now.  on the
#        install media, the casper/minimal.en.manifest.full file has
#        this at the end (I sorted it alphabetically for comparison's
#        sake)... I bet that's the list of snaps that get installed.
#        Let's verify that, shall we?
#
#        snap:bare       stable  5
#        snap:core24     stable  1587
#        snap:desktop-security-center    1/stable/ubuntu-26.04   150
#        snap:firefox    stable/ubuntu-26.04     8107
#        snap:firmware-updater   1/stable/ubuntu-26.04   226
#        snap:gnome-46-2404      stable/ubuntu-26.04     153
#        snap:gtk-common-themes  stable/ubuntu-26.04     1535
#        snap:mesa-2404  stable/ubuntu-26.04     1165
#        snap:prompting-client   1/stable/ubuntu-26.04   204
#        snap:snap-store 2/stable/ubuntu-26.04   1367
#        snap:snapd      stable  26865
#        snap:snapd-desktop-integration  stable/ubuntu-26.04     361
#
#        oh and look what else I found in casper/minimal.squashfs
#
# drwxr-xr-x root/root                75 2026-04-23 11:21 squashfs-root/var/lib/snapd/seed
# drwxr-xr-x root/root               458 2026-04-23 11:21 squashfs-root/var/lib/snapd/seed/assertions
# -rw-r--r-- root/root              1666 2026-04-23 11:20 squashfs-root/var/lib/snapd/seed/assertions/account
# -rw-r--r-- root/root              2439 2026-04-23 11:20 squashfs-root/var/lib/snapd/seed/assertions/account-key
# -rw-r--r-- root/root              4521 2026-04-23 11:21 squashfs-root/var/lib/snapd/seed/assertions/bare_5.assert
# -rw-r--r-- root/root              4556 2026-04-23 11:20 squashfs-root/var/lib/snapd/seed/assertions/core24_1587.assert
# -rw-r--r-- root/root              4853 2026-04-23 11:20 squashfs-root/var/lib/snapd/seed/assertions/desktop-security-center_150.assert
# -rw-r--r-- root/root              7926 2026-04-23 11:20 squashfs-root/var/lib/snapd/seed/assertions/firefox_8107.assert
# -rw-r--r-- root/root              4976 2026-04-23 11:20 squashfs-root/var/lib/snapd/seed/assertions/firmware-updater_226.assert
# -rw-r--r-- root/root              4743 2026-04-23 11:21 squashfs-root/var/lib/snapd/seed/assertions/gnome-46-2404_153.assert
# -rw-r--r-- root/root              5195 2026-04-23 11:21 squashfs-root/var/lib/snapd/seed/assertions/gtk-common-themes_1535.assert
# -rw-r--r-- root/root              4735 2026-04-23 11:21 squashfs-root/var/lib/snapd/seed/assertions/mesa-2404_1165.assert
# -rw-r--r-- root/root               957 2026-04-23 11:20 squashfs-root/var/lib/snapd/seed/assertions/model
# -rw-r--r-- root/root              4647 2026-04-23 11:21 squashfs-root/var/lib/snapd/seed/assertions/prompting-client_204.assert
# -rw-r--r-- root/root              5583 2026-04-23 11:21 squashfs-root/var/lib/snapd/seed/assertions/snap-store_1367.assert
# -rw-r--r-- root/root              5195 2026-04-23 11:21 squashfs-root/var/lib/snapd/seed/assertions/snapd-desktop-integration_361.assert
# -rw-r--r-- root/root              4556 2026-04-23 11:20 squashfs-root/var/lib/snapd/seed/assertions/snapd_26865.assert
# -rw-r--r-- root/root              1099 2026-04-23 11:21 squashfs-root/var/lib/snapd/seed/seed.yaml
# drwxr-xr-x root/root               423 2026-04-23 11:21 squashfs-root/var/lib/snapd/seed/snaps
# -rw------- root/root              4096 2026-04-23 11:21 squashfs-root/var/lib/snapd/seed/snaps/bare_5.snap
# -rw------- root/root          70094848 2026-04-23 11:20 squashfs-root/var/lib/snapd/seed/snaps/core24_1587.snap
# -rw------- root/root          20602880 2026-04-23 11:20 squashfs-root/var/lib/snapd/seed/snaps/desktop-security-center_150.snap
# -rw------- root/root         286994432 2026-04-23 11:20 squashfs-root/var/lib/snapd/seed/snaps/firefox_8107.snap
# -rw------- root/root          17301504 2026-04-23 11:20 squashfs-root/var/lib/snapd/seed/snaps/firmware-updater_226.snap
# -rw------- root/root         635518976 2026-04-23 11:21 squashfs-root/var/lib/snapd/seed/snaps/gnome-46-2404_153.snap
# -rw------- root/root          96141312 2026-04-23 11:21 squashfs-root/var/lib/snapd/seed/snaps/gtk-common-themes_1535.snap
# -rw------- root/root         414167040 2026-04-23 11:21 squashfs-root/var/lib/snapd/seed/snaps/mesa-2404_1165.snap
# -rw------- root/root          19738624 2026-04-23 11:21 squashfs-root/var/lib/snapd/seed/snaps/prompting-client_204.snap
# -rw------- root/root          16449536 2026-04-23 11:21 squashfs-root/var/lib/snapd/seed/snaps/snap-store_1367.snap
# -rw------- root/root            593920 2026-04-23 11:21 squashfs-root/var/lib/snapd/seed/snaps/snapd-desktop-integration_361.snap
# -rw------- root/root          51654656 2026-04-23 11:20 squashfs-root/var/lib/snapd/seed/snaps/snapd_26865.snap
#
#       So, googling has revealed we can "seed" snapd and that on
#       first-boot these snaps will get installed (I assume quickly
#       w/out downloading a mess of stuff).  Links of interest:
#
#       https://forum.snapcraft.io/t/seed-new-snap-into-the-image/25301
#
#       The question then is: For our image builder, does it make more
#       sense to 1) snap install these in the crooted environment or
#       2) seed snapd like the ubuntu installer.
#
#       Since there's no easy way to download all these seeded files
#       from the installer.iso, it might make more sense to just go
#       with option 1 - chrooted snap install
#
#       ok, I cannot do `snap install` in a chrooted environemnt,
#       because it needs tons of fancy stuff runing that isn't... but
#       I *can* run snap download and setup the seeding from inside
#       the crooted environment.
#
SNAPS_WORKSTATION =
SNAPS_WORKSTATION += bare
SNAPS_WORKSTATION += core24
SNAPS_WORKSTATION += desktop-security-center
SNAPS_WORKSTATION += firefox
SNAPS_WORKSTATION += firmware-updater
SNAPS_WORKSTATION += gnome-46-2404
SNAPS_WORKSTATION += gtk-common-themes
SNAPS_WORKSTATION += mesa-2404
SNAPS_WORKSTATION += prompting-client
SNAPS_WORKSTATION += snap-store
SNAPS_WORKSTATION += snapd
SNAPS_WORKSTATION += snapd-desktop-integration
# we add thunderbird because it should have been installed when the
# dep was installed via apt.
SNAPS_WORKSTATION += thunderbird


.build/RUCKUS-WORKSTATION-APT: .build/RUCKUS-WORKSTATION-ARCH
	$(MAKE) sysroot-mount-vfs
	cat debconf-config.dat | $(CHROOT) $(APT_INSTALL) --install-recommends $(PKGS_WORKSTATION)
	touch $@

.build/RUCKUS-WORKSTATION-REMOVE: .build/RUCKUS-WORKSTATION-APT
	$(MAKE) sysroot-mount-vfs
	cat debconf-config.dat | $(CHROOT) $(APT_REMOVE) $(PKGS_WORKSTATION_REMOVE)
	touch $@

# FIXME: some of the snaps have channel's other than "stable" on the
#        official installation media.  For example,
#        desktop-security-center is 1/stable/ubuntu-26.04, firefox and
#        a bunch of others are stable/ubuntu-26.04.  What the impact
#        of me just specifying "stable" for all of them...?
#
#        hmm... actually, I'm explicitly downloading from the "stable"
#        channel so maybe it's fine?
#
.build/RUCKUS-WORKSTATION-SNAP: .build/RUCKUS-WORKSTATION-REMOVE
	$(MAKE) sysroot-mount-vfs
	mkdir -p sysroot/var/lib/snapd/seed/{assertions,snaps}
	cp -v snapd-seedfiles/{account,account-key,model} sysroot/var/lib/snapd/seed/assertions
	echo "snaps:" > sysroot/var/lib/snapd/seed/seed.yaml
	for name in $(SNAPS_WORKSTATION); do \
	  $(CHROOT) snap download --target-directory=/var/lib/snapd/seed/snaps \
	    --channel=stable $$name; \
	  echo "  -" >> sysroot/var/lib/snapd/seed/seed.yaml; \
	  echo "    name: $$name" >> sysroot/var/lib/snapd/seed/seed.yaml; \
	  echo "    channel: stable" >> sysroot/var/lib/snapd/seed/seed.yaml; \
	  snap=$$(basename $$(realpath sysroot/var/lib/snapd/seed/snaps/$$name*.snap)); \
	  echo "    file: $$snap" >> sysroot/var/lib/snapd/seed/seed.yaml; \
	done
	mv sysroot/var/lib/snapd/seed/snaps/*.assert sysroot/var/lib/snapd/seed/assertions/
	$(CHROOT) snap prepare-image --classic /var/lib/snapd/seed/assertions/model /var/lib/snapd/seed
	touch $@


sysroot/tmp/google-chrome.deb:
	wget -qO- https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb > $@

sysroot/tmp/google-earth.deb:
	wget -qO- https://dl.google.com/linux/direct/google-earth-pro-stable_current_amd64.deb > $@

sysroot/tmp/teamviewer.deb:
	wget -qO- https://download.teamviewer.com/download/linux/teamviewer_amd64.deb > $@

# FIXME: poo, this doesn't add an upstream ppa... but it does work
sysroot/tmp/zoom.deb:
	wget -qO- https://zoom.us/client/latest/zoom_amd64.deb > $@

sysroot/tmp/steam.deb:
	wget -qO- https://repo.steampowered.com/steam/archive/precise/steam_latest.deb > $@

.PHONY: DOWNLOADS_WORKSTATION
DOWNLOADS_WORKSTATION: sysroot/tmp/google-chrome.deb
DOWNLOADS_WORKSTATION: sysroot/tmp/google-earth.deb
DOWNLOADS_WORKSTATION: sysroot/tmp/teamviewer.deb
DOWNLOADS_WORKSTATION: sysroot/tmp/zoom.deb
DOWNLOADS_WORKSTATION: sysroot/tmp/steam.deb

PKGS_WORKSTATION_LOCAL =
PKGS_WORKSTATION_LOCAL += google-chrome
PKGS_WORKSTATION_LOCAL += google-earth
PKGS_WORKSTATION_LOCAL += teamviewer
PKGS_WORKSTATION_LOCAL += zoom
PKGS_WORKSTATION_LOCAL += steam

# fetch debs and install locally
.build/RUCKUS-WORKSTATION-LOCAL: .build/RUCKUS-WORKSTATION-SNAP
	$(MAKE) sysroot-mount-vfs
	$(MAKE) DOWNLOADS_WORKSTATION
	cat debconf-config.dat | $(CHROOT) $(APT_INSTALL) --install-recommends $(patsubst %,/tmp/%.deb,$(PKGS_WORKSTATION_LOCAL))
	touch $@

# service enable/disable, tweaks, etc
#
# FIXME: audit services
#
# FIXME: I've noticed early on that there's a new policy file the
#        explicitly prohibits auto-mounting of removable ext4
#        filesystems... which I find offensively stupid and annoying.
#        I'm just not sure what the *best* fix is yet.
#
#        /usr/lib/udev/rules.d/64-ext4.rules
#
#        could sed a # into the the "^SUBSYSTEM" line, but not sure
#        how that will jive with apt upgrades down the road.
#
.build/RUCKUS-WORKSTATION-FINAL: .build/RUCKUS-WORKSTATION-LOCAL
	$(MAKE) sysroot-mount-vfs
	rm -rfv sysroot/tmp/*
	sed -i 's|^SUBSYSTEM|#SUBSYSTEM|' sysroot/usr/lib/udev/rules.d/64-ext4.rules
	sed -i 's|^SUBSYSTEM|#SUBSYSTEM|' sysroot/usr/lib/udev/rules.d/64-xfs.rules
	echo "# don't let udisks make borky disk-mount buttons for zpools" > sysroot/usr/lib/udev/rules.d/99-hide-zpools.rules
	echo 'SUBSYSTEM=="block", ENV{ID_FS_TYPE}=="zfs_member", ENV{UDISKS_IGNORE}="1"'  >> sysroot/usr/lib/udev/rules.d/99-hide-zpools.rules
	touch $@

workstation.sqsh: .build/RUCKUS-WORKSTATION-FINAL
	$(MAKE) sysroot-umount-vfs
	mksquashfs upper_layer/upper $@ $(SQSH_COMP) -noappend -wildcards



# Targets for building bootable installers

installer-common/.DONE:
	mkdir -p installer-common/images
	for x in $(SQSH_FILES); do \
	  if [ $$(stat -c %s $$x) -ge $$((4*1024*1024*1024)) ]; then \
	    split -b 2G -d --verbose $$x installer-common/images/$$(basename $$x)SPLIT ; \
	  else \
	    ln -vs $$x installer-common/images/ ; \
	  fi \
	done
	ln -vs $(abs_srcdir)/sqsh_layers.conf installer-common/images/
	ln -vs $(FW_FILES) installer-common/
	ln -vs $(KERNEL_FILES) installer-common/
	touch $@

installer-iso/.DONE: installer-common/.DONE
	pushd installer-common && \
	  find -L * -type d -exec mkdir -vp ../installer-iso/{} \; && \
	  find -L * -type f -exec ln -vs $(abs_srcdir)/installer-common/{} ../installer-iso/{} \; && \
	  popd
	mkdir -p installer-iso/isolinux
	ln -vs $(ISOLINUX_FILES) installer-iso/isolinux/
	ln -vs $$(realpath syslinux.cfg) installer-iso/isolinux/isolinux.cfg
	ln -vs $$(realpath sourceruckus*.png) installer-iso/isolinux/
	mkdir -p installer-iso/EFI/boot
	ln -vs $(SYSLINUX_FILES) installer-iso/EFI/boot/
	mv -v installer-iso/EFI/boot/syslinux.efi installer-iso/EFI/boot/bootx64.efi
	ln -vs $$(realpath syslinux.cfg) installer-iso/EFI/boot/
	ln -vs $$(realpath sourceruckus*.png) installer-iso/EFI/boot/
	touch $@

# FIXME: I can't seen to figure out any way to boot syslinux or
#        isolinux via EFI from an ISO... I'm starting to suspect you
#        just can't (i.e., becuase isolinux only supports BIOS and
#        syslinux doesn't support ISO9660), which is awkward.
#
#        This does create a perfectly usable ISO that boots in BIOS
#        mode, but then the installer cannot access EFIVARS
#
#        Note the -eltorito-alt-boot .... -no-emul-boot lines
#
#        https://wiki.syslinux.org/wiki/index.php?title=ISOLINUX
#        https://wiki.syslinux.org/wiki/index.php?title=Isohybrid
#        https://unix.stackexchange.com/questions/283846/why-can-my-custom-build-iso-not-boot-in-efi
#        https://unix.stackexchange.com/questions/312789/making-a-uefi-bootable-image-with-mkisofs-and-eltorito-boot-efiboot-img
#
#        For now, we'll still create the .iso, and include the
#        syslinux stuff that doesn't work because it doesn't hurt
#        anything and maybe I'm just doing it slightly wrong?  But
#        we'll also create an installer-usb directory that can just be
#        copied to a vfat flash disk to be an EFI bootable installer
#
#        We could revisit this and switch from syslinux to
#        systemd-boot or grub2 to get a bootable iso...
#
sourceruckus.iso: installer-iso/.DONE
	genisoimage -rdlf -V sourceruckus \
	  -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot \
	  -boot-load-size 4 -boot-info-table \
	  -eltorito-alt-boot \
	  -e EFI/boot/bootx64.efi \
	  -no-emul-boot \
	  -o $@ -m .DONE installer-iso
	$(ISOHYBRID) --uefi $@

# NOTE: This ruckususb.img is bootable on traditional BIOS systems
#       (via EXTLINUX) and on EFI systems (via SYSLINUX).
#
#       This is accomplished by creating a small BIOS boot partition
#       (marked w/ the LegacyBIOSBootable attribute) and a larger vfat
#       partition with all the installer files and EFI boot files.
#
#       p1: 1G ext4: bare minimum BIOS boot files (kernel, initrd,
#       extlinux)
#
#       p2: 12G fat: EFI boot files (kernel, initrd, syslinux),
#       fw.sqsh, squashfs images
#
#       EXTLINUX can boot from a GPT disk (thankfully) using the
#       special gptmbr.bin instead of plain old mbr.bin (and marking
#       it with the LegacyBIOSBootable attribute).
#
#       The net effect is that the disk will definately be UFI
#       bootable and will *probably* be BIOS bootable in the off
#       chance your system doesn't support EFI.
#
installer-usb/.DONE: installer-common/.DONE
	pushd installer-common && \
	  find -L * -type d -exec mkdir -vp ../installer-usb/{} \; && \
	  find -L * -type f -exec ln -vs $(abs_srcdir)/installer-common/{} ../installer-usb/{} \; && \
	  popd
	mkdir -p installer-usb/EFI/boot
	ln -vs $(SYSLINUX_FILES) installer-usb/EFI/boot/
	mv -v installer-usb/EFI/boot/syslinux.efi installer-usb/EFI/boot/bootx64.efi
	cat syslinux.cfg | sed 's|=sourceruckus|=ruckususb|g' > installer-usb/EFI/boot/syslinux.cfg
	ln -vs $$(realpath sourceruckus*.png) installer-usb/EFI/boot/
	touch $@

installer-usb-boot/.DONE:
	mkdir -p installer-usb-boot/extlinux
	ln -vs $(KERNEL_FILES) installer-usb-boot/
	ln -vs $(EXTLINUX_FILES) installer-usb-boot/extlinux/
	cat syslinux.cfg | sed -e 's|=sourceruckus|=ruckususb|g' \
	  -e 's|menu.c32|vesamenu.c32|' \
	  -e 's|installer=EFI=force|installer=BOOT=1G|' > installer-usb-boot/extlinux/extlinux.conf
	ln -vs $$(realpath sourceruckus*.png) installer-usb-boot/extlinux/
	touch $@

# Using dd to write this img file to a flash drive results in a
# BIOS/UEFI bootable installer, but requires us to run this target as
# root.  I suppose that's not a huge deal since the all the
# debootstrap stuff above needs root privs, too.
#
# NOTE: Once dd-ing this onto a flash drive, the kernel will complain
#       about the partition table not ending at the end of the disk
#       (assuming your flash drive isn't exactly 12G).  It doesn't
#       hurt anything, but if you care, this warning can be fixed by
#       running `fdisk /dev/whatever`, going into expert mode, doing
#       `v`, and following the instructions.
#
# FIXME: why do we use sudo all over in here when we don't with the
#        debootstrap stuff above...?  i suppose it's slightly safer to
#        only run the bits as root that have to be?
#
ruckususb.img: installer-usb/.DONE installer-usb-boot/.DONE
	rm -f $@.tmp
	dd if=/dev/zero of=$@.tmp bs=1M seek=13K count=0
	echo -e "label:gpt\nsize=1G,attrs=LegacyBIOSBootable\n," | sfdisk $@.tmp
	mkdir -p t
	loop=$$(sudo losetup -f) && \
	  sudo losetup -P $$loop $@.tmp && \
	  sudo mkfs.ext4 -L ruckususb-boot -O^64bit $${loop}p1 && \
	  sudo mount $${loop}p1 t && \
	  sudo cp -vrL --preserve=mode,timestamps installer-usb-boot/* t/ && \
	  sudo $(EXTLINUX) --install t/extlinux && \
	  sudo umount t && \
	  sudo sh -c "cat $(EXTLINUX_GPTMBR) > $$loop" && \
	  sudo mkfs.vfat -n ruckususb $${loop}p2 && \
	  sudo mount $${loop}p2 t && \
	  sudo cp -vrL --preserve=mode,timestamps installer-usb/* t/ && \
	  sudo umount t && \
	  sudo losetup -d $$loop
	rmdir t
	mv $@.tmp $@


#
# FIXME: could aslo make a target for just creating an empty flash
#        drive with kernel/initrd/fw/syslinux... (i.e., leave out the
#        images, just for initramsys resuce disk) we'd want to sed the
#        installer lines out of syslinux.cfg if we do that.
#
#ruckusrescue.img


installer-clean:
	rm -rf installer-*
	rm -f sourceruckus.iso
	rm -f ruckususb.img
