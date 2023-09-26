#!/bin/bash
#
# htc - script for comfortable use of HiTech C

VERSION=0.72
LAST_UPDATE="09 Jun 2002"

#############################################################################
#
# Some points of consideration:
#
# - By setting the appropriate values for HTC_LIBRARYPATHS and HTC_CRT,
#   you can make htc compile applications for OS-es other than MSXDOS or
#   CP/M -- for UZIX, for example :-)
#
# - you need to have cpmemu installed
#
# - you need to have our other script, named 'htlibr', installed for linking
#
# - the path to HiTech C's .com files has to be fully lowercase
#   (restriction of cpmemu :-/) 
#
# - HiTech C is expected to be installed as follows:
#
#    ...$HTC_PATH/         - HiTech C base dir
#                 bin      - CPM binaries (p1, cgen, zas, opt and link)
#                 inc      - 'system' include files
#                 lib      - 'system' libraries (libc.lib, libf.lib ...)
#                            and the C RunTime startup code (CRT.O)
#
#############################################################################

#############################################################################
#
# Usage
#

Usage ()
{
  cat <<-END
	htc - z80 C cross-compiler using Hi-Tech C and cpmemu
	version $VERSION - $LAST_UPDATE - Aurora (Eric Boon & Manuel Bilderbeek)

	Usage: htc [options] file ...
	Options:
	  -o<file>          Output to <file> (default: aout.com)
	  -O                Turn optimization on
	  -D<name[=value]>  #define name (with value)
	  -U<name>          #undef name
	  -I<path>          Include <path> in #include path
	  -E                Preprocess only
	  -S                Compile only; do not assemble or link
	  -c                Compile and assemble, but do not link
	  -v                Be verbose
	  -l<library>       Link lib<library>.lib
	  -m                Request the linker to produce a link map
	  -h                Print this help
  
	Filetype is determined by extension:
	  .c                C source
	  .i                Pre-processed C source
	  .as .asm .gen     Assembler source
	  .o  .obj          Object files
	  .lib              Libraries

	ENVIRONMENT VARIABLES:
	  HTC_PATH          Path to HiTech C install
	                    (/usr/local/lib/hitech)
	  HTC_CPMEMU        Path to 'cpm', the CP/M emulator
	                    (/usr/local/bin/cpm)
	  HTC_LIBRARYPATHS  Path to search for libraries
	                    ("./lib ../lib $HTC_PATH/lib")
	  HTC_CRT           Path to C run-time object code
	                    ($HTC_PATH/lib/crt.o)

	TIPS:
	* If you plan to use files that originate from CP/M or MSX-DOS,
	  remember to 'cpm2unix' them to get rid of ^Z and other rubbish
	  the CP/M leaves at the end of the file! Htc can't and therefore
	  doesn't check that, but will definitely choke!

	* The -l option is recognized anywhere on the cmd line, other options
	  only at the beginning.  

	DISCLAIMER:
	  It works for us, there's no guarantee that it will for you! :-)
	  Nevertheless, please send bug reports, questions, remarks,
	  thank-you's or beer to: Aurora1@Chello.nl

	COPYRIGHT:
	  Find yourself a copy of the GNU Public License.
	END

  exit 1
}


#############################################################################
#
# Try to find the CPM emulator
#

Check_cpmemu ()
{
  CPMEMU_LIST="$HTC_CPMEMU `which cpm` /usr/local/bin/cpm "

  for MY_CPM in $CPMEMU_LIST ; do
    if [ ! -x $MY_CPM ] ; then break ; fi
  done

  if [ -z $MY_CPM ] ; then
    echo "* $ME: Unable to find cpm emulator" \
         "Please set \$HTC_CPMEMU appropriately"
    exit 1
  fi
}


#############################################################################
#
# Try to find Hi Tech C
#

Check_hitech ()
{
  MY_HTC_PATH=$HTC_PATH
  if [ -z $MY_HTC_PATH ] ; then
    MY_HTC_PATH="/usr/local/lib/hitech"
  fi

  if [ ! -d $MY_HTC_PATH ] ; then
    echo "* $ME: Unable to find HiTech C" \
         "Please set \$HTC_PATH appropriately"
    exit 1
  fi

  if [ $VERBOSE == 1 ] ; then
    echo "HITECH PATH    : $MY_HTC_PATH"
  fi

  P1=$MY_HTC_PATH/bin/p1.com
  CGEN=$MY_HTC_PATH/bin/cgen.com
  OPTIM=$MY_HTC_PATH/bin/optim.com
  ZAS=$MY_HTC_PATH/bin/zas.com
  LINK=$MY_HTC_PATH/bin/link.com

  if [[ ! ( -e $P1 && -e $CGEN && -e $OPTIM && -e $ZAS && -e $LINK ) ]] ; then
    echo "* $ME: Unable to find one or more HiTech C components! Oops..."
    exit 1
  fi
  
  MY_LIB_PATHS=$HTC_LIBRARYPATHS
  if [ -z $MY_LIB_PATHS ] ; then
    MY_LIB_PATHS=" ./lib ../lib $MY_HTC_PATH/lib "
  fi

  if [ $VERBOSE == 1 ] ; then
    echo "LIBRARY PATHS  : $MY_LIB_PATHS"
  fi
  
  MY_CRT=$HTC_CRT
  if [ -z $MY_CRT ] ; then
    MY_CRT=$MY_HTC_PATH/lib/crt.o
  fi

  if [ $DO_LINK ] ; then
    if [ ! -f $MY_CRT ] ; then
      echo "* $ME: Unable to find run-time startup code. " \
           "Please set \$HTC_CRT"
      exit 1
    fi
    
    if [ $VERBOSE == 1 ] ; then
      echo "RUNTIME OBJECT : $MY_CRT"
    fi
  fi
}

#############################################################################
#
# Find GCC
#

Check_gcc ()
{
  if ! which gcc >/dev/null ; then
    echo "* $ME: Unable to find GCC! Please set your \$PATH appropriately"
    exit 1
  fi
}

#############################################################################
#
# Find htlibr
#

Check_htlibr ()
{
  if ! which htlibr >/dev/null ; then
    echo "* $ME: Unable to find htlibr! Please set your \$PATH appropriately"
    exit 1
  fi
}

#############################################################################
#
# Pre-processing
#

do_cpp ()
{
  if [ $VERBOSE == 1 ] ; then
    echo "- Preprocessing $FILENAME"
  fi

  rm -f $FILENAME.t TMP_$FILENAME.c $FILENAME.i

  # some conversion - gcc doesn't like #asm/#endasm, so we temporarily
  # replace that with __asm and __endasm

  awk ' { gsub(/#asm/,"__asm") ;        \
          gsub(/#endasm/, "__endasm") ; \
	  print } ' $FILENAME.c > $FILENAME.t1

  if ( gcc -x c -E -nostdinc -I$MY_HTC_PATH/inc \
           $INCLUDE $DEFINE $UNDEF $FILENAME.t1 > $FILENAME.t2 ) ; then

    awk '/^#/        { print "# " $2 " " $3 ; }     \
         /^([^#]|$)/ { gsub(/__asm/, "#asm") ;      \
	               gsub(/__endasm/, "#endasm"); \
		       print ; }' $FILENAME.t2 > $FILENAME.i ;

    if [ $DO_COMPILE == 1 ] ; then
      do_cgen
      rm $FILENAME.i
    fi
  fi

  rm -f $FILENAME.t1 $FILENAME.t2
}

#############################################################################
#
# Compiling
#

do_cgen ()
{
  if [ $DO_COMPILE == 1 ] ; then
    if [ $VERBOSE == 1 ] ; then
      echo "- Compiling $FILENAME"
    fi

    $MY_CPM -f -x $P1 $FILENAME.i $FILENAME.t1
    $MY_CPM -f -x $CGEN $FILENAME.t1 $FILENAME.as
    rm $FILENAME.t1
    do_opt
    
    if [ $DO_ASSEMBLE == 1 ] ; then
      OLDFILEEXT=$FILEEXT
      FILEEXT=".as"
      do_asm
      rm -f $FILENAME.as
      FILEEXT=$OLDFILEEXT
    fi
  fi
}

#############################################################################
#
# Optimizing
#

do_opt ()
{
  if [ $DO_OPTIM == 1 ] ; then
    if [ $VERBOSE == 1 ] ; then
      echo "- Optimizing $FILENAME"
    fi

    $MY_CPM -f -x $OPTIM $FILENAME.as $FILENAME.t
    rm -f $FILENAME.as
    mv $FILENAME.t $FILENAME.as
  fi
}

#############################################################################
#
# Assembling
#

do_asm ()
{
  if [ $DO_ASSEMBLE == 1 ] ; then
    if [ $VERBOSE == 1 ] ; then
      echo "- Assembling $FILENAME"
    fi

    if [ $DO_OPTIM == 1 ] ; then
      ZASOPT="-J -N"
    else
      ZASOPT="-N"
    fi
    
    $MY_CPM -f -x $ZAS $ZASOPT -o$FILENAME.o ${FILENAME}$FILEEXT
  fi
}

#############################################################################
#
# Linking
#

do_link ()
{
  if [ $DO_LINK == 1 ] ; then
    if [ $DO_MAP == 1 ] ; then
      MAPFILE=${TARGET/%*([^.])/map}
      MAP=-M$MAPFILE
    else
      MAP=
      MAPFILE="<none>"
    fi

    if [ $VERBOSE == 1 ] ; then
      echo "- Linking to $TARGET, mapfile=$MAPFILE"
    fi

#
# build temporary library to avoid problems with command lines longer than 128 
# characters
# 

    if [[ -e tmp.lib || -e tmpcrt.o ]] ; then
      echo "Found an existing tmp.lib or tmpcrt.o, which is used internally."
      echo "Fudeba!"
      exit 1
    fi

    htlibr -b r tmp.lib $OBJECTS

    cp $MY_CRT tmpcrt.o
    $MY_CPM -f -x $LINK -Z $MAP -Ptext=0,data,bss -C100H -O$TARGET \
                        tmpcrt.o tmp.lib $REALLIBS
    
    rm tmpcrt.o tmp.lib

    if [ ! -s $TARGET ] ; then
      echo "* $ME: linking failed. No ouput. Aborting..."
      rm -f $TARGET
      return 1
    fi
  fi

  return 0
}


#############################################################################
#
# M A I N
#
#############################################################################

#
# Enable extended pattern matching
#

shopt -s extglob

#
# Preset some vars...
#

TARGET=""
DEFINE="-DMSX"
UNDEF=""
INCLUDE="-I."
VERBOSE=0
LIBS=""

DO_OPTIM=0
DO_COMPILE=1
DO_ASSEMBLE=1
DO_LINK=1
DO_MAP=0

#
# Whoami ?
#
ME=${0/*\//}

#
# Collect options...
#

while getopts D:EI:OSU:cl:mo:vh c; do
  case $c in
    D)
      DEFINE="$DEFINE -D$OPTARG"
      ;;
    U)
      UNDEF="$UNDEF -U$OPTARG"
      ;;
    E)
      DO_COMPILE=0
      DO_ASSEMBLE=0
      DO_LINK=0
      ;;
    I)
      INCLUDE="$INCLUDE -I$OPTARG"
      ;;
    O)
      DO_OPTIM=1
      ;;
    S)
      DO_ASSEMBLE=0
      DO_LINK=0
      ;;
    c)
      DO_LINK=0
      ;;
    l)
      LIBS="$LIBS $OPTARG"
      ;;
    m)
      DO_MAP=1
      ;;
    o)
      if [ ! -z $TARGET ] ; then
        echo "* $ME: Multiple targets"
	exit 1
      else
        TARGET=$OPTARG
      fi
      ;;
    v)
      VERBOSE=1
      ;;
        
    *)
      Usage
      ;;
  esac
done

if [ $VERBOSE == 1 ] ; then
  echo "$ME v$VERSION - $LAST_UPDATE - (C) 2002 Aurora"
fi

#
# Show'em what we're gonna do
#

if [ $VERBOSE == 1 ] ; then
  if [ $DO_LINK == 1 ] ; then
    echo -n "TARGET         : "
    if [ -z $TARGET ] ; then
      echo "(not set)"
    else
      echo $TARGET
    fi
  fi
  echo    "DEFINE         : " $DEFINE
  echo    "INCLUDE        : " $INCLUDE
  echo -n "COMPILE        : cpp"
  if [ $DO_COMPILE == 1 ] ; then
    echo -n " p1 gcen"
    if [ $DO_OPTIM == 1 ] ; then
      echo -n " opt"
    fi
    if [ $DO_ASSEMBLE == 1 ] ; then
      echo -n " zas"
      if [ $DO_LINK == 1 ] ; then
        echo -n " link"
      fi
    fi
  fi
  echo ""
fi

let CNT=$OPTIND-1 
shift $CNT
unset CNT
FILES=( $* )

if [ -z $FILES ] ; then
  echo "* $ME: No input files. (Type $ME -h to get some help.)"
  exit 1
fi

if [ $VERBOSE == 1 ] ; then
  echo "FILES          : ${FILES[*]} (${#FILES[*]} files to process...)"
fi

#
# Check if everything is there
#

Check_cpmemu
Check_hitech
Check_gcc
Check_htlibr

#
# Ready to rock and roll...
#

#
# Now process each file
#
for FILE in ${FILES[*]} ; do

#
# Check whether we found another library ...
#
  if [[ $FILE == -l* ]] ; then

    LIBNAME=${FILE/-l/}
    if [ -z $LIBNAME ] ; then
      echo "* $ME: Missing library name for -l"
      exit 1
    fi
    LIBS="$LIBS lib$LIBNAME.lib"

    if [ $VERBOSE == 1 ] ; then
      echo " Found -l for lib$LIBNAME.lib"
    fi
    
    # go on with next $FILE
    continue
  fi

#
# ... or a dangling option
#

  if [[ $FILE == "-*" ]] ; then
    echo "* $ME: Unexpected option $FILE - skipping"
    continue
  fi

#
# Does the file we want to process exist?
#

  if [[ ! ( -e $FILE ) ]] ; then
    echo "* $ME: Unable to find $FILE!"
    exit 1
  fi

#
# Pattern magic :-) - get file basename, extension and path
#

  FILENAME=${FILE/%.+([^.])/}
  FILEEXT=${FILE/${FILENAME}/}
  FILEPATH=${FILE/%*([^\/])/}
  FILENAME=${FILENAME/${FILEPATH}/}

#
# No path names in source files. Tolerate ./, though
#

  if [[ ! ( $FILEPATH == "./" || -z $FILEPATH ) ]] ; then
    echo "* $ME: $FILE - pathnames in source/object files are not allowed! Sorry."
    exit 1
  fi

  if [ $VERBOSE == 1 ] ; then
    echo "Processing ${FILENAME}$FILEEXT"
  fi
  
  case $FILEEXT in
    .c)
      do_cpp
      OBJECTS="$OBJECTS ${FILENAME}.o"
      ;;
    .i)
      do_cgen 
      OBJECTS="$OBJECTS ${FILENAME}.o"
      ;;
    .as|.asm|.gen)
      do_asm
      OBJECTS="$OBJECTS ${FILENAME}.o"
      ;;
    .o|.obj)
      if [ $VERBOSE == 1 ] ; then
        echo "- Nothing to do for $FILENAME"
      fi	
      OBJECTS="$OBJECTS ${FILENAME}$FILEEXT"
      ;;
    .lib)
      if [ $VERBOSE == 1 ] ; then
        echo "- Nothing to do for $FILENAME"
      fi
      LIBS="$LIBS ${FILENAME}$FILEEXT"
      ;;
    *)
      echo "* $ME: Couldn't determine how to process $FILE!"
      exit 1
    ;;
  esac
done

#
# Done compiling objects - now link the stuff
#

LINK_OK=0

if [ $DO_LINK == 1 ] ; then

  LIBS="$LIBS libc.lib"

  if [ $VERBOSE == 1 ] ; then  
    echo "LIBS           : " $LIBS
  fi

#
# Find the libraries
#
#   RMLIBS are libraries which are copied from elsewhere to here.
#     This has to be done since the cpm linker doesn't recognize
#     pathnames :-(  These have to be removed after linking
#

  RMLIBS=" "
  REALLIBS=" "

  for LIB in $LIBS ; do

# check current dir (.)
    if [ -e "${LIB}" ] ; then
      REALLIBS="$REALLIBS ${LIB}"
      continue
    fi

# check the other paths

    LIBFOUND=0

    for LIBPATH in $MY_LIB_PATHS ; do
      THELIB="$LIBPATH/${LIB}"
      if [ -e $THELIB ] ; then
        cp $THELIB .
        RMLIBS="$RMLIBS ${LIB}"
        REALLIBS="$REALLIBS ${LIB}"
        LIBFOUND=1
        break
      fi
    done
    if [ $LIBFOUND == 0 ] ; then
      echo "* $ME: Unable to find ${LIB}!"
      exit 1
    fi
  done

#
# Check target
#

  if [ -z $TARGET ] ; then
    TARGET=${FILES[0]/%*([^.])/com}
    echo "* $ME: Outputting to implicit target $TARGET"
  fi

#
# Link!
#

  if do_link ; then
    LINK_OK=0
  else
    LINK_OK=1
  fi

#
#  Remove trash
#
  rm -f $RMLIBS
fi

# Done! Thank you for flying htc

if [[ $LINK_OK == 0 && $VERBOSE == 1 ]] ; then
  echo "* $ME: Done!"
fi

exit $LINK_OK
