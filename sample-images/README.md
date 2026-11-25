# Images de test

Échantillons libres utilisés pour tester la lecture de tous les formats supportés par l'appli.

| Format | Fichiers | Source |
|---|---|---|
| RAW (CR2, CR3, NEF, DNG, RAF, RW2) | `raw/` | [ExifTool `t/images`](https://github.com/exiftool/exiftool/tree/master/t/images) (Perl Artistic License) |
| TIFF / TIF | `tiff/` | ExifTool + [Pillow `Tests/images`](https://github.com/python-pillow/Pillow/tree/main/Tests/images) (HPND license) |
| AVIF | `avif/` | [link-u/avif-sample-images](https://github.com/link-u/avif-sample-images) (MIT) |
| WebP | `webp/` | ExifTool + Pillow |

Le WebP contient des métadonnées EXIF/XMP ; le TIF est un GeoTIFF — utiles pour tester le tri par date/GPS.
