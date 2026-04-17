# Maintainer: Marcel Zimmer <https://www.marcelzimmer.de>
pkgname=mz-protokoll
pkgver=1.0.0
pkgrel=1
pkgdesc="Desktop-App zum Erstellen und Exportieren von Meeting-Protokollen (Markdown & PDF)"
arch=('x86_64')
url="https://github.com/marcelzimmer/mz-protokoll"
license=('MIT')
depends=('gcc-libs')
source=("${pkgname}-${pkgver}.zip::https://github.com/marcelzimmer/mz-protokoll/releases/download/v${pkgver}/mz-protokoll-linux-x86_64.zip")
sha256sums=('f045a9cf063bb08828eb779e6db46ed184ad4b49eef7a9057a933f0adad68c1f')

package() {
    cd "$srcdir"

    install -Dm755 mz-protokoll "$pkgdir/usr/bin/mz-protokoll"

    install -Dm644 icon.png "$pkgdir/usr/share/icons/hicolor/256x256/apps/mz-protokoll.png"

    install -Dm644 /dev/stdin "$pkgdir/usr/share/applications/mz-protokoll.desktop" << EOF
[Desktop Entry]
Name=MZ-Protokoll
Comment=Meeting-Protokolle erstellen und exportieren
Exec=mz-protokoll
Icon=mz-protokoll
Type=Application
Categories=Office;
Terminal=false
EOF
}
