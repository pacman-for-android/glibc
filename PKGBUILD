# Maintainer: Giancarlo Razzolini <grazzolini@archlinux.org>
# Maintainer: Frederik Schwan <freswa at archlinux dot org>
# Contributor: Bartłomiej Piotrowski <bpiotrowski@archlinux.org>
# Contributor: Allan McRae <allan@archlinux.org>

# toolchain build order: linux-api-headers->glibc->binutils->gcc->glibc->binutils->gcc
# NOTE: valgrind requires rebuilt with each major glibc version

pkgbase=glibc
pkgname=(glibc)
pkgver=2.38
_commit=6b99458d197ab779ebb6ff632c168e2cbfa4f543
pkgrel=3.10
arch=(x86_64 aarch64)
url='https://www.gnu.org/software/libc'
license=(GPL LGPL)
makedepends=(git gd python)
options=(staticlibs !lto)
source=(git+https://sourceware.org/git/glibc.git#commit=${_commit}
        locale.gen.txt
        locale-gen
        lib32-glibc.conf
        sdt.h sdt-config.h
        reenable_DT_HASH.patch
        fix-malloc-p1.patch
        fix-malloc-p2.patch
        glibc-change-pathes.patch
)
validpgpkeys=(7273542B39962DF7B299931416792B4EA25340F8 # Carlos O'Donell
              BC7C7372637EC10C57D7AA6579C43DFBF1CF2187) # Siddhesh Poyarekar
b2sums=('SKIP'
        '8b4cb1fec0a5c5447816bcb7c622a34806e38ea719869ec4d830bf8ddca6ee880dac41f612d47ea84072fac538221eaf93eb7690a2701e3724376f1dcab211f4'
        '10db43cc8dee6efd8349448609910f513d8bb551fecea82823a2e2bc7f2ed45f9e9280e253a0df76fa2030efb9ffa97838d2a32cad6c4d2310a9af53631315b9'
        '7c265e6d36a5c0dff127093580827d15519b6c7205c2e1300e82f0fb5b9dd00b6accb40c56581f18179c4fbbc95bd2bf1b900ace867a83accde0969f7b609f8a'
        'a6a5e2f2a627cc0d13d11a82458cfd0aa75ec1c5a3c7647e5d5a3bb1d4c0770887a3909bfda1236803d5bc9801bfd6251e13483e9adf797e4725332cd0d91a0e'
        '214e995e84b342fe7b2a7704ce011b7c7fc74c2971f98eeb3b4e677b99c860addc0a7d91b8dc0f0b8be7537782ee331999e02ba48f4ccc1c331b60f27d715678'
        '35e03ed912e1b0cd23783ab83ce919412885c141344905b8b67bbad4a86c48cf3e893806060e48d5737514ff80cea0b58b0e1f15707c32224579c416dcd810c0'
        '28c983bcebc0eeeb37a60756ccee50d587a99d5e2100430d5c0ee51a19d9b2176a4013574a7d72b5857302fbb60d371bbf0b3cdb4fc700a1dbe3aae4a42b04b9'
        'c3e94f5b0999878ff472e32f49dc13c20eb9db68c633017cb7824617eb824cf6cff7ea53b92962926e0ee84fd39736616298dcb926356625dd124f3754e79932'
        'f21927030e8ca222fb74aea23e9750bfe3bdc4ecc0beb6129e5466bf915481dd15872946ced40457508adb2ea4e06bbc67aacef42beafd5613542b965b18f602')


prepare() {
  mkdir -p glibc-build lib32-glibc-build

  [[ -d glibc-$pkgver ]] && ln -s glibc-$pkgver glibc
  cd glibc

  # Re-enable `--hash-style=both` for building shared objects due to issues with EPIC's EAC
  # which relies on DT_HASH to be present in these libs.
  # reconsider 2023-01
  patch -Np1 -i "${srcdir}"/reenable_DT_HASH.patch

  patch -Np1 -i "${srcdir}"/fix-malloc-p1.patch
  patch -Np1 -i "${srcdir}"/fix-malloc-p2.patch
  patch -Np1 -i "${srcdir}"/glibc-change-pathes.patch
  sed -i '1s|.*|#!/data/usr/bin/sh|' configure scripts/{install-sh,pylint,move-if-change,rellns-sh,mkinstalldirs,update-copyrights,*.sh,cpp}
}

build() {
  local _configure_flags=(
      --prefix=/data/usr
      --sbindir=/data/usr/bin
      --localstatedir=/data/var
      --sysconfdir=/data/etc
      --with-headers=/data/usr/include
      --enable-bind-now
      # --enable-cet
      --enable-fortify-source
      --enable-kernel=4.4
      --enable-stack-protector=strong
      --enable-systemtap
      --disable-profile
      --disable-werror
  )

  cd "${srcdir}"/glibc-build

  echo "slibdir=/data/usr/lib" >> configparms
  echo "rtlddir=/data/usr/lib" >> configparms
  echo "sbindir=/data/usr/bin" >> configparms
  echo "rootsbindir=/data/usr/bin" >> configparms

  # Credits @allanmcrae
  # https://github.com/allanmcrae/toolchain/blob/f18604d70c5933c31b51a320978711e4e6791cf1/glibc/PKGBUILD
  # remove fortify for building libraries
  # CFLAGS=${CFLAGS/-Wp,-D_FORTIFY_SOURCE=2/}

  "${srcdir}"/glibc/configure \
      --libdir=/data/usr/lib \
      --libexecdir=/data/usr/lib \
      "${_configure_flags[@]}"

  make -O -j8

  # build info pages manually for reproducibility
  make info


  # pregenerate C.UTF-8 locale until it is built into glibc
  # (https://sourceware.org/glibc/wiki/Proposals/C.UTF-8, FS#74864)-
  elf/ld.so --library-path "$PWD" locale/localedef -c -f ../glibc/localedata/charmaps/UTF-8 -i ../glibc/localedata/locales/C ../C.UTF-8/
}

# Credits for skip_test() and check() @allanmcrae
# https://github.com/allanmcrae/toolchain/blob/f18604d70c5933c31b51a320978711e4e6791cf1/glibc/PKGBUILD
skip_test() {
  test=${1}
  file=${2}
  sed -i "/\b${test} /d" "${srcdir}"/glibc/${file}
}

check() {
  return 0
  cd glibc-build

  # adjust/remove buildflags that cause false-positive testsuite failures
  sed -i '/FORTIFY/d' configparms                                     # failure to build testsuite
  sed -i 's/-Werror=format-security/-Wformat-security/' config.make   # failure to build testsuite
  sed -i '/CFLAGS/s/-fno-plt//' config.make                           # 16 failures
  sed -i '/CFLAGS/s/-fexceptions//' config.make                       # 1 failure
  LDFLAGS=${LDFLAGS/,-z,now/}                                         # 10 failures

  # The following tests fail due to restrictions in the Arch build system
  # The correct fix is to add the following to the systemd-nspawn call:
  # --system-call-filter="@clock @memlock @pkey"
  skip_test test-errno-linux        sysdeps/unix/sysv/linux/Makefile
  skip_test tst-mlock2              sysdeps/unix/sysv/linux/Makefile
  skip_test tst-ntp_gettime         sysdeps/unix/sysv/linux/Makefile
  skip_test tst-ntp_gettimex        sysdeps/unix/sysv/linux/Makefile
  skip_test tst-pkey                sysdeps/unix/sysv/linux/Makefile
  skip_test tst-process_mrelease    sysdeps/unix/sysv/linux/Makefile
  skip_test tst-adjtime             time/Makefile

  TIMEOUTFACTOR=20 make -O check
}

package() {
  pkgdesc='GNU C Library'
  depends=('linux-api-headers>=4.10' tzdata filesystem)
  optdepends=('gd: for memusagestat'
              'perl: for mtrace')
  install=glibc.install
  backup=(data/etc/gai.conf
          data/etc/locale.gen
          data/etc/nscd.conf)

  make -C glibc-build install_root="${pkgdir}" SHELL=/data/usr/bin/sh install
  rm -f "${pkgdir}"/data/etc/ld.so.cache

  # Shipped in tzdata
  rm -f "${pkgdir}"/data/usr/bin/{tzselect,zdump,zic}

  cd glibc

  install -dm755 "${pkgdir}"/data/usr/lib/{locale,systemd/system,tmpfiles.d}
  install -m644 nscd/nscd.conf "${pkgdir}"/data/etc/nscd.conf
  install -m644 nscd/nscd.service "${pkgdir}"/data/usr/lib/systemd/system
  install -m644 nscd/nscd.tmpfiles "${pkgdir}"/data/usr/lib/tmpfiles.d/nscd.conf
  install -dm755 "${pkgdir}"/data/var/db/nscd

  install -m644 posix/gai.conf "${pkgdir}"/data/etc/gai.conf

  install -m755 "${srcdir}"/locale-gen "${pkgdir}"/data/usr/bin

  # Create /etc/locale.gen
  install -m644 "${srcdir}"/locale.gen.txt "${pkgdir}"/data/etc/locale.gen
  sed -e '1,3d' -e 's|/| |g' -e 's|\\| |g' -e 's|^|#|g' \
    "${srcdir}"/glibc/localedata/SUPPORTED >> "${pkgdir}"/data/etc/locale.gen

  # Add SUPPORTED file to pkg
  sed -e '1,3d' -e 's|/| |g' -e 's| \\||g' \
    "${srcdir}"/glibc/localedata/SUPPORTED > "${pkgdir}"/data/usr/share/i18n/SUPPORTED

  # install C.UTF-8 so that it is always available
  install -dm755 "${pkgdir}"/data/usr/lib/locale
  cp -r "${srcdir}"/C.UTF-8 -t "${pkgdir}"/data/usr/lib/locale
  sed -i '/#C\.UTF-8 /d' "${pkgdir}"/data/etc/locale.gen

  # Provide tracing probes to libstdc++ for exceptions, possibly for other
  # libraries too. Useful for gdb's catch command.
  install -Dm644 "${srcdir}"/sdt.h "${pkgdir}"/data/usr/include/sys/sdt.h
  install -Dm644 "${srcdir}"/sdt-config.h "${pkgdir}"/data/usr/include/sys/sdt-config.h
}

