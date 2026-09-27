AC_PREREQ([2.65])
m4_define(sr_major_version, 1)
m4_define(sr_minor_version, 0)
m4_define(sr_micro_version, 0)
m4_define(sr_devtag, [dev])
m4_ifnblank(sr_devtag,
	[m4_define(sr_version, sr_major_version.sr_minor_version.sr_micro_version-sr_devtag)],
	[m4_define(sr_version, sr_major_version.sr_minor_version.sr_micro_version)])

# FIXME: can i get a timestamp in here to append to sqsh images?
#        didn't i used to do that somewhere...?

AC_INIT([Source Ruckus Resolute System Builder], [sr_version], [veggiemike@sourceruckus.org], [sourceruckus-resolute])
AC_CONFIG_AUX_DIR([.])
#AC_CONFIG_MACRO_DIR([m4])

AC_SUBST(SR_MAJOR_VERSION, sr_major_version)
AC_SUBST(SR_MINOR_VERSION, sr_minor_version)
AC_SUBST(SR_MICRO_VERSION, sr_micro_version)
AC_SUBST(SR_DEVTAG, sr_devtag)

AM_INIT_AUTOMAKE([dist-xz no-dist-gzip tar-pax foreign subdir-objects color-tests])

AC_PROG_INSTALL
AC_PROG_MKDIR_P
AC_PROG_LN_S

# check for sed /w --in-place
#
# NOTE: Using sed -i in makefiles is frowned upon because it's not
#       portable.  We make 'sed -i' a requirement and check for it here
#       because... I like it and I think pretty much everyone has it.  But
#       I do want to get bonked upside the head if it's going to cause
#       problems.
#
AC_PROG_SED
AS_TMPDIR(sr-)
AC_CACHE_CHECK([whether $ac_cv_path_SED supports -i], [sr_cv_sed_i],
  [AS_IF([touch $tmp/sed && $ac_cv_path_SED -i 's|||' $tmp/sed 2>/dev/null],
      sr_cv_sed_i=yes,
      sr_cv_sed_i=no)])
rm -f $tmp/sed && rmdir $tmp
if test $sr_cv_sed_i != yes; then
    AC_MSG_ERROR([sed -i is required])
fi

# check for needed programs
AC_DEFUN([NEEDED_PROGS],
	 [dnl list them here
	  wget
	  bash
	 ])

m4_foreach_w([P], NEEDED_PROGS,
  [AC_PATH_PROG(m4_bpatsubst(m4_toupper(P), [\.], [_]), P)
   AS_IF([test "$m4_bpatsubst(m4_toupper(P), [\.], [_])" = ""],
     AC_MSG_ERROR([Could not find P]))
  ])


# FIXME: check sudo skills or that we're running as root

# manage image list
img_list=es.sqsh
AC_ARG_ENABLE(es-file,
  AS_HELP_STRING([--disable-es-file], [disable building specialized embedded file-server image, es-file.sqsh]),
  es_file=$enableval,
  es_file=yes)
AS_IF([test "x$es_file" = "xyes"], img_list="$img_list es-file.sqsh")
AC_ARG_ENABLE(es-auth,
  AS_HELP_STRING([--disable-es-auth], [disable building specialized embedded auth-server image, es-auth.sqsh]),
  es_auth=$enableval,
  es_auth=yes)
AS_IF([test "x$es_auth" = "xyes"], img_list="$img_list es-auth.sqsh")
AC_ARG_ENABLE(es-web,
  AS_HELP_STRING([--disable-es-web], [disable building specialized embedded web-server image, es-web.sqsh]),
  es_web=$enableval,
  es_web=yes)
AS_IF([test "x$es_web" = "xyes"], img_list="$img_list es-web.sqsh")
AC_ARG_ENABLE(server,
  AS_HELP_STRING([--disable-server], [disable building server image, server.sqsh (depends on es.sqsh)]),
  server=$enableval,
  server=yes)
AS_IF([test "x$server" = "xyes"], img_list="$img_list server.sqsh")
AC_ARG_ENABLE(vmm,
  AS_HELP_STRING([--disable-vmm], [disable building virtual machine monitor image, vmm.sqsh (depends on server.sqsh)]),
  vmm=$enableval,
  vmm=yes)
AS_IF([test "x$vmm" = "xyes" -a "x$server" = "xyes"], img_list="$img_list vmm.sqsh")
AC_ARG_ENABLE(workstation,
  AS_HELP_STRING([--disable-workstation], [disable building workstation image, workstation.sqsh (depends on vmm.sqsh)]),
  workstation=$enableval,
  workstation=yes)
AS_IF([test "x$workstation" = "xyes" -a "x$vmm" = "xyes"], img_list="$img_list workstation.sqsh")


# build a devel-enabled base image
AM_CONDITIONAL([ES_DEVEL], [test x$ES_DEVEL = xyes])
AC_ARG_ENABLE(devel,
  AS_HELP_STRING([--enable-devel], [enable building special embedded devolopemnt server, es-devel.sqsh, instead of the normal images]),
  es_devel=$enableval,
  es_devel=no)
AS_IF([test "x$es_devel" = "xyes"], img_list=es-devel.sqsh)
AC_SUBST(ES_DEVEL, $es_devel)
AM_CONDITIONAL([ES_DEVEL], [test x$ES_DEVEL = xyes])

AC_SUBST(IMG_LIST, $img_list)


# NOTE: The extra `echo` here is to force variables to be a single space-delimited line
#
# FIXME: these should have sane defaults, but i'm setting them to my
#        specific system during development.
#
KERNEL_FILES=$(echo $(realpath /scrap/bootfiles/*6.18.y*))
SYSLINUX_FILES=$(echo $(realpath /scrap/ruckusrd/subprojects/syslinux/FOO/usr/share/syslinux/efi64/*))
ISOLINUX_FILES=$(echo $(realpath /scrap/ruckusrd/subprojects/syslinux/FOO/usr/share/syslinux/{isolinux.bin,{ldlinux,libcom32,libmenu,libutil,menu,vesamenu,chain}.c32}))
EXTLINUX_FILES=$(echo $(realpath /scrap/ruckusrd/subprojects/syslinux/FOO/usr/share/syslinux/*.c32))
EXTLINUX_GPTMBR=$(echo $(realpath /scrap/ruckusrd/subprojects/syslinux/FOO/usr/share/syslinux/gptmbr.bin))
EXTLINUX=$(echo $(realpath /scrap/ruckusrd/subprojects/syslinux/FOO/sbin/extlinux))
ISOHYBRID=/scrap/ruckusrd/subprojects/syslinux/FOO/usr/bin/isohybrid
FW_FILES=$(echo $(realpath /scrap/ruckusrd/subprojects/fw.sqsh))
SQSH_FILES=$(echo $(realpath /scrap/images/resolute/*.sqsh))

# NOTE: This is going to select the LAST specified kernel and initrd
#       file, in case there are duplicates.  The thinking being, if
#       you want to override the system's initrd you can just add it
#       to KERNEL_FILES last.
#
BOOTKERNEL=$(basename $(echo $KERNEL_FILES | tr ' ' '\n' | grep vmlinuz | tr '\n' ' ' | awk '{print $NF}'))
BOOTINITRD=$(basename $(echo $KERNEL_FILES | tr ' ' '\n' | grep initrd | tr '\n' ' ' | awk '{print $NF}'))

AC_ARG_VAR(KERNEL_FILES, [Kernel files to include on the installer.])
AC_ARG_VAR(SYSLINUX_FILES, [SYSLINUX files to include on the installer.])
AC_ARG_VAR(EXTLINUX_FILES, [EXTLINUX files to include on the installer.])
AC_ARG_VAR(EXTLINUX_GPTMBR, [Path to gptmbr.bin.])
AC_ARG_VAR(EXTLINUX, [Path to extlinux.])
AC_ARG_VAR(ISOLINUX_FILES, [ISOLINUX files to include on the installer.])
AC_ARG_VAR(ISOHYBRID, [Path to isohybrid.])
AC_ARG_VAR(FW_FILES, [Firmware files to include on the installer.])
AC_ARG_VAR(SQSH_FILES, [Sqsh files to include on the installer.])
AC_ARG_VAR(BOOTKERNEL, [The kernel file for the installer to boot.])
AC_ARG_VAR(BOOTINITRD, [The initrd file for the installer to boot.])

AC_CONFIG_FILES([Makefile
		 syslinux.cfg
		 ])
AC_OUTPUT

echo
echo
echo
echo "------------------------------------------------------------------------"
echo "$PACKAGE_STRING"
echo "------------------------------------------------------------------------"
echo
echo "Configuration Options Summary:"
echo
echo "Enabled Sqsh Images:"
echo "  $img_list"
echo
#echo "Kernel Files: $KERNEL_FILES"
#echo "SYSLINUX Files: $SYSLINUX_FILES"
#echo "EXTLINUX Files: $EXTLINUX_FILES"
#echo "gptmbr.bin: $EXTLINUX_GPTMBR"
#echo "extlinux: $EXTLINUX"
#echo "ISOLINUX Files: $ISOLINUX_FILES"
#echo "isohybrid: $ISOHYBRID"
#echo "Firmware Files: $FW_FILES"
#echo "Sqsh Files: $SQSH_FILES"
#echo "BOOTKERNEL: $BOOTKERNEL"
#echo "BOOTINITRD: $BOOTINITRD"
#echo
