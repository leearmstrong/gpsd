### SCons build recipe for the GPSD project

# Important targets:
#
# build     - build the software (default)
# dist      - make distribution tarball
# install   - install programs, libraties, and manual pages
# uninstall - undo an install
#
# check     - run regression and unit tests.
# splint    - run the splint static tester on the code
# cppcheck  - run the cppcheck static tester on the code
# xmllint   - run xmllint on the documentation
# testbuild - test-build the code from a tarball

# Unfinished items:
# * Qt binding
# * Out-of-directory builds: see http://www.scons.org/wiki/UsingBuildDir

# Release identification begins here
gpsd_version = "3.0~dev"
libgps_major = 20
libgps_minor = 0
libgps_age   = 0
# Release identification ends here

EnsureSConsVersion(1,2,0)

import os, sys, commands, glob
from distutils.util import get_platform

#
# Build-control options
#

def internalize(s):
    return s.replace('-', '_')

boolopts = (
    # GPS protocols
    ("nmea",          True,  "NMEA support"),
    ("ashtech",       True,  "Ashtech support"),
    ("earthmate",     True,  "DeLorme EarthMate Zodiac support"),
    ("evermore",      True,  "EverMore binary support"),
    ("fv18",          True,  "San Jose Navigation FV-18 support"),
    ("garmin",        True,  "Garmin kernel driver support"),
    ("garmintxt",     True,  "Garmin Simple Text support"),
    ("geostar",       True,  "Geostar Protocol support"),
    ("itrax",         True,  "iTrax hardware support"),
    ("mtk3301",       True,  "MTK-3301 support"),
    ("navcom",        True,  "Navcom support"),
    ("oncore",        True,  "Motorola OnCore chipset support"),
    ("sirf",          True,  "SiRF chipset support"),
    ("superstar2",    True,  "Novatel SuperStarII chipset support"),
    ("tnt",           True,  "True North Technologies support"),
    ("tripmate",      True,  "DeLorme TripMate support"),
    ("tsip",          True,  "Trimble TSIP support"),
    ("ubx",           True,  "UBX Protocol support"),
    # Non-GPS protocols
    ("aivdm",         True,  "AIVDM support"),
    ("gpsclock",      True,  "GPSClock support"),
    ("ntrip",         True,  "NTRIP support"),
    ("oceanserver",   True,  "OceanServer support"),
    ("rtcm104v2",     True,  "rtcm104v2 support"),
    ("rtcm104v3",     True,  "rtcm104v3 support"),
    # Time service
    ("ntpshm",        True,  "NTP time hinting support"),
    ("pps",           True,  "PPS time syncing support"),
    ("pps_on_cts",    False, "PPS pulse on CTS rather than DCD"),
    # Export methods
    ("socket-export", True,  "data export over sockets"),
    ("dbus-export",   False,  "enable DBUS export support"),
    ("shm-export",    True,  "export via shared memory"),
    # Communication
    ("bluez",         False, "BlueZ support for Bluetooth devices"),
    ("ipv6",          True,  "build IPv6 support"),
    # Client-side options
    ("clientdebug",   True,  "client debugging support"),
    ("oldstyle",      True,  "oldstyle (pre-JSON) protocol support"),
    ("libgpsmm",      True,  "build C++ bindings"),
    ("libQgpsmm",     False, "build QT bindings"),
    ("reconfigure",   True,  "allow gpsd to change device settings"),
    ("controlsend",   True,  "allow gpsctl/gpsmon to change device settings"),
    ("cheapfloats",   True,  "float ops are cheap, compute error estimates"),
    ("squelch",       False, "squelch gpsd_report/gpsd_hexdump to save cpu"),
    # Miscellaneous
    ("profiling",     False, "Build with profiling enabled"),
    ("timing",        True,  "latency timing support"),
    ("control-socket",True,  "control socket for hotplug notifications")
    )
for (name, default, help) in boolopts:
    internal_name = internalize(name)
    if default:
        AddOption('--disable-'+ name,
                  dest=internal_name,
                  default=True,
                  action="store_false",
                  help=help)
    else:
        AddOption('--enable-'+ name,
                  dest=internal_name,
                  default=False,
                  action="store_true",
                  help=help)

nonboolopts = (
    ("gpsd-user",   "USER",     "privilege revocation user",     ""),
    ("gpsd-group",  "GROUP",    "privilege revocation group",    ""),
    
    ("prefix",      "PREFIX",   "installation directory prefix", "/usr/local/"),

    ("limited-max-clients", "CLIENTS",  "maximum allowed clients",       0),
    ("limited-max-devices", "DEVICES",  "maximum allowed devices",       0),
    ("fixed-port-speed",    "SPEED",    "fixed serial port speed",       0),
    )
for (name, metavar, help, default) in nonboolopts:
        internal_name = internalize(name)
        AddOption('--enable-'+ name,
                  type='string',
                  dest=internal_name,
                  metavar=metavar,
                  default=default,
                  nargs=1,
                  action="store",
                  help=help)

AddOption("--prefix",
          type="string",
          dest="prefix",
          metavar="PREFIX",
          default="/usr/local",
          nargs=1,
          action="store",
          help="installation path prefix")

pathopts = (
    ("sysconfdir",  "SYCONFDIR","system configuration directory","/etc"),
    ("bindir",      "BINDIR",   "application binaries directory","/bin"),
    ("libdir",      "LIBDIR",   "dystem libraries",              "/lib"),
    ("sbindir",     "SBINDIR",  "system binaries directory",     "/sbin"),
    ("mandir",      "MANDIR",   "manual pages directory",        "/share/man"),
    ("docdir",      "DOCDIR",   "documents directory",           "/share/doc"),
    )
for (name, metavar, help, default) in pathopts:
    internal_name = internalize(name)
    AddOption('--' + name,
              type='string',
              metavar=metavar,
              default=default,
              nargs=1,
              action="store",
              help=help)

#
# Environment creation
#

env = Environment(tools=["default", "tar"])
env.SConsignFile(".sconsign.dblite")

env['VERSION'] = gpsd_version

env.Append(LIBPATH=['.'])

# Placeholder so we can kluge together something like VPATH builds
# $SRCDIR replaces occurrences for $(srcdir) in the autotools build.
env['SRCDIR'] = '.'

# Because absolute paths are safer
for variant in ['python2.7', 'python2.6', 'python2.5', 'python2.4', "python"]:
    python = WhereIs(variant)
    if python:
        env["PYTHON"] = python
        break
else:
    print "No Python - how are you running this script?"
    Exit(1)

if env['CC'] == 'gcc':
    # Enable all GCC warnings except uninitialized and
    # missing-field-initializers, which we can't help triggering because
    # of the way some of the JSON code is generated.
    # Also not including -Wcast-qual
    env.Append(CFLAGS=Split('''-Wextra -Wall -Wno-uninitialized
                            -Wno-missing-field-initializers -Wcast-align
                            -Wmissing-declarations -Wmissing-prototypes
                            -Wstrict-prototypes -Wpointer-arith -Wreturn-type
                            -D_GNU_SOURCE'''))

# Tell generated binaries to look in the current directory for
# shared libraries. Should be handled sanely by scons on all systems.
# Not good to use '.' or a relative path here; it's a security risk.
# At install time we should use chrpath to remove RPATH from the executables
# again.
env.Append(RPATH=os.path.realpath(os.curdir))

# Give deheader a way to set compiler flags
if 'MORECFLAGS' in os.environ:
    env.Append(CFLAGS=Split(os.environ['MORECFLAGS']))

# Should we build with profiling?
if ARGUMENTS.get('profiling', 0):
    env.Append(CCFLAGS=['-pg'])
    env.Append(LDFLAGS=['-pg'])

# Get a slight speedup by not doing automatic RCS and SCCS fetches.
env.SourceCode('.', None)

# Should we build with debug symbols?
if ARGUMENTS.get('debug', 0):
    env.Append(CCFLAGS=['-g'])
    env.Append(CCFLAGS=['-O0'])
else:
    env.Append(CCFLAGS=['-O2'])

## Build help

if GetOption("help"):
    Return()

## Configuration

def CheckPKG(context, name):
    context.Message( 'Checking for %s... ' % name )
    ret = context.TryAction('pkg-config --exists \'%s\'' % name)[0]
    context.Result( ret )
    return ret

def CheckChrpath(context):
    context.Message( 'Checking for chrpath... ')
    ret = context.TryAction('chrpath -v')[0]
    context.Result( ret )
    return ret

config = Configure(env, custom_tests = { 'CheckPKG' : CheckPKG, 'CheckChrpath' : CheckChrpath })

confdefs = ["/* gpsd_config.h.  Generated by scons, do not hand-hack.  */\n\n"]

confdefs.append('#define VERSION "%s"\n\n' % gpsd_version)

cxx = config.CheckCXX()

for f in ("daemon", "strlcpy", "strlcat"):
    if config.CheckFunc(f):
        confdefs.append("#define HAVE_%s 1\n\n" % f.upper())
    else:
        confdefs.append("/* #undef HAVE_%s */\n\n" % f.upper())

if config.CheckLib('ncurses'):
    ncurseslibs = ['ncurses']
else:
    ncurseslibs = []

if config.CheckPKG('libusb-1.0'):
    confdefs.append("#define HAVE_LIBUSB 1\n\n")
    env.MergeFlags(['!pkg-config libusb-1.0 --cflags'])
    flags = env.ParseFlags('!pkg-config libusb-1.0 --libs')
    usblibs = flags['LIBS']
else:
    confdefs.append("/* #undef HAVE_LIBUSB */\n\n")
    usblibs = []

if config.CheckLib('librt'):
    confdefs.append("#define HAVE_LIBRT 1\n\n")
    # System library - no special flags
    rtlibs = ["rt"]
else:
    confdefs.append("/* #undef HAVE_LIBRT */\n\n")
    rtlibs = []

dbus_export_value = GetOption(internalize('dbus-export'))
if type(dbus_export_value) == type(True) and dbus_export_value and config.CheckPKG('dbus-1') and config.CheckPKG('dbus-glib-1'):
    confdefs.append("#define HAVE_DBUS 1\n\n")
    env.MergeFlags(['!pkg-config --cflags dbus-glib-1 dbus-1'])
    dbus_xmit = env.ParseFlags('!pkg-config --libs dbus-1')
    dbus_xmit_libs = dbus_xmit['LIBS']
    dbus_recv = env.ParseFlags('!pkg-config --libs dbus-glib-1')
    dbus_recv_libs = dbus_recv['LIBS']
else:
    confdefs.append("/* #undef HAVE_DBUS */\n\n")
    dbus_xmit_libs = []
    dbus_recv_libs = []

if config.CheckPKG('bluez'):
    confdefs.append("#define HAVE_BLUEZ 1\n\n")
    env.MergeFlags(['!pkg-config bluez --cflags'])
    flags = env.ParseFlags('!pkg-config bluez --libs')
    bluezlibs = flags['LIBS']
else:
    confdefs.append("/* #undef HAVE_BLUEZ */\n\n")
    bluezlibs = []

if config.CheckHeader("sys/timepps.h"):
    confdefs.append("#define HAVE_SYS_TIMEPPS_H 1\n\n")
else:
    confdefs.append("/* #undef HAVE_SYS_TIMEPPS_H */\n\n")

if config.CheckChrpath():
    have_chrpath = True
else:
    have_chrpath = False

# Map options to libraries required to support them that might be absent.
optionrequires = {
    "bluez": ["libbluez"],
    "dbus_export" : ["libdbus-1", "libdbus-glib-1"],
    }

keys = map(lambda x: (x[0],x[2]), boolopts) + map(lambda x: (x[0],x[2]), nonboolopts) + map(lambda x: (x[0],x[2]), pathopts)
keys.sort()
for (key,help) in keys:
    key = internalize(key)
    value = GetOption(key)

    if value and key in optionrequires:
        for required in optionrequires[key]:
            if not config.CheckLib(required):
                print "%s not found, %s cannot be enabled." % (required, key)
                value = False
                break

    # This is a kluge.  It's meant to make the behavior of this recipe be
    # plug-compatible with the autotools build.  When we discard the autotools
    # build, it can be removed.
    if key == "sysconfdir":
        env.Append(CFLAGS='-DSYSCONFDIR=\'"%s"\'' \
                   % os.path.join(GetOption("prefix"), value))
        continue

    confdefs.append("/* %s */\n"%help)
    if type(value) == type(True):
        if value:
            confdefs.append("#define %s_ENABLE 1\n\n" % key.upper())
        else:
            confdefs.append("/* #undef %s_ENABLE */\n\n" % key.upper())
    elif value in (0, ""):
        confdefs.append("/* #undef %s */\n\n" % key.upper())
    else:
        if type(value) == type(-1):
            confdefs.append("#define %d %s\n\n" % (key.upper(), value))
        elif type(value) == type(""):
            confdefs.append("#define %s \"%s\"\n\n" % (key.upper(), value))
        else:
            raise ValueError

confdefs.append('''
/* will not handle pre-Intel Apples that can run big-endian */
#if defined __BIG_ENDIAN__
#define WORDS_BIGENDIAN 1
#else
#undef WORDS_BIGENDIAN
#endif

/* Some libcs do not have strlcat/strlcpy. Local copies are provided */
#ifndef HAVE_STRLCAT
# ifdef __cplusplus
extern "C" {
# endif
size_t strlcat(/*@out@*/char *dst, /*@in@*/const char *src, size_t size);
# ifdef __cplusplus
}
# endif
#endif
#ifndef HAVE_STRLCPY
# ifdef __cplusplus
extern "C" {
# endif
size_t strlcpy(/*@out@*/char *dst, /*@in@*/const char *src, size_t size);
# ifdef __cplusplus
}
# endif
#endif

#define GPSD_CONFIG_H
''')

with open("gpsd_config.h", "w") as ofp:
    ofp.writelines(confdefs)

manbuilder = None
mangenerator = ''
if WhereIs("xsltproc"):
    mangenerator = 'xsltproc'
    docbook_url_stem = 'http://docbook.sourceforge.net/release/xsl/current/'
    docbook_man_uri = docbook_url_stem + 'manpages/docbook.xsl'
    docbook_html_uri = docbook_url_stem + 'html/docbook.xsl'
    build = "xsltproc --nonet %s $SOURCE >$TARGET"
    htmlbuilder = build % docbook_html_uri
    manbuilder = build % docbook_man_uri
elif WhereIs("xmlto"):
    mangenerator = 'xmlto'
    htmlbuilder = "xmlto html-nochunks $SOURCE; mv `basename $TARGET` $TARGET"
    manbuilder = "xmlto man $SOURCE; mv `basename $TARGET` $TARGET"
else:
    print "Neither xsltproc nor xmlto found, documentation cannot be built."
if manbuilder:
    env['BUILDERS']["Man"] = Builder(action=manbuilder)
    env['BUILDERS']["HTML"] = Builder(action=htmlbuilder,
                                      src_suffix=".xml", suffix=".html")

# Gentoo systems can have a problem with the Python path
if os.path.exists("/etc/gentoo-release"):
    print "This is a Gentoo system."
    print "Adjust your PYTHONPATH to see library directories under /usr/local/lib"

env = config.Finish()

## Two shared libraries provide most of the code for the C programs

libgps_sources = [
	"ais_json.c",
	"daemon.c",
	"gpsutils.c",
	"geoid.c",
	"gpsdclient.c",
	"gps_maskdump.c",
	"hex.c",
	"json.c",
	"libgps_core.c",
	"libgps_json.c",
	"libgps_shm.c",
        "libgps_sock.c",
	"netlib.c",
	"rtcm2_json.c",
	"shared_json.c",
	"strl.c",
]
if cxx and GetOption('libgpsmm'):
    libgps_sources.append("libgpsmm.cpp")

libgpsd_sources = [
	"bits.c",
	"bsd-base64.c",
	"crc24q.c",
	"gpsd_json.c",
	"isgps.c",
	"libgpsd_core.c",
	"net_dgpsip.c",
	"net_gnss_dispatch.c",
	"net_ntrip.c",
	"packet.c",
	"pseudonmea.c",
	"serial.c",
	"srecord.c",
	"subframe.c",
	"timebase.c",
	"drivers.c",
	"driver_aivdm.c",
	"driver_evermore.c",
	"driver_garmin.c",
	"driver_garmin_txt.c",
	"driver_geostar.c",
	"driver_italk.c",
	"driver_navcom.c",
	"driver_nmea.c",
	"driver_oncore.c",
	"driver_rtcm2.c",
	"driver_rtcm3.c",
	"driver_sirf.c",
	"driver_superstar2.c",
	"driver_tsip.c",
	"driver_ubx.c",
	"driver_zodiac.c",
]

compiled_gpslib = env.SharedLibrary(target="gps", source=libgps_sources)
compiled_gpsdlib = env.SharedLibrary(target="gpsd", source=libgpsd_sources)

# The libraries have dependencies on system libraries

gpslibs = ["gps", "m"]
gpsdlibs = ["gpsd"] + usblibs + bluezlibs + gpslibs

# Source groups

gpsd_sources = ['gpsd.c','ntpshm.c','shmexport.c','dbusexport.c']

gpsmon_sources = [
    'gpsmon.c',
    'monitor_italk.c',
    'monitor_nmea.c',
    'monitor_oncore.c',
    'monitor_sirf.c',
    'monitor_superstar2.c',
    'monitor_tnt.c',
    'monitor_ubx.c',
    ]

## Production programs
gpsd = env.Program('gpsd', gpsd_sources,
                   LINKFLAGS = "-pthread",
                   LIBS = gpsdlibs + rtlibs + dbus_xmit_libs)
gpsdecode = env.Program('gpsdecode', ['gpsdecode.c'], LIBS=gpsdlibs+rtlibs)
gpsctl = env.Program('gpsctl', ['gpsctl.c'], LIBS=gpsdlibs+rtlibs)
gpsmon = env.Program('gpsmon', gpsmon_sources, LIBS=gpsdlibs + ncurseslibs)
gpspipe = env.Program('gpspipe', ['gpspipe.c'], LIBS=gpslibs)
gpxlogger = env.Program('gpxlogger', ['gpxlogger.c'], LIBS=gpslibs+dbus_recv_libs)
lcdgps = env.Program('lcdgps', ['lcdgps.c'], LIBS=gpslibs)
cgps = env.Program('cgps', ['cgps.c'], LIBS=gpslibs + ncurseslibs)

binaries = [gpsd, gpsdecode, gpsctl, gpspipe, gpxlogger, lcdgps]
if ncurseslibs:
    binaries += [cgps, gpsmon]

# Test programs
# TODO: conditionally add test_gpsmm and test_qgpsmm
test_float = env.Program('test_float', ['test_float.c'])
test_geoid = env.Program('test_geoid', ['test_geoid.c'], LIBS=gpslibs)
test_json = env.Program('test_json', ['test_json.c'], LIBS=gpslibs)
test_mkgmtime = env.Program('test_mkgmtime', ['test_mkgmtime.c'], LIBS=gpslibs)
test_trig = env.Program('test_trig', ['test_trig.c'], LIBS=["m"])
test_packet = env.Program('test_packet', ['test_packet.c'], LIBS=gpsdlibs)
test_bits = env.Program('test_bits', ['test_bits.c', "bits.c"])
test_gpsmm = env.Program('test_gpsmm', ['test_gpsmm.cpp'], LIBS=gpslibs)
test_libgps = env.Program('test_libgps', ['test_libgps.c'], LIBS=gpslibs)
testprogs = [test_float, test_trig, test_bits, test_packet,
             test_mkgmtime, test_geoid, test_json, test_libgps]
if cxx and GetOption("libgpsmm"):
    testprogs.append(test_gpsmm)

# Python programs
python_progs = ["gpscat", "gpsfake", "gpsprof", "xgps", "xgpsspeed"]
python_modules = ["gps/__init__.py", "gps/misc.py", "gps/fake.py",
                  "gps/gps.py", "gps/client.py"]

# Build Python binding
#
PYEXTENSIONS = ["gpspacket.so", "gpslib.so"]

# Build Python modules and scripts using distutils via setup.py.
# We define build-lib and build-scripts as distutils might have been
# configured to use different directories, but we want to use the
# produced files within the regress-driver script - therefore we
# need to build them in directories we know about.
#
# TODO:  Should the dependency on libgps.la be enforced inside
# setup.py?  (See the variable 'needed_files' in setup.py.)
#
# The following to-do items are remnants from the autotools build.
# They may no longer be relevant given the differences in the scons
# build model.
#
# TO-DO: distutils uses a temporary directory; pass in an appopriate
# path so that it is under the build directory, not the source
# directory.  For now, chmod +w the source directory because it's
# better to have the VPATH build work than to have purity so that
# distcheck can work.
#
# TO-DO: the way shlibs are put in the gps directory of the source tree
# and used for regression is wrong for VPATH builds.  regress-driver
# should instead load from pylibdir, and then this symlink
# hack can be removed.  For now, try to be parallel.
#
abs_builddir = os.getcwd()
pylibdir    = "build/lib.%s-%s"  % (get_platform(), sys.version[0:3])
pyscriptdir = "build/scripts-%s" % sys.version[0:3]
python_parts = env.Command('python-parts',
                           ["gpspacket.c",
                            "gpsclient.c",
                            "setup.py",
                            compiled_gpslib,
                            compiled_gpsdlib]
                           + python_progs
                           + python_modules, [
    "(cd $SRCDIR; chmod a+w .; "
    "env version=" + gpsd_version + " abs_builddir=" + abs_builddir + " MAKE='scons -Q' "
    "$PYTHON setup.py build " +
    "--build-lib " + os.path.join(abs_builddir, pylibdir) + " " +
    "--build-scripts " + os.path.join(abs_builddir, pyscriptdir) + " " +
    ("--mangenerator '%s') && " % mangenerator) +
    ("(cd '%s' && " % abs_builddir) +
    "mkdir -p gps && cd gps && rm -f *.so && " +
    "ln -s %s/%s/gps/*.so . ) " % (abs_builddir, pylibdir)
    ])

#
# Special dependencies to make generated files
# TO-DO: Inline the Python scripts.
#

env.Command(target = "packet_names.h", source="packet_states.h", action="""
	rm -f $TARGET &&\
	sed -e '/^ *\([A-Z][A-Z0-9_]*\),/s//   \"\1\",/' <$SOURCE >$TARGET &&\
	chmod a-w $TARGET""")

env.Command(target="timebase.h", source="leapseconds.cache",
            action='$PYTHON leapsecond.py -h $SOURCE >$TARGET')

env.Command(target="gpsd.h", source="gpsd_config.h", action="""\
	rm -f $TARGET &&\
	echo \"/* This file is generated.  Do not hand-hack it! */\" >$TARGET &&\
	cat $TARGET-head >>$TARGET &&\
	cat gpsd_config.h >>$TARGET &&\
	cat $TARGET-tail >>$TARGET &&\
	chmod a-w $TARGET""")
Depends(target="gpsd.h", dependency="gpsd.h-head")
Depends(target="gpsd.h", dependency="gpsd.h-tail")

env.Command(target="gps_maskdump.c", source="maskaudit.py", action='''
	rm -f $TARGET &&\
        $PYTHON $SOURCE -c $SRCDIR >$TARGET &&\
        chmod a-w $TARGET''')
Depends(target="gps_maskdump.c", dependency="gps.h")
Depends(target="gps_maskdump.c", dependency="gpsd.h")

env.Command(target="ais_json.i", source="jsongen.py", action='''\
	rm -f $TARGET &&\
	$PYTHON $SOURCE --ais --target=parser >$TARGET &&\
	chmod a-w $TARGET''')

generated_sources = ['packet_names.h', 'timebase.h', 'gpsd.h',
                     'gps_maskdump.c', 'ais_json.c']

# Under autotools this depended on Makefile. We need it to depend
# on the state of the build-system variables.
env.Command(target="revision.h", source="gpsd_config.h", action='''
	rm -f $TARGET &&\
	$PYTHON -c \'from datetime import datetime; print "#define REVISION \\"%s\\"\\n" % (datetime.now().isoformat()[:-4])\' >$TARGET &&\
	chmod a-w revision.h''')

# leapseconds.cache is a local cache for information on leapseconds issued
# by the U.S. Naval observatory. It gets kept in the repository so we can
# build without Internet access.
env.Command(target="leapseconds.cache", source="leapsecond.py",
            action='$PYTHON $SOURCE -f $TARGET')

# Instantiate some file templates.  We'd like to use the Substfile builtin
# but it doesn't seem to work in scons 1.20
def substituter(target, source, env):
    substmap = (
        ('@VERSION@', gpsd_version),
        ('@prefix@',  GetOption('prefix')),
        ('@PYTHON@',  "$PYTHON"),
        )
    with open(str(source[0])) as sfp:
        content = sfp.read()
    for (s, t) in substmap:
        content = content.replace(s, t)
    with open(str(target[0]), "w") as tfp:
        tfp.write(content)
for fn in ("packaging/rpm/gpsd.spec.in", "libgps.pc.in", "libgpsd.pc.in",
       "jsongen.py.in", "maskaudit.py.in", "valgrind-audit.in"):
    builder = env.Command(source=fn, target=fn[:-3], action=substituter)
    post1 = env.AddPostAction(builder, 'chmod -w $TARGET')
    if fn.endswith(".py.in"):
        env.AddPostAction(post1, 'chmod +x $TARGET')

# Documentation

base_manpages = {
    "gpsd.8" : "gpsd.xml",
    "gpsd_json.5" : "gpsd_json.xml",
    "gps.1" : "gps.xml",
    "cgps.1" : "gps.xml",
    "lcdgps.1" : "gps.xml",
    "libgps.3" : "libgps.xml",
    "libgpsmm.3" : "libgpsmm.xml",
    "libQgpsmm.3" : "libgpsmm.xml",
    "libgpsd.3" : "libgpsd.xml",
    "gpsmon.1": "gpsmon.xml",
    "gpsctl.1" : "gpsctl.xml",
    "gpspipe.1" : "gpspipe.xml",
    "gpsdecode.1" : "gpsdecode.xml",
    "srec.5" : "srec.xml",
    }
python_manpages = {
    "gpsprof.1" : "gpsprof.xml",
    "gpsfake.1" : "gpsfake.xml",
    "gpscat.1" : "gpscat.xml",
    "xgpsspeed.1" : "gps.xml",
    "xgps.1" : "gps.xml",
    }

manpage_targets = []
if manbuilder:
    for (man, xml) in base_manpages.items():
        manpage_targets.append(env.Man(source=xml, target=man))

## Where it all comes together

build = env.Alias('build', binaries + python_parts + manpage_targets)
env.Default(*build)

## Installation and deinstallation

# Not here because too distro-specific: udev rules, desktop files, init scripts
# Possible bug: scons install -n doesn't show ldconfig postaction

for (name, metavar, help, default) in pathopts:
    exec name + " = os.path.join(GetOption('prefix') + GetOption('%s'))" % name

binaryinstall = []
binaryinstall.append(env.Install(sbindir, gpsd))
binaryinstall.append(env.Install(bindir,  [gpsdecode, gpsctl, gpspipe, gpxlogger, lcdgps]))
if ncurseslibs:
    binaryinstall.append(env.Install(bindir, [cgps, gpsmon]))
libversion = "%d.%d.%d" % (libgps_major, libgps_minor, libgps_age)
binaryinstall.append(env.InstallAs(source=compiled_gpslib,
              target=os.path.join(libdir, "libgps.so." + libversion)))
binaryinstall.append(env.InstallAs(source=compiled_gpsdlib,
              target=os.path.join(libdir, "libgpsd.so." + libversion)))
if have_chrpath:
    env.AddPostAction(binaryinstall, 'chrpath -d $TARGET')
if not (ARGUMENTS.get('debug', 0) or ARGUMENTS.get('profiling', 0)):
    env.AddPostAction(binaryinstall, 'strip $TARGET')

maninstall = []
for manpage in base_manpages:
    section = manpage.split(".")[1]
    dest = os.path.join(mandir, "man"+section, manpage)
    maninstall.append(env.InstallAs(source=manpage, target=dest))
install = env.Alias('install', binaryinstall + maninstall)
env.AddPostAction(install, ['ldconfig'])

def Uninstall(nodes):
    deletes = []
    for node in nodes:
        if node.__class__ == install[0].__class__:
            deletes.append(Uninstall(node.sources))
        else:
            deletes.append(Delete(str(node)))
    return deletes
uninstall = env.Command('uninstall', '', Flatten(Uninstall(Alias("install"))) or "")
env.AddPostAction(uninstall, ['ldconfig'])
env.AlwaysBuild(uninstall)
env.Precious(uninstall)

# Utility productions

def Utility(target, source, action):
    target = env.Command(target=target, source=source, action=action)
    env.AlwaysBuild(target)
    env.Precious(target)
    return target

# setup.py needs this
Utility('version' '', [],
        '@echo ' + gpsd_version + "\n")

# Report splint warnings
# Note: test_bits.c is unsplintable because of the PRI64 macros.
env['SPLINTOPTS'] = "-I/usr/include/libusb-1.0 +quiet -DSYSCONFDIR='\"./\"' "

def Splint(target,sources, description, params):
    return Utility(target,sources,[
            '@echo "Running splint on %s..."'%description,
            '-splint $SPLINTOPTS %s %s'%(" ".join(params)," ".join(sources)),
            ])

splint_table = [
    ('splint-daemon',gpsd_sources,'daemon', ['-exportlocal', '-redef']),
    ('splint-libgpsd',libgpsd_sources,'libgpsd', ['-exportlocal', '-redef']),
    ('splint-libgps',libgps_sources,'user-side libraries', ['-exportlocal', '-redef']),
    ('splint-cgps',['cgps.c'],'cgps', ['-exportlocal']),
    ('splint-gpsctl',['gpsctl.c'],'gpsctl', ['']),
    ('splint-gpsmon',gpsmon_sources,'gpsmon', ['-exportlocal']),
    ('splint-gpspipe',['gpspipe.c'],'gpspipe', ['']),
    ('splint-gpsdecode',['gpsdecode.c'],'gpsdecode', ['']),
    ('splint-gpxlogger',['gpxlogger.c'],'gpxlogger', ['']),
    ('splint-test_packet',['test_packet.c'],'test_packet test harness', ['']),
    ('splint-test_mkgmtime',['test_mkgmtime.c'],'test_mkgmtime test harness', ['']),
    ('splint-test_geoid',['test_geoid.c'],'test_geoid test harness', ['']),
    ('splint-test_json',['test_json.c'],'test_json test harness', ['']),
    ]

for (target,sources,description,params) in splint_table:
    env.Alias('splint',Splint(target,sources,description,params))


Utility("cppcheck", ["gpsd.h", "packet_names.h"],
        "cppcheck --template gcc --all --force $SRCDIR")

# Check the documentation for bogons, too
Utility("xmllint", glob.glob("*.xml"),
	"for xml in $SOURCES; do xmllint --nonet --noout --valid $$xml; done")

# Use deheader to remove headers not required.  If the statistics line
# ends with other than '0 removed' there's work to be done.
Utility("deheader", generated_sources, [
	'deheader -x cpp -x contrib -x gpspacket.c -x gpsclient.c -x monitor_proto.c -i gpsd_config.h -i gpsd.h -m "MORECFLAGS=\'-Werror -Wfatal-errors -DDEBUG -DPPS_ENABLE\' scons -Q"',
        ])

env.Alias('checkall', ['cppcheck','xmllint','splint'])

#
# Regression tests begin here
#
# Note that the *-makeregress targets re-create the *.log.chk source
# files from the *.log source files.

# Check that all Python modules compile properly 
def check_compile(target, source, env):
    for pyfile in source:
        'cp %s tmp.py'%(pyfile)
        '$PYTHON -tt -m py_compile tmp.py'
        'rm -f tmp.py tmp.pyc'
python_compilation_regress = Utility('python-comilation-regress',
        Glob('*.py') + Glob('gps/*.py') + python_progs + ['SConstruct'], check_compile)

# Regression-test the daemon
gps_regress = Utility("gps-regress", [gpsd, python_parts],
        '$SRCDIR/regress-driver test/daemon/*.log')

# Test that super-raw mode works. Compare each logfile against itself
# dumped through the daemon running in R=2 mode.  (This test is not
# included in the normal regressions.)
Utility("raw-regress", [gpsd, python_parts],
	'$SRCDIR/regress-driver test/daemon/*.log')

# Build the regression tests for the daemon.
Utility('gps-makeregress', [gpsd, python_parts],
	'$SRCDIR/regress-driver -b test/daemon/*.log')

# To build an individual test for a load named foo.log, put it in
# test/daemon and do this:
#	regress-driver -b test/daemon/foo.log

# Regression-test the RTCM decoder.
rtcm_regress = Utility('rtcm-regress', [gpsdecode], [
	'@echo "Testing RTCM decoding..."',
	'for f in $SRCDIR/test/*.rtcm2; do '
		'echo "Testing $${f}..."; '
		'$SRCDIR/gpsdecode -j <$${f} >/tmp/test-$$$$.chk; '
		'diff -ub $${f}.chk /tmp/test-$$$$.chk; '
	'done;',
	'@echo "Testing idempotency of JSON dump/decode for RTCM2"',
	'$SRCDIR/gpsdecode -e -j <test/synthetic-rtcm2.json >/tmp/test-$$$$.chk; '
		'grep -v "^#" test/synthetic-rtcm2.json | diff -ub - /tmp/test-$$$$.chk; '
		'rm /tmp/test-$$$$.chk',
        ])

# Rebuild the RTCM regression tests.
Utility('rtcm-makeregress', [gpsdecode], [
	'for f in $SRCDIR/test/*.rtcm2; do '
		'$SRCDIR/gpsdecode -j < ${f} > ${f}.chk; '
	'done'
        ])

# Regression-test the AIVDM decoder.
aivdm_regress = Utility('aivdm-regress', [gpsdecode], [
	'@echo "Testing AIVDM decoding..."',
	'for f in $SRCDIR/test/*.aivdm; do '
		'echo "Testing $${f}..."; '
		'$SRCDIR/gpsdecode -u -c <$${f} >/tmp/test-$$$$.chk; '
		'diff -ub $${f}.chk /tmp/test-$$$$.chk; '
	'done;',
	'@echo "Testing idempotency of JSON dump/decode for AIS"',
	'$SRCDIR/gpsdecode -e -j <$SRCDIR/test/synthetic-ais.json >/tmp/test-$$$$.chk; '
		'grep -v "^#" $SRCDIR/test/synthetic-ais.json | diff -ub - /tmp/test-$$$$.chk; '
		'rm /tmp/test-$$$$.chk',
        ])

# Rebuild the AIVDM regression tests.
Utility('aivdm-makeregress', [gpsdecode], [
	'for f in $SRCDIR/test/*.aivdm; do '
		'$SRCDIR/gpsdecode -u -c <$${f} > $${f}.chk; '
	'done',
        ])

# Regression-test the packet getter.
packet_regress = Utility('packet-regress', [test_packet], [
    '@echo "Testing detection of invalid packets..."',
    '$SRCDIR/test_packet | diff -u $SRCDIR/test/packet.test.chk -',
    ])

# Rebuild the packet-getter regression test
Utility('packet-makeregress', [test_packet], [
    '$SRCDIR/test_packet >$SRCDIR/test/packet.test.chk',
    ])

# Rebuild the packet-getter regression test
Utility('geoid-makeregress', [test_geoid], [
    '$SRCDIR/test_geoid 37.371192 122.014965 >$SRCDIR/test/geoid.test.chk'])

# Regression-test the geoid tester.
geoid_regress = Utility('geoid-regress', [test_geoid], [
    '@echo "Testing the geoid model..."',
    '$SRCDIR/test_geoid 37.371192 122.014965 | diff -u $SRCDIR/test/geoid.test.chk -',
    ])

# Regression-test the calendar functions
time_regress = Utility('time-regress', [test_mkgmtime], [
    '$SRCDIR/test_mkgmtime'
    ])

# Regression test the unpacking code in libgps
unpack_regress = Utility('unpack-regress', [test_libgps], [
    '@echo "Testing the client-library sentence decoder..."',
    '$SRCDIR/regress-driver -c $SRCDIR/test/clientlib/*.log',
    ])

# Build the regression test for the sentence unpacker
Utility('unpack-makeregress', [test_libgps], [
    '@echo "Rebuilding the client sentence-unpacker tests..."',
    '$SRCDIR/regress-driver -c -b $SRCDIR/test/clientlib/*.log'
    ])

# Unit-test the JSON parsing
json_regress = Utility('json-regress', [test_json], [
    '$SRCDIR/test_json'
    ])

# Unit-test the bitfield extractor - not in normal tests
bits_regress = Utility('bits-regress', [test_bits], [
    '$SRCDIR/test_bits'
    ])

# Run all normal regression tests
check = env.Alias('check', [
    python_compilation_regress,
    gps_regress,
    rtcm_regress,
    aivdm_regress,
    packet_regress,
    geoid_regress,
    time_regress,
    unpack_regress,
    json_regress])

env.Alias('testregress', check)

# The website directory
#
# None of these productions are fired by default.

env.Alias('website', Split('''
    www/gpscat.html www/gpsctl.html www/gpsdecode.html 
    www/gpsd.html www/gpsfake.html www/gpsmon.html 
    www/gpspipe.html www/gpsprof.html www/gps.html 
    www/libgpsd.html www/libgpsmm.html www/libgps.html
    www/srec.html
    www/AIVDM.html www/NMEA.html
    www/protocol-evolution.html www/protocol-transition.html
    www/client-howto.html www/writing-a-driver.html
    www/index.html www/hardware.html
    www/performance/performance.html
    www/internals.html
    '''))

# asciidoc documents
for stem in ['AIVDM', 'NMEA',
             'protocol-evolution', 'protocol-transition'
             'client-howto']:
    env.Command('www/%s.html' % stem, 'www/%s.txt' % stem,    
            ['asciidoc -a toc -o www/%s.html www/%s.txt' % (stem, stem)])

if htmlbuilder:
    # DocBook documents
    for stem in ['writing-a-driver', 'performance/performance']:
        env.HTML('www/%s.html' % stem, 'www/%s.xml' % stem)

    # The internals manual.
    # Doesn't capture dependencies on the subpages
    env.HTML('www/internals.html', '$SRCDIR/doc/explanation.xml')

# The index page
env.Command('www/index.html', 'www/index.html.in',
            ['sed -e "/@DATE@/s//`date \'+%B %d, %Y\'`/" <$SOURCE >$TARGET'])

# The hardware page
env.Command('www/hardware.html', ['gpscap.py',
                                  'www/hardware-head.html',
                                  'gpscap.ini',
                                  'www/hardware-tail.html'],
            ['(cat www/hardware-head.html; $PYTHON gpscap.py; cat www/hardware-tail.html) >www/hardware.html'])


# Experimenting with pydoc.  Not yet fired by any other productions.

env.Alias('pydoc', "www/pydoc/index.html")

# We need to run epydoc with the Python version we built the modules for.
# So we define our own epydoc instead of using /usr/bin/epydoc
EPYDOC = "python -c 'from epydoc.cli import cli; cli()'"
env.Command('www/pydoc/index.html', python_progs + glob.glob("*.py")  + glob.glob("gps/*.py"), [
	'mkdir -p www/pydoc',
	EPYDOC + " -v --html --graph all -n GPSD $SOURCES -o www/pydoc",
        ])

# Productions for setting up and performing udev tests.
#
# Requires root. Do "udev-install", then "tail -f /var/log/syslog" in
# another window, then run 'make udev-test', then plug and unplug the
# GPS ad libitum.  All is well when you get fix reports each time a GPS
# is plugged in.

Utility('udev-install', '', [
	'cp $SRCDIR/gpsd.rules /lib/udev/rules.d/25-gpsd.rules',
	'cp $SRCDIR/gpsd.hotplug $SRCDIR/gpsd.hotplug.wrapper /lib/udev/',
	'chmod a+x /lib/udev/gpsd.hotplug /lib/udev/gpsd.hotplug.wrapper',
        ])

Utility('udev-uninstall', '', [
	'rm -f /lib/udev/{gpsd.hotplug,gpsd.hotplug.wrapper}',
	'rm -f /lib/udev/rules.d/25-gpsd.rules',
        ])

Utility('udev-test', '', [
	'$SRCDIR/gpsd -N -n -F /var/run/gpsd.sock -D 5',
        ])

# Release machinery begins here
#
# We need to be in the actual project repo (i.e. not doing a -Y build)
# for these productions to work.

if os.path.exists("gpsd.c") and os.path.exists(".gitignore"):
    distfiles = commands.getoutput(r"git ls-files |egrep -v '^(www|devtools|packaging|repo)'")
    distfiles = distfiles.split()
    distfiles.remove(".gitignore")
    distfiles += generated_sources
    distfiles += base_manpages.keys() + python_manpages.keys()

    dist = env.Command('dist', distfiles, [
        '@tar -czf gpsd-${VERSION}.tar.gz $SOURCES',
        '@ls -l gpsd-${VERSION}.tar.gz',
        ])
    env.Clean(dist, "gpsd-${VERSION}.tar.gz")

    # Make RPM from the specfile in packaging
    Utility('dist-rpm', dist, 'rpm -ta $SOURCE')

    # Make sure build-from-tarball works.
    Utility('testbuild', [dist], [
        'tar -xzvf gpsd-${VERSION}.tar.gz',
        'cd gpsd-${VERSION}; ./configure; make',
        'rm -fr gpsd-${VERSION}',
        ])

    # This is how to ship a release to Berlios incoming.
    # It requires developer access verified via ssh.
    #
    upload_ftp = Utility('upload-ftp', dist, [
            'shasum gpsd-${VERSION}.tar.gz >gpsd-${VERSION}.sum',
            'lftp -c "open ftp://ftp.berlios.de/incoming; mput $SOURCE gpsd-${VERSION}.sum"',
            ])

    #
    # This is how to tag a release.
    # It requires developer access verified via ssh.
    #
    release_tag = Utility("release-tag", '', [
            'git tag -s -m "Tagged for external release $VERSION" release-$VERSION',
            'git push --tags'
            ])

    #
    # Ship a release, providing all regression tests pass.
    # The clean is necessary so that dist will remake revision.h
    # with the current revision level in it.
    #
    Utility('ship', '', [check, dist, upload_ftp, release_tag])

# The following sets edit modes for GNU EMACS
# Local Variables:
# mode:python
# End:
